# webware-acl: Session Handoff (2026-09-20)

> Written at the end of the 2026-09-20 session. **webware-acl is the repo currently
> under work.** Read this before touching anything: the live thread is one PR and
> one *unmade design decision*, not a code problem.

---

## TL;DR

- **PR [#46](https://github.com/webinertia/webware-acl/pull/46) is MERGED**: merge
  commit `e311834` on `0.1.x`. Its over-claimed checklist box was corrected first.
- **`1.0.x` is now the repo DEFAULT branch**, created from the post-merge `0.1.x` at
  `e311834` (org convention: components land on `N.N.x`). `0.1.x`, `1.0.x` and
  `origin/1.0.x` are all at `e311834`.
- Six legs are still red. **After cause (B) is fixed, cause (A) alone keeps them red**:
  and (A) is not mechanical, it is the un-made design decision below.
- **Blocked on a design decision, with a seeding gap behind it.** `acl:init-db` writes
  no rule rows for any `user.manager.*` route, so the ACL has no Guest-allow /
  Member-deny pair to ask about - which is why `ForbiddenHandler` fell back to
  `$user->getIdentity()`. See §3a.
- **No edits were made** toward that fix. The superseded 6-file working tree was
  discarded (recoverable from `2e21d0b`).
- **`acl#44` is still OPEN.** `webinertia/webware-core#33` (`Role::getRoles()`) is the
  core-side dependency, and acl cannot consume it until the core floor rises.

---

## 1. Repository / branch state (verified 2026-09-20)

| item | value |
| --- | --- |
| default branch | **`1.0.x`** (flipped 2026-09-20; was `0.1.x`) |
| `0.1.x` / `1.0.x` / `origin/1.0.x` tip | `e311834` - "Merge pull request #46 from webinertia/refactor/remove-role-proxies" |
| local checkout branch | `docs/session-handoff-2026-09-20` |
| working tree | clean; the superseded 6-file attempt discarded (`2e21d0b`) |
| worktrees | main checkout only - `/tmp/acl-verify` removed |

### The superseded 6-file attempt was DISCARDED

The working tree previously held a "userinterface contract conformance attempt" that
*adapted* the two role proxy classes (`getRoles(): ?array` → `getRoles(): iterable`).
#46 *removed* those classes instead, so committing it would have reintroduced 405
lines that #46 deletes. Discarded on 2026-09-20; still recoverable from the dangling
commit **`2e21d0b`** ("wip: userinterface contract conformance attempt", 2026-09-18).

### The autoload fatal is GONE: this section is superseded

Before #46, a fatal at autoload blocked the suite entirely:

```
Fatal error: Declaration of SingleRoleUserProxy::getRoles(): ?array must be
compatible with Webware\Core\UserInterface::getRoles(): Traversable|array
```

That class no longer exists, so **the suite now runs**. Measured on `1.0.x`
post-merge, 2026-09-20: `Tests: 281, Assertions: 703, Errors: 3` - all three cause (A).
Renovate #43/#45 predate the merge and target the now non-default `0.1.x`.

---

## 2. PR #46: what it does

**Title:** Drop the per-role user proxies from ACL authorization
**URL:** https://github.com/webinertia/webware-acl/pull/46
**Branch:** `refactor/remove-role-proxies` → `0.1.x` · **MERGED** (`e311834`)
**Net:** 8 files, **+19 / −405**

`Acl::isAllowed()` unwrapped the caller into one `SingleRoleUserProxy` per entry in
`getRoles()` and granted when any of them passed. That layer existed only to present a
user as a Laminas role - but the contract already carries exactly one:
`Webware\Core\UserInterface` extends both
`Laminas\Permissions\Acl\Role\RoleInterface` (`getRoleId()`) and
`Laminas\Permissions\Acl\ProprietaryInterface` (`getOwnerId()`).

The loop is replaced with direct delegation. **Everything around it is kept**: this
is the part a future session must not "simplify" away:

```php
if (null === $role) { return false; }                    // guest deny - unchanged
$this->load();                                            // DB rule loading - unchanged
if (! $this->hasResource($resource)) { return false; }    // FAIL CLOSED - unchanged
return parent::isAllowed($role, $resource, $privilege);    // was: the proxy loop
```

Removing the loop leaves `UserRoleIterator` and `SingleRoleUserProxy` with no
consumers, so both go - along with their tests, the empty `src/Role/` and
`test/unit/Role/` directories, and the baseline entries that covered them
(11 analyzer, 2 linter).

### Behaviour delta: read before approving

**Any-role-wins is dropped; authorization is now `getRoleId()`.** That is a narrowing,
not a no-op. No user state in this component can observe it, because the only
implementation of the contract returns `[$this->roleId]` from `getRoles()` and
`$this->roleId` from `getRoleId()` - the same value - so the loop always iterated
exactly once with a proxy equivalent to the user. But a host returning more than one
role from `getRoles()` would previously have been granted if *any* role allowed, and
is now judged on the primary role only.

Two tests carried the old semantics and could not survive as written:

- `userWithMultipleRolesIsAllowedWhenAnyRolePasses` - **deleted**; its premise no
  longer exists.
- `userWithNoRolesIsDenied` - re-expressed as
  `userWhoseRoleHasNoMatchingRuleIsDenied`. An empty role list has no representation
  now: the loop's zero-iteration path returned `false`, whereas the direct call must
  resolve the role in the registry or Laminas throws `InvalidArgumentException`. Two
  fixtures that passed `[]` incidentally now register the role they assert nothing about.

---

## 3. CI: the two independent causes of red

Run **`35523375886`** = failure. Of 10 checks:

| check | state |
| --- | --- |
| `qa / Mago (PHP 8.4)` | SUCCESS |
| `qa / Mago (PHP 8.5)` | SUCCESS |
| `qa / Codecov` | SKIPPED |
| `qa / Mutation testing` | SKIPPED |
| `qa / Test (PHP 8.4 \| latest)` | FAILURE |
| `qa / Test (PHP 8.4 \| locked)` | FAILURE |
| `qa / Test (PHP 8.4 \| lowest)` | FAILURE |
| `qa / Test (PHP 8.5 \| latest)` | FAILURE |
| `qa / Test (PHP 8.5 \| locked)` | FAILURE |
| `qa / Test (PHP 8.5 \| lowest)` | FAILURE |

### Cause A: pre-existing stub errors (visible on `locked` / `lowest`)

```
Tests: 281, Assertions: 703, Errors: 3

PHPUnit\Framework\MockObject\IncompatibleReturnValueException: Method getIdentity
may not return value of type null, its declared return type is "string"
```

Tests: `ForbiddenHandlerFactoryTest` ×2, `ForbiddenHandlerTest` ×1. Pre-existing in
both the pre- and post-removal states - **not caused by #46**.

### Cause B: core constant removed, acl has not adopted (visible on `latest`)

```
Tests: 281, Assertions: 670, Errors: 20

Error: Undefined constant Webware\Acl\Acl::DEVELOPER_ROLE_ID
Error: Undefined constant Webware\Core\AclInterface::DEVELOPER_ROLE_ID
```

The other 17 errors are this. `DEVELOPER_ROLE_ID` was **deleted from core** when the
`Webware\Core\Role` enum landed, and acl still references it at:

```
src/Acl.php:219          if ($this->hasRole(self::DEVELOPER_ROLE_ID)) {
src/Acl.php:220          $this->setRule(self::OP_ADD, self::TYPE_ALLOW, self::DEVELOPER_ROLE_ID);
src/Console/AclSchema.php:52    ['roleId' => AclInterface::DEVELOPER_ROLE_ID, 'parentId' => '["Administrator"]'],
src/Console/AclSchema.php:114   'roleId' => AclInterface::DEVELOPER_ROLE_ID,
src/Console/AclSchema.php:135   'roleId' => AclInterface::DEVELOPER_ROLE_ID,
```

The `locked` legs cannot see the break because their lock still pins a core that has
the constant; `latest` resolves one that does not. Tracked by **acl#44**.

---

## 3a. The seeding gap under the blocker

`acl:init-db` writes only `AclSchema::roleSeeds()` (four roles) and `ruleSeeds()`
(Developer's `acl.manager.*` rules). **No rule row is written for any `user.manager.*`
route.** Consequences:

- `Acl::load()` registers resources from the `acl_rule` rows plus the route collector,
  so those routes exist as resources but carry no rule;
- the fail-closed guard (`if (! $this->hasResource($resource)) { return false; }`)
  denies everything the seed does not cover;
- and there is **no Guest-allow / Member-deny pair anywhere in the DB**, so no ACL
  question can distinguish a guest from a denied member - which is exactly why
  `ForbiddenHandler` fell back to `$user->getIdentity()`.

The real policy exists only in IMS `data/schema/999_seed.sql` (hand-run SQL, outside
the ecosystem) and, inert, in `webware-usermanager` `ConfigProvider::getAclConfig()`
(`roles` / `resources` / `allow` / `deny`; nothing reads them - only `resources` is
consumed, by `ResourceListHandler`).

Two related findings:

- **`AclSchema::roleSeeds()` is WET**: a hand-typed duplicate of `Core\Role::getRoles()`,
  already drifted: JSON parent strings vs arrays, and `AclInterface::DEVELOPER_ROLE_ID`
  vs `Role::Developer->value`. Seven hand-typed role names in four lines.
- **Nothing tests the seed → load → decision seam.** `Acl::load()` is private
  (`src/Acl.php:167`) and reached only from `isAllowed()` (`src/Acl.php:122`).
  `AclTest` drives `isAllowed()` against a mocked adapter with hand-supplied rows;
  `InitDBCommandIntegrationTest` reads back only `roleId` (never `parentId`) and never
  calls `isAllowed`; `MessageHandlerMiddlewareIntegrationTest` uses a stubbed ACL.

## 4. THE BLOCKER: an unmade design decision

The user's instruction, verbatim:

> "Yes, fold them in. ALL work to a passing state is expected on this PR"

**Cause A cannot be folded in mechanically**, because the code under test is itself
wrong. `ForbiddenHandler` decides guest-vs-authenticated with `$user->getIdentity()`,
and that check is unreachable - `getIdentity(): string` is non-nullable, and `$user`
can be `null` from `getAttribute()`. The 3 failing tests stub `getIdentity()` to
`null` against `string`, which is PHPUnit correctly objecting to a test that was
written to exercise a branch that cannot exist.

The user's design corrections, verbatim:

> "The name ForbiddenHandler should tell you all you need to know. Forbidden is not
> has Identity"

> "the responsibility of Authenticated is proxied to Authorized because we represent
> non authenticated users as a guest Role. Which means 'Authenticated' is now moot.
> Its completely Role based. Guest/Member are the only possible states to check
> against."

So the branch must be replaced with an **ACL role question**, not an identity
question. Guest/Member is expressed in the ACL by Guest-allow +
**Member-deny overriding an inherited Guest-allow** (Member's parent is Guest).

### Unresolved sub-questions: settle these first

1. Which ACL resource the handler asks about - the concrete candidate is usermanager's
   login route `user.manager.session.read`.
2. Is the discriminator the ACL question, or `getRoleId()`?
3. Does the guest redirect move to `AuthorizationMiddleware`, leaving
   `ForbiddenHandler` to handle denied *members* only (in which case it drops `$user`
   entirely)?

**No code has been changed for this.** Do not start editing until (1) - (3) are answered
by the user.

---

## 5. acl#44: the other open item

```
#44 [OPEN] bug - Align with webware-core: adopt Webware\Core\Role and drop the role proxies
https://github.com/webinertia/webware-acl/issues/44
```

This is what clears Cause B. It means adopting the `Role` enum at the five sites listed
in §3 and dropping the remaining role-proxy surface. Note that #46 already removes
`SingleRoleUserProxy` + `UserRoleIterator`, so what is left for #44 is the constant /
enum adoption in `src/Acl.php` and `src/Console/AclSchema.php`.

---

## 6. Deferred / paired from this session

- **`IdentityMiddleware` payload typing**: deliberately deferred by the user, to land
  in the same pass as acl#44, because the middleware and the ACL pipeline are closely
  intertwined. Sites: `IdentityMiddleware.php:32,36` (callable type), `:53`
  (`@var ... $userInfo`), `IdentityMiddlewareFactory.php:22`. Expect a test cascade
  (`IdentityMiddlewareTest` builds inline factories).
- **Attribute-key gap (publisher-side fix).** `IdentityMiddleware` publishes the user
  under `Webware\Core\UserInterface::class`, but `webware-admin`'s
  `DashboardMiddleware:41` reads `Mezzio\Authentication\UserInterface::class` → always
  `null`. Container aliases do **not** affect PSR-7 attribute keys; they are plain
  class-strings and never container-resolved.
- **CI reusable-workflow ref move `@0.1.x` → `@1.0.x`** across 13 component repos
  (`webware-event` alone is already on `@1.0.x`). It must **ride with each repo's next
  real work**, so the `mago fmt` churn (1.48.1 → 1.49.0 formatter difference) lands in
  the same PR rather than in a tripwire PR of its own.
- **`messagebus-event`** has unreleased BC-breaking work past `2.0.0-beta.1`.

### Fleet open PRs (verified 2026-09-20)

| repo | open PRs |
| --- | --- |
| `webware-acl` | #46 (ours), #45 renovate major-webware, #43 renovate lock-file maintenance |
| `webware-htmx` | #8 renovate lock, #7 renovate webware-tools v1 |
| `webware-navigation` | #7 chore pin webware-tools 1.0.0-beta.2 |
| `webware-console` | #19 renovate lock |
| `webware-core` | **none** |
| `webware-usermanager` | **none** |

---

## 7. How to run things

- Gates: `mago format --check && mago lint && mago analyze && mago guard`
- Unit suite: `composer test` (in the `tooling` container: `docker compose exec -T tooling composer test`)
- Integration suite: `docker compose exec -T tooling composer test-integration` - the
  tooling container bind-mounts the **main** checkout, which is why #46 was verified in
  a separate worktree instead.
- **`format --check` reports 2 files on this repo and that is pre-existing.** The host
  `mago` is 1.49.0 while this repo's CI pins 1.48.1; the 1.48→1.49 formatter differs.
  Verified at the time by stashing the change and reproducing it on the pristine tree.
- Re-verify the runner before trusting it - the host PHP floor and the container
  mount both bite.

---

## 8. Standing rules that apply to this repo

- **The user merges acl PRs.** Do not merge #46. (Explicit: "Dont merge acl.")
- **The user cuts all releases and tags.** Never tag, never `gh release`.
- **Never commit on the default branch.** Feature branch first.
- **Commits require `-s`** (the org PR template demands signoff).
- **`mago.toml` stays the stub.** General guard rules live in `webware-tools`; acl-local
  rules are domain-specific only. Do not re-add general rules locally.
- **Do not suppress mago findings without asking.** `@mago-expect` needs approval first.
- **Never present an inference as fact**: the §4 sub-questions above are exactly the
  kind of thing that must be asked, not assumed.
