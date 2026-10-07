# Rule rationale

Why the non-default choices in the shared `mago.toml` are what they are. The
configuration is inherited unchanged, via
[contenir/contenir-qa-tools](https://github.com/contenir/contenir-qa-tools), from
[php-db/phpdb-qa-tools](https://github.com/php-db/phpdb-qa-tools), where it was
extracted from the `phpdb-validator` refactor — the first php-db component
migrated to Mago — and reflects what survived contact with a real codebase.
Peptolab packages adopt it as-is.

## Formatter

- **`print-width = 120`, 4-space indentation** — carried over from the
  laminas-coding-standard era; matches the existing codebase style.
- **`preserve-breaking-*` family** — intentional line breaks (broken argument
  lists, member-access chains, arrays, conditionals) survive reformatting
  instead of being collapsed when they fit the print width. This dramatically
  reduces diff churn on already-formatted code.
- **`align-assignment-like` / `sort-class-methods`** — opinionated readability
  choices ratified during the validator refactor. `sort-class-methods` makes
  method order deterministic, eliminating "where does this method go" review
  comments.
- **`space-after-logical-not-unary-prefix-operator`** — `! $foo` over `!$foo`,
  matching the laminas-coding-standard convention.

## Linter

- **`minimum-fail-level = "Help"`** — everything the linter reports fails the
  build. Rules relegated to `level = "Help"` are still enforced; the level only
  affects display severity.
- **`integrations = ["laminas", "phpunit"]`** — enables framework-aware rules
  for the two ecosystems the shared base was written for.
- **Naming rules** (`class-name`, `interface-name`, `trait-name` with
  `psr = true`; camelCase methods/variables) — direct continuation of the
  PSR-12/laminas-coding-standard naming policy.
- **`yoda-conditions`** — ratified during the refactor; guards against
  accidental assignment in conditions.
- **`function-name` excludes `src/functions/*`** — free functions in the
  conventional functions directory may use snake_case, matching common
  PHP library practice.
- **`too-many-methods` excludes `test/`** — test cases legitimately accumulate
  many test methods.

## Analyzer

- **`excludes = ["test"]`** — analysis focuses on shipped code; tests are
  covered by the linter (including the `phpunit` integration) and by running.
- **`check-throws` + shared `unchecked-exceptions`** — enforces documented
  `@throws` for checked exceptions. `Error`, `LogicException` (programmer
  errors), and PHPUnit's internal exceptions are unchecked, as is
  `Laminas\Validator\Exception\RuntimeException` which laminas-validator
  throws pervasively.
- **`enforce-class-finality` / `require-api-or-internal`** — every class is
  final unless deliberately opened; every symbol declares whether it is public
  API. These are the strictest choices in the base and the most likely
  migration friction; repositories may relax them locally with a tracking
  issue.
- **`class-initializers`** — the base lists only the one initializer every
  component shares, `PHPUnit\Framework\TestCase::setUp`. Because arrays
  concatenate on merge, repositories append their own (e.g. Laminas'
  `Element::init` / `BaseInputFilter::init`, ...) without repeating the base
  list.

## Versioning policy

Following laminas-coding-standard's precedent: enabling new rules (or
tightening existing ones) lands only in **major** releases of this package.
Rule changes merged in from upstream follow the same policy.
Minor and patch releases may fix documentation, templates, or relax/adjust
rules in a backwards-compatible direction.
