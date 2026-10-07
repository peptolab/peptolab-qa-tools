# Migration guide

Three starting points are covered: a Peptolab repository that already uses the upstream
`php-db/phpdb-qa-tools` package, one that carries an inlined copy of its `mago.toml`, and
one still on laminas-coding-standard.

## From php-db/phpdb-qa-tools

The Mago configuration and PHPUnit baseline are the same, so only the package name and
the CI reference change.

### 1. Swap the dev dependency

```sh
composer remove --dev php-db/phpdb-qa-tools
composer require --dev peptolab/peptolab-qa-tools:0.1.x-dev
```

### 2. Point mago.toml at the new package

```toml
extends = "vendor/peptolab/peptolab-qa-tools/mago.toml"
```

### 3. Point CI at the Peptolab workflow

```yaml
uses: peptolab/peptolab-qa-tools/.github/workflows/continuous-integration.yml@0.1.x
```

Codecov upload failures now fail the build, so if `enable-codecov` is on make sure
`CODECOV_TOKEN` is set as a peptolab organisation or repository secret before merging.

If the repository ran a separate job only to install system tools for its integration
tests, fold it into the shared workflow with `run-integration: true` and `apt-packages`.

## From an inlined mago.toml

Some repositories copied the php-db base into their own `mago.toml` rather than extending
it (for example while they still ran PHP 8.1, or to drop the base's `test` analyzer
exclusion). Once the repository runs PHP 8.2 or later, replace the copied shared settings
with `extends = "vendor/peptolab/peptolab-qa-tools/mago.toml"` and keep only the
project-specific ones. Settings the base enforces but the repository relaxed must stay as
local overrides; an analyzer exclusion the base adds can't be removed by an extending file,
so a repository that needs its tests analysed keeps the inlined copy.

## From laminas-coding-standard

Repository-by-repository, in dependency order.

### Prerequisites

Install the Mago binary once per machine (see the [README](../README.md#prerequisite-install-mago)).

### 1. Swap dev dependencies

```sh
composer remove --dev laminas/laminas-coding-standard
# also remove psalm/psalm, vimeo/psalm, or phpstan/phpstan if present
composer require --dev peptolab/peptolab-qa-tools:0.1.x-dev
```

### 2. Remove phpcs artifacts

```sh
rm -f phpcs.xml phpcs.xml.dist .phpcs-cache
```

Also remove any `psalm.xml` / `phpstan.neon` and their baselines.

### 3. Add the minimal mago.toml

Copy the `mago.toml` shown in the [README](../README.md#1-mago). Append
repository-specific `[analyzer]` settings (e.g. `class-initializers`) as needed.

### 4. Adopt the PHPUnit baseline

```sh
cp vendor/peptolab/peptolab-qa-tools/templates/phpunit.xml.dist .
```

Point the suites at `test/Unit` / `test/Integration` and name them `unit` /
`integration`, matching the other Peptolab packages.

### 5. Update composer scripts

Adopt the standard `check` / `cs-check` / `cs-fix` / `static-analysis` / `test` /
`test-integration` / `mutation-test` scripts (see the README).

### 6. Reformat in a single isolated commit

```sh
composer cs-fix
git add -A
git commit -m "Apply mago formatting"
git rev-parse HEAD >> .git-blame-ignore-revs
git add .git-blame-ignore-revs
git commit -m "Ignore mago reformat in git blame"
```

Keeping the mechanical reformat isolated (and recorded in `.git-blame-ignore-revs`)
keeps `git blame` useful.

### 7. Triage remaining findings

Fix or explicitly suppress remaining lint/analyzer findings in follow-up commits.
Large repositories may temporarily relax specific rules in their local `mago.toml`
(child settings override the base) — open a tracking issue to remove the relaxation.

### 8. Update CI

Replace the phpcs/psalm workflow steps with the standard QA workflow (see the README).
