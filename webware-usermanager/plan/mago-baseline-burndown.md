---
goal: Burn down the mago lint and analysis baselines
version: 0.1
date_created: 2026-09-11
owner: Joey Smith
status: 'In Progress'
tags: [mago, baseline, quality, refactor]
---

# Mago baseline burndown

## Context

`lint-baseline.toml` and `analysis-baseline.toml` currently suppress **123 lint** and
**250 analysis** findings. Measured 2026-09-11 on `0.1.x` at `7005072` (after the
Phase 3 MessageBus merge and the composer update):

- `mago lint --ignore-baseline --stats` → 123 findings, 18 distinct codes
- `mago analyze --ignore-baseline --stats` → 250 findings, 40 distinct codes
- Sum of `count` in each baseline equals those totals exactly, and neither run reports
  stale entries - the baselines have **zero slack**, so no finding is hidden beyond the
  ones counted here.

Useful negative result: clearing baseline entries cannot reveal additional findings.
New findings only appear from changed code or a mago/`webware-tools` bump. The work is
therefore purely "fix the 373 known findings", which is why it is split into tranches
with one PR each.

**Current state (2026-09-15, `chore/mago-burndown-2-mechanical`):** lint suppresses
**40** findings across 25 entries (was 123) and analysis **144** across 122 entries (was
250); `mago analyze --verify-baseline` reports the baseline in sync. Tranche 1 landed as
PR #22; tranche 2 is complete on its branch except for the findings awaiting decisions;
tranche 5 is complete.

Both totals moved from the previous measurement (`78e60a6`: 43/27 and 157/128) because
the mailer `1.0.0-beta.4` upgrade landed on this branch: the adapter API change retired
`redundant-null-coalesce` (6), `redundant-cast` (4) and `impossible-nonnull-entry-check`
(1), and two entries were pruned as stale. The domain-command refactor carried on the same
branch brought all seven `CommandHandler` classes to full coverage.

The unapproved Psl `unchecked-exceptions` exemption in `mago.toml` is gone: reverted in
PR #23, merged to `0.1.x` as `8edf30b`, then merged into this branch. Nothing suppresses
`unhandled-thrown-type` any more. All 72 of those findings were fixed in tranche 5, so no
entry of that code remains baselined, and any new throw site must be documented at the
source with the interface its concrete exception implements.

## Policy (locked 2026-09-11)

1. **Safe fixes only.** `mago lint --fix` / `mago analyze --fix` without `--unsafe` or
   `--potentially-unsafe`. Unsafe candidates are surfaced with `--dry-run` and are not
   applied without explicit approval.
2. **Never suppress without approval.** No `@mago-expect` / `@mago-ignore` is added
   without a specific approval for that exact line and code.
3. **Surface anything that could change behavior before changing it.** This explicitly
   includes control-flow restructuring (`no-else-clause`), string building
   (`string-style`), operand reordering (`yoda-conditions`), and any fix where
   equivalence is not self-evident.
4. **Fix at the source**, not at call sites.
5. One issue code per commit; run `mago fmt` first so format churn stays out of the diff.
6. Prune dead baseline entries after each batch:
   `mago lint --remove-outdated-baseline-entries` and the `analyze` equivalent.
7. Re-run the full gate after every batch (see Verification below).

## Verification gate (every batch)

Read both the exit code and the summary line of every check. **Never gate on a `grep` of
the output.** mago phrases its summaries inconsistently (`found 3 issues` vs `No issues
found`), and `INFO`/`WARN` lines go to stderr, so a grep for one wording can silently
match nothing and a broken batch passes. That is exactly how three `no-redundant-use`
findings (a non-zero `mago lint` exit) slipped through the tranche 2 gate.

Count findings with mago's own report limiting, never by grepping rendered output:

