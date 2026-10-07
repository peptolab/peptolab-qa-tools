# peptolab-qa-tools

Shared [Mago](https://mago.carthage.software/) + PHPUnit configuration and reusable CI workflow
for Peptolab packages.

## Fork of contenir/contenir-qa-tools

This repository is a fork of [contenir/contenir-qa-tools](https://github.com/contenir/contenir-qa-tools),
itself a fork of [php-db/phpdb-qa-tools](https://github.com/php-db/phpdb-qa-tools).
The Mago base configuration and the PHPUnit baseline are unchanged from php-db; the
reusable CI workflow carries the Contenir changes:

- **No AI attributions** — an extra `attributions` job fails the build when a pull request
  title, description or commit message carries an AI attribution.
- **Codecov failures fail the build** — `fail_ci_if_error: true`, so a silent upload
  failure can't hide a coverage regression.
- **`apt-packages` input** — installs Ubuntu packages before the `test` and `mutation-test`
  jobs, for system tools the tests shell out to (e.g. `imagemagick`).
- **Pinned runners** — every job runs on `ubuntu-24.04` rather than `ubuntu-latest`.

Changes are merged in from `contenir/contenir-qa-tools`, which in turn tracks php-db:

```sh
git remote add upstream https://github.com/contenir/contenir-qa-tools.git
git fetch upstream
git merge upstream/0.1.x
```

## Prerequisite: install Mago

Mago is a self-contained static binary and is **not** delivered through Composer.
Install it once per machine:

```sh
curl --proto '=https' --tlsv1.2 -sSf https://carthage.software/mago.sh | bash
# or
brew install mago
# or
cargo install mago
```

The shared configuration pins the expected Mago version, so a stale or too-new
binary is flagged immediately.

## Installation

```sh
composer require --dev peptolab/peptolab-qa-tools
```

## Usage

### 1. Mago

Create a `mago.toml` in your repository root that extends the shared base and
adds only the project-specific facts. Peptolab packages keep their suites under
`test/`, which is what the shared base assumes:

```toml
extends = "vendor/peptolab/peptolab-qa-tools/mago.toml"
php-version = "8.3.0"

[source]
paths = ["src", "test"]
includes = ["vendor"]

[formatter]
# Keep `(new Foo())->bar()`: CI formats under each job's PHP version, and the
# unparenthesised form PHP 8.4+ allows does not parse on 8.3, the minimum.
parentheses-around-new-in-member-access = true
```

Merge semantics: nested tables merge deeply, arrays concatenate (parent first),
and child scalars win — so you can tighten or relax individual rules locally
without forking the whole standard.

### 2. PHPUnit

Copy the strict baseline into your repository (PHPUnit has no config inheritance):

```sh
cp vendor/peptolab/peptolab-qa-tools/templates/phpunit.xml.dist .
```

The template's suites point at `test/unit` and `test/integration`. Peptolab packages
use `test/Unit` and `test/Integration` with suites named `unit` and `integration`, so
adjust the `<testsuites>` block after copying.

### 3. Composer scripts

Add the standard scripts to your `composer.json`:

```json
{
    "scripts": {
        "check": ["@cs-check", "@static-analysis", "@test", "@test-integration"],
        "cs-check": ["mago format --check", "mago lint"],
        "cs-fix": ["mago format", "mago lint --fix"],
        "static-analysis": "mago analyze",
        "test": "phpunit --colors=always --testsuite unit",
        "test-integration": "phpunit --colors=always --testsuite integration",
        "test-coverage": "phpunit --colors=always --coverage-clover clover.xml",
        "mutation-test": "infection"
    }
}
```

### 4. CI

This repository ships a reusable CI workflow
([`.github/workflows/continuous-integration.yml`](.github/workflows/continuous-integration.yml))
with five jobs: `attributions` (no AI attributions), `mago` (format/lint/analyze/guard),
`test` (unit + optional integration, across a `php x [lowest, locked, latest]` matrix),
and two optional downstream jobs, `codecov` and `mutation-test`, both gated on `test`
succeeding. A consuming repository's entire CI file becomes:

```yaml
# .github/workflows/continuous-integration.yml
name: "Continuous Integration"

on:
  push:
  pull_request:

jobs:
  qa:
    uses: peptolab/peptolab-qa-tools/.github/workflows/continuous-integration.yml@0.1.x
    secrets: inherit
    with:
      php-versions: '["8.3", "8.4", "8.5"]'
      run-integration: true
      # Only when the tests shell out to system tools.
      apt-packages: "qpdf poppler-utils"
      # Codecov needs CODECOV_TOKEN; Infection needs INFECTION_DASHBOARD_API_KEY.
      enable-codecov: false
      enable-infection: false
```

See [Workflow architecture](docs/workflow-architecture.md) for the full job
graph, the DB-service mechanics, and the Codecov/Infection secrets wiring.

## Documentation

- [Migration guide](docs/migration.md) — moving a Peptolab repository onto the shared toolchain.
- [Rule rationale](docs/rules.md) — why the non-default choices are what they are.
- [Workflow architecture](docs/workflow-architecture.md) — job-split design for DB-backed integration tests, Codecov, and Infection.

## License

BSD-3-Clause. See [LICENSE](LICENSE).
