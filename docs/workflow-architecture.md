# Workflow architecture

This document describes the job-split design implemented in
[`continuous-integration.yml`](../.github/workflows/continuous-integration.yml):
an attribution check, Mago, a PHPUnit matrix with an optional seeded RDBMS for
integration testing, plus optional Codecov and Infection mutation testing.

The design is inherited from
[php-db/phpdb-qa-tools](https://github.com/php-db/phpdb-qa-tools) via
[contenir/contenir-qa-tools](https://github.com/contenir/contenir-qa-tools),
which adds the `attributions` job, the `apt-packages` input, Codecov failing
the build, and runners pinned to `ubuntu-24.04`.

## Job graph

Only genuine data/gating dependencies are expressed via `needs:`; everything
else runs in parallel for speed.

```mermaid
graph LR
    attributions[attributions job]
    mago[mago job]
    test[test job] --> codecov[codecov job]
    test --> infection[mutation-test job]
```

- `attributions`, `mago` and `test` have **no `needs:`** between them —
  they're independent gates, none consumes another's output.
- `codecov` and `mutation-test` both **`needs: [test]`** — real dependencies
  (artifact consumption / gating), not just ordering preference.

Every job runs on `ubuntu-24.04` rather than `ubuntu-latest`, so a runner
image upgrade can't change system packages underneath a passing build.

## `attributions` job

Peptolab policy: no AI attribution in pull request titles, descriptions or
commit messages. On `pull_request` the job checks the title, the description
and every commit between the base branch and `HEAD`; on `push` it checks the
pushed range (`before..after`), falling back to the last commit when a branch
is first created. Any match fails the build with an error annotation.

## `mago` job

Matrix: `php-versions` only. Runs `mago format --check`, `mago lint`,
`mago analyze`, `mago guard`. No DB, no dependency-strategy matrix — the
committed `composer.lock` is enough for type resolution.

## `test` job

Matrix: `php x [lowest, locked, latest]`.

### System packages

When `apt-packages` is set, the listed Ubuntu packages are installed with
`apt-get` before the tests run — for tools the tests shell out to, such as
the qpdf and Poppler CLIs `php2pdf`'s integration tests drive. The list is passed through an env
var and word-split, never interpolated into the script. Empty (the default)
skips the step.

### DB service (manual step, not native `services:`)

Native job-level `services:` blocks *can* be conditionally disabled (an
empty `image:` expression means the service won't start — see
[GitHub's docs](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idservicesservice_idimage)),
so conditionality alone isn't why this uses a manual step. The real
constraint: `services.<id>.env` is a **static YAML map** — its *keys* must
be fixed at authoring time, and expressions can only substitute values, not
key names. Different engines need different env var names entirely
(MySQL's `MYSQL_ROOT_HOST`/`MYSQL_DATABASE`, Postgres's
`POSTGRES_PASSWORD`/`POSTGRES_DB`, ...), so a single generic `db-env-json`
input (arbitrary keys, unknown to the workflow until runtime) can't be
expressed through native `services:` at all — only a step that parses JSON
at runtime (the `jq` unpacking below) can.

**This is deliberately engine-agnostic** — the workflow never hardcodes
MySQL (or any other engine). A repository with no database never sets
`db-image` and pays zero cost. A repository that needs one (MySQL, Postgres,
MariaDB, Oracle, MSSQL, ...) supplies its own image/env/port/health-command;
SQLite needs no service at all (embedded, no server) and also just omits
`db-image`.

```yaml
- name: Start DB service
  if: inputs.db-image != ''
  run: |
    docker run -d --name db \
      -p ${{ inputs.db-port }}:${{ inputs.db-port }} \
      $(echo '${{ inputs.db-env-json }}' | jq -r 'to_entries[] | "-e \(.key)=\(.value)"' | tr '\n' ' ') \
      ${{ inputs.db-image }}

- name: Wait for DB to be healthy
  if: inputs.db-image != ''
  run: |
    for i in $(seq 1 ${{ inputs.db-health-retries }}); do
      docker exec db ${{ inputs.db-health-cmd }} && exit 0
      sleep ${{ inputs.db-health-interval-seconds }}
    done
    echo "DB did not become healthy in time" && exit 1
```

`jq` unpacks the `db-env-json` map into `-e KEY=VALUE` flags since `docker
run` doesn't accept JSON directly. The health check polls via `docker exec`
in a retry loop rather than Docker's native `HEALTHCHECK`, so it works the
same regardless of image. No explicit teardown is needed — GitHub-hosted
runners are ephemeral.

`db-health-retries` / `db-health-interval-seconds` default to `30` / `2`
(60s total — plenty for MySQL/Postgres/MariaDB) but are overridable per
caller, since Oracle/MSSQL images can take several minutes to become ready.

### Coverage

Unit tests always run; integration tests run `if: inputs.run-integration`.
On exactly one canonical leg (`matrix.php == inputs.coverage-php-version &&
matrix.dependencies == 'locked'`), coverage is collected (pcov → `clover.xml`)
and uploaded via `actions/upload-artifact` — the only leg downstream jobs
need.

## `codecov` job

`needs: [test]`, job-level `if: inputs.enable-codecov`. Downloads the
`clover.xml` artifact and runs `codecov/codecov-action`. No PHP setup, no DB.
`fail_ci_if_error: true` — a failed upload (missing token, Codecov outage)
fails the build rather than passing silently. (php-db runs report-only.)

`CODECOV_TOKEN` is an org-wide upload token (set it as a peptolab
organisation secret before enabling Codecov), so it can't infer the target
repo on its own — pass `slug: ${{ github.repository }}`. Inside a
*reusable* workflow, `github.repository` already resolves to the **calling**
repo, so this works with no extra input.

## `mutation-test` job

`needs: [test]` (gating — skip if base tests already failed). Job-level
`if: inputs.enable-infection`. Needs its own full environment (checkout,
setup-php **with `tools: mago`**, the `apt-packages` step, `composer install
--locked`, the same conditional DB-startup step as `test` if
`run-integration`) since Infection re-executes the suite per mutant — it
can't just consume `test`'s artifact the way `codecov` does.

**No `needs: [mago]`.** `infection.json5`'s `staticAnalysisTool: "mago"`
makes Infection invoke `mago analyze` internally against mutants that escape
the test suite — that's a tooling requirement inside this job (the `mago`
binary + the repo's own `mago.toml`), not a cross-job dependency on the
`mago` job.

`INFECTION_DASHBOARD_API_KEY` is a per-repo secret (generated by registering
the repo at dashboard.stryker-mutator.io). Either
`INFECTION_DASHBOARD_API_KEY` or `STRYKER_DASHBOARD_API_KEY` works as the env
var name. **Must be passed as an `env:` block on the `run:` step, not as a
`with:` input** — Infection reads it from the environment, not a CLI flag.

`min-msi` / `min-covered-msi` (both default `"10"`) control the MSI
threshold that fails the job. The default is deliberately low for
repositories just adopting mutation testing; raise both once a repository
has a real baseline to ratchet up from. Passed as `composer mutation-test -- --min-msi=...
--min-covered-msi=... --logger-github` (the composer script itself stays
plain `infection`, flags are appended at invocation time).

## Secrets

Because `CODECOV_TOKEN` is org-scoped and `INFECTION_DASHBOARD_API_KEY` is
repo-scoped, each consuming repo's caller workflow should invoke this
reusable workflow with `secrets: inherit` rather than an explicit per-secret
mapping — it transparently pulls from whichever scope actually defines each
secret. Cross-repo `secrets: inherit` works for reusable workflows called
within the same GitHub org.

## Inputs

| Input | Purpose |
|---|---|
| `php-versions` | JSON array of PHP versions for the matrix (default `["8.2", "8.3", "8.4", "8.5"]`). |
| `run-integration` | Run the integration suite as well as the unit suite (default `false`). |
| `composer-options` | Extra flags passed to `composer install`. |
| `db-image` | Container image for the DB service (e.g. `mysql:8.0`). Empty = no DB. |
| `db-env-json` | JSON object of container env vars. |
| `db-port` | Port to expose/map. |
| `db-health-cmd` | Command run via `docker exec` to check readiness. |
| `db-health-retries` | Max health-check attempts (default `30`). Raise for slow-starting engines (Oracle, MSSQL). |
| `db-health-interval-seconds` | Seconds to sleep between health-check attempts (default `2`). |
| `enable-codecov` | Turns on the `codecov` job. |
| `enable-infection` | Turns on the `mutation-test` job. |
| `coverage-php-version` | Which matrix leg is canonical for coverage/mutation. |
| `min-msi` | Minimum MSI (%) required to pass `mutation-test` (default `"10"`). |
| `min-covered-msi` | Minimum covered-code MSI (%) required to pass `mutation-test` (default `"10"`). |
| `test-env-json` | JSON object of extra env vars exported (via `$GITHUB_ENV`) before running tests in `test` and `mutation-test`. |
| `apt-packages` | Space-separated Ubuntu packages `apt-get install`ed at the start of `test` and `mutation-test`, for tools the tests shell out to (e.g. `imagemagick`). Empty (default) skips the step. |

Plus `secrets: CODECOV_TOKEN`, `INFECTION_DASHBOARD_API_KEY` on
`workflow_call` (both `required: false`).

### Overriding a `phpunit.xml.dist` connection setting for CI

PHPUnit's `<env>` element does **not** override an already-set real
environment variable unless `force="true"` is set. This means a caller can
override a `phpunit.xml.dist` default (e.g. a DB hostname that needs to
differ between local dev and CI) via `test-env-json`, without editing that
file or needing `force="true"` (which would break the local-dev default).

For example, a repository whose local `compose.yml` runs PHP and MySQL on the
same Docker Compose network would default its hostname variable to `mysql`
(the container name). In CI the `test`/`mutation-test` jobs run directly on
the runner VM (no `container:` key), so the DB started by the `docker run`
step is only reachable via `127.0.0.1` and the mapped port. Setting
`test-env-json: '{"TESTS_DB_HOSTNAME":"127.0.0.1"}'` in the caller workflow
resolves this without touching `phpunit.xml.dist`.

## Reference example

`contenir/contenir-storage`'s caller workflow
(`.github/workflows/continuous-integration.yml`) is the reference example:
integration tests, Codecov, Infection at 100% MSI, and `apt-packages` for
ImageMagick. For a DB-backed example, `php-db/phpdb-mysql` wires up every
`db-*` input.