| Need | Command |
|---|---|
| One code only | `mago analyze --retain-code <CODE> --reporting-format=short` (repeatable; lint takes the same flag, or `--only <CODE>` to skip other rules) |
| Per-code counts | `mago analyze --stats` (= `--reporting-format code-count`, highest first) |
| Counts from the baseline | `mago inspect-baseline analysis-baseline.toml --group code` |
| Totals only | `--reporting-format=count` |

- `mago format --check` - clean
- `mago lint` / `mago analyze` - exit 0, `No issues found`, and the filtered count must
  match the baseline total
- `mago analyze --verify-baseline` - `Baseline is up to date`. This is the check that
  catches baseline drift a grep cannot see; `--fail-on-out-of-sync-baseline` makes a run
  fail on it
- `mago guard` - clean
- `php -d zend.assertions=1 vendor/bin/phpunit --testsuite "unit test"` - green
- Coverage - 100% classes / methods / lines (`XDEBUG_MODE=coverage`, `--cache-directory /tmp/phpunit-cache-um`)

Integration tests need MySQL and run in the tooling container
(`docker compose exec -T tooling composer test-integration`); they cover
`Repository\UserRepository` only.

### Coverage gotchas (measured 2026-09-11)

- **The `zend.assertions=1` requirement is gone** (tranche 2). Each `CommandHandler`
  used to assert its command type as the first line of `handle()`; with the default
  `zend.assertions=-1` that line never executed, so a plain `phpunit --coverage-text`
  run reported 5 classes at 50% methods (94.44% / 97.15% / 99.44%) and the flag was
  needed to reach 100%. The handlers then moved to `Type\instance_of()->assert()`, an
  ordinary statement. That guard is gone entirely now: message-bus 2 resolves a handler
  from the message's exact concrete class, so `handle()` declares the concrete command
  type as its parameter and the `instance_of` assertion could only ever have fired on a
  direct, non-bus call. Coverage is 100% with or without the flag. The flag is
  still passed for the recorded gate because it costs nothing and keeps the command
  stable.
- **`composer test-coverage` has no `--testsuite` filter**, so it runs unit *and*
  integration. `TESTS_ADAPTER_MYSQL_HOSTNAME` is `127.0.0.1` everywhere - the compose
  database publishes 3306 to the host and CI's service publishes the same port - so the
  same command runs unchanged on the host and in CI.
- **Container runs leave root-owned artifacts in the workspace**, which silently corrupt
  host measurements: `.phpunit.cache/code-coverage/` (root-owned, causes ~95
  `file_put_contents` permission warnings per host coverage run) and `clover.xml`
  (root-owned, mode 644, so host runs cannot overwrite it and stale numbers persist).
  Use `--cache-directory /tmp/phpunit-cache-um` on the host, and do not trust a
  `clover.xml` whose mtime predates the run that supposedly produced it.

## Inventory: lint (123)

| Code | Level | Count | Tranche |
|---|---|---|---|
| `string-style` | help | 30 | 1 |
| `no-else-clause` | help | 21 | Deferred - kept baselined |
| `literal-named-argument` | warning | 11 | 1 |
| `no-fully-qualified-global-class-like` | help | 11 | 1 |
| `yoda-conditions` | help | 11 | 1 |
| `no-redundant-use` | warning | 9 | 1 |
| `excessive-parameter-list` | error | 6 | 4 |
| `ambiguous-constant-access` | help | 4 | 2 |
| `no-isset` | warning | 4 | 2 |
| `prefer-array-spread` | warning | 4 | 2 - **done** |
| `assert-description` | warning | 3 | 2 - **done** (no `assert()` calls remain) |
| `too-many-methods` | error | 3 | 4 |
| `cyclomatic-complexity` | error | 1 | 4 |
| `halstead` | warning | 1 | 4 |
| `inline-variable-return` | warning | 1 | 2 |
| `no-assign-in-condition` | warning | 1 | 2 |
| `no-literal-password` | error | 1 | 2 |
| `no-negated-ternary` | help | 1 | 2 |

## Inventory: analysis (250)

| Code | Level | Count | Tranche |
|---|---|---|---|
| `unhandled-thrown-type` | error | 72 | 5 |
| `imprecise-type` | warning | 29 | 3 |
| `mixed-assignment` | warning | 15 | 3 |
| `missing-override-attribute` | error | 10 | 2 |
| `mixed-argument` | error | 10 | 3 |
| `unsafe-instantiation` | warning | 10 | 3 |
| `possibly-invalid-argument` | error | 9 | 4 |
| `non-existent-property` | error | 8 | 4 |
| `redundant-null-coalesce` | help | 6 | 2 - **done**: retired by mailer `1.0.0-beta.4` |
| `less-specific-argument` | error | 5 | 3 |
| `mixed-return-statement` | error | 5 | 3 |
| `redundant-comparison` | help | 5 | 2 |
| `redundant-cast` | help | 4 | 2 - **done**: retired by mailer `1.0.0-beta.4` |
| `redundant-condition` | warning | 4 | 2 |
| `redundant-type-comparison` | warning | 4 | 2 |
| `uninitialized-property` | error | 4 | 4 |
| `class-must-be-final` | warning | 3 | 2 |
| `invalid-return-statement` | error | 3 | 4 |
| `invalid-type-cast` | warning | 3 | 4 |
| `mixed-method-access` | error | 3 | 3 |
| `mixed-operand` | error | 3 | 3 |
| `redundant-docblock-type` | warning | 3 | 2 |
| `redundant-logical-operation` | help | 3 | 2 |
| `unreachable-else-clause` | error | 3 | 4 |
| `impossible-condition` | warning | 2 | 4 |
| `invalid-property-assignment-value` | error | 2 | 4 |
| `mixed-array-access` | error | 2 | 3 |
| `nullable-return-statement` | error | 2 | 4 |
| `possible-method-access-on-null` | error | 2 | 4 |
| `possibly-false-argument` | error | 2 | 4 |
| `possibly-null-argument` | error | 2 | 4 |
| `possibly-null-operand` | warning | 2 | 4 |
| `property-type-coercion` | error | 2 | 4 |
| `docblock-parameter-narrowing` | error | 1 | 3 |
| `impossible-nonnull-entry-check` | warning | 1 | 2 - **done**: retired by mailer `1.0.0-beta.4` |
| `incompatible-parameter-type` | error | 1 | 3 |
| `less-specific-nested-return-statement` | error | 1 | 3 |
| `missing-api-or-internal` | warning | 1 | 2 |
| `missing-property-type` | warning | 1 | 3 |
| `possibly-undefined-string-array-index` | warning | 1 | 4 |
| `unused-parameter` | help | 1 | 3 |

## Tranches

| # | Scope | Findings | Branch | PR | Status |
|---|---|---|---|---|---|
| 1 | Mechanical lint: autofixable style codes | 93 | `chore/mago-burndown-1-lint-mechanical` | #22 merged | 72 fixed; 21 `no-else-clause` deferred (baselined) |
| 2 | Mechanical analysis + remaining lint: `redundant-*`, `missing-override-attribute`, `no-isset`, `ambiguous-constant-access`, `prefer-array-spread`, `assert-description`, and the long tail of size/level reports | 40 | `chore/mago-burndown-2-mechanical` | - | 12 fixed; `prefer-array-spread` (4) and `assert-description` (3) retired; 3 retired by the `with*` null-merge fix; `unused-parameter` (1) fixed by hand in `f16578c`; 18 redundant imports dropped in `8818adb`; `excessive-parameter-list` 6->5; `redundant-null-coalesce`, `redundant-cast` and `impossible-nonnull-entry-check` (11) retired by the mailer `1.0.0-beta.4` bump; remainder awaiting decisions |
| 3 | Type precision at the source: `imprecise-type`, `mixed-*`, `unsafe-instantiation`, `less-specific-argument`, docblock narrowing | 82 | - | - | Not started |
| 4 | Error-level correctness: `possibly-*`, `non-existent-property`, `uninitialized-property`, `invalid-*`, `unreachable-else-clause`, plus the remaining lint errors | 38 | - | - | Not started |
| 5 | `unhandled-thrown-type` - document `@throws` using the interface the concrete exception implements | 72 | `chore/mago-burndown-2-mechanical` | - | **Done** (`78e60a6`): all 72 fixed. Declaring an exception makes it part of the callee's contract, so a further 36 findings surfaced at callers and were fixed in the same branch until the analysis converged |

Tranche 3 and 4 will move as findings resolve each other: fixing a type at its source
(retiring `imprecise-type` or a `mixed-*` root) typically retires downstream findings in
the same code family, so each tranche's count is re-measured when its branch opens
rather than assumed from this table.

## Deferred / blocked findings

**`no-else-clause` (21)** - deferred 2026-09-11 by owner decision: left baselined until
each site is worked through individually. Mago ships no safe autofix for this rule, and
its stated intent (guard clauses and early returns) is a control-flow restructure rather
than a mechanical rewrite.

| Site group | Count | Why it is not mechanical |
|---|---|---|
| `Entity/User.php` property hooks | 10 | value-producing `if`/`elseif`/`else` inside `set` hooks - needs either an early `return;` after assignment or extracted normalizer helpers, and extraction adds methods while `too-many-methods` (3) and `cyclomatic-complexity` (1) are still pending in tranche 4 |
| `Command/CreateUserCommand.php` hooks | 7 | same shape as above |
| `Http/Middleware/IdentityMiddleware.php` | 2 | authentication path - dropping `else` splits the identity/guest decision across two terminal `return $handler->handle(...)` calls |
| `Http/Admin/Middleware/ProcessUpdateUserMiddleware.php` | 1 | a straightforward guard clause, grouped with the above for consistency |
| `Admin/Dashboard/RegisterWidgetListener.php` | 1 | `if`/`else` counter inside a `foreach`; becomes `continue` or a ternary |

The 21 entries stay in `lint-baseline.toml` so `mago lint` remains green. When any site
is fixed later, prune its entry with `--remove-outdated-baseline-entries` in the same
commit.

**`prefer-array-spread` (4)** - **done** (2026-09-11). Landed as `refactor: use array spread
instead of array_merge`, after the `with*` null-merge fix supplied the typed local that
the raw spread needed. Equivalence was verified case by case: integer keys are renumbered
in both forms and string keys let the later operand win in both, so precedence is
unchanged at every site. Mago ships no autofix for this rule.

**Potentially-unsafe set** - mago classifies these as `--potentially-unsafe`, and owner
policy forbids that flag: `redundant-logical-operation` (2), `unused-parameter` (1).
`unused-parameter` was **resolved by hand** (2026-09-13, `f16578c`): the parameter belongs
to `User::exchangeArray()`, which `RowPrototypeInterface` requires and which always
throws, so it is never read. Renaming it to `$_data` retires the finding without removing
a signature parameter or changing the contract. The rename re-keyed one
`imprecise-type` entry, whose message quoted the old parameter name, so
`analysis-baseline.toml` was regenerated by mago for that entry to match again; the
finding stays baselined with the rest of the `imprecise-type` family (29) for tranche 3
rather than being fixed in isolation. Net baseline 230 -> 229.
`redundant-logical-operation` (2) still needs a per-site decision.
The six `redundant-null-coalesce` findings previously listed here were not a local
problem at all - see the config-contract entry below.

**`redundant-cast` (4), `redundant-null-coalesce` (6) and
`impossible-nonnull-entry-check` (1) - resolved**
(`webware/webware-mailer`). All eleven sat in `SendVerificationEmailListenerFactory` and
shared one cause: a single over-specific `@var` shape annotation on the `config` service.
The annotation claimed every leaf was a present non-null `string`, so the analyzer reported
the four `??` fallbacks and the four `(string)` casts as redundant. The annotation was
false - the `config` service is not validated at runtime, which is exactly why the
fallbacks and casts existed.

Mailer `1.0.0-beta.4` landed the contract tracked as **webinertia/webware-mailer#14**
(governed by **webinertia/webware-tools#20**): `MailerInterface::getAdapter()` is
non-nullable with `setAdapter()` removed, the adapter config is a typed shape, and
`Message` is immutable with `with*()` builders. The listener no longer reads adapter
config at all - it dispatches a domain command through the message bus - so the
annotation and all eleven findings went with it. No entry of these codes remains
baselined.

**Structural lint codes** - `excessive-parameter-list` (5), `too-many-methods` (3),
`cyclomatic-complexity` (1), `halstead` (1), `no-literal-password` (1) need refactors or
design decisions (splitting a parameter list changes call sites; splitting a class changes
the public surface), so they belong with tranche 4 rather than a mechanical pass.

## Findings discovered during the burndown

**`@throws` placement and global-class resolution** (2026-09-13, `78e60a6`). Two rules
that made an apparently-correct pass fail silently:

- A docblock must precede a method's **attributes**. Inserting it between `#[Override]`
  and the method leaves the tag unattached and the finding stays open; this failed 22 of
  29 sites while every file still looked right at a glance.
- A tag naming a **global** class only resolves when that class is imported. In
  `View\Helper\UserUrl`, `@throws InvalidArgumentException` resolves against the current
  namespace and matches nothing; the fix is a `use InvalidArgumentException;` import (an
  FQN `\InvalidArgumentException` would trip `no-fully-qualified-global-class-like`).

Declaring an exception is contagious: `@throws` on a callee makes the analyser report the
same type at every call site, so tranche 5 took three passes (72 -> 22 -> 14 -> 0) rather
than one.

**Fatal in `User::withRoleId()`, false positive in `User::withDetail()`**
(`src/Entity/User.php`). Both call `array_merge()` on a property declared
`array|string|null`, which is why the analyzer reported `possibly-invalid-argument` on
each. Reverting the prototype and exercising both on a default `User` showed the two are
not equivalent:

- `withRoleId()` - genuine `TypeError: array_merge(): Argument #1 must be of type array,
  null given` on `new User()`. Its hook getter is `get => $this->roleId ?? null`, which
  passes `null` through.
- `withDetail()` - not a runtime fault. Its hook getter is `get => $this->details ?? []`,
  which coerces `null` to `[]` before `array_merge()` sees it. The analyzer does not
  narrow through the getter hook, so the finding is defensive-only here.

Fixed with variant B (an `is_array()` guard into a local), chosen over an `(array)` cast so
no cast is introduced on a nullable value. The `withDetail()` guard is retained as a
type-consistent guard against the declared `array|string|null` even though the getter
currently makes the string branch unreachable.

This also unblocks the `prefer-array-spread` rewrite, which becomes safe once the merged
operand is a typed local rather than a nullable property.

## Notes for the analyze tranches

- Consult `vendor/webware/webware-tools/mago-analysis-types.md` before writing or
  correcting any docblock type (shapes, `list<T>`, `int<min,max>`, `class-string<T>`,
  `interface-string<T>`, `key-of<T>`, conditional types, case-sensitivity rules).
- `@throws` targets the exception **interface** where one exists
  (`Psr\Container\ContainerExceptionInterface`, `Mezzio\Template\Exception\ExceptionInterface`);
  keep the concrete class when it implements no interface.
- Bounce `mago analyze --fix` ↔ `mago lint --fix` ↔ `mago fmt`; analyze fixes can emit
  fully-qualified names that the lint rules then normalize.
