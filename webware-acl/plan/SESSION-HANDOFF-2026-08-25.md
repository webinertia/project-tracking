# Webware ACL Refactor: Session Handoff (2026-08-25)

> Written so the work can be resumed from WSL. Read this first; it is the
> authoritative "where we are" snapshot.

## TL;DR

- Branch `refactor/acl-reorg-1`, PR #11 → `0.1.x`.
- TASK-001, TASK-002, **TASK-003 are DONE** (safety-net tests complete and green).
- This session started the **contracts move to webware-core** to break the
  `acl ↔ usermanager` circular dependency that was failing CI.
- That work is **partially done and uncommitted**, and there are **two known
  broken items** (see "BROKEN / MUST FIX").

---

## 1. Refactor plan status

| Task | Status |
|------|--------|
| TASK-001 baseline validation | ✅ done |
| TASK-002 mago baseline mitigation | ✅ done |
| TASK-003 characterization/safety-net tests | ✅ done |
| TASK-004 Guard perimeter rules | ✅ done (2026-08-28, PR #24) |
| Phase 2 (Http namespace moves) TASK-005/006 | ✅ done (2026-08-27, PR #20) |
| Phase 3 (MessageBus changes) TASK-007/008/009 | ✅ done (2026-08-28, PR #24) |
| Phase 4 (migrations/CLI) TASK-010 | ⬜ blocked (IMS WSL branch) |
| Bug fixes #13 - #18 | ✅ done (2026-08-28) |

### TASK-003 results (committed in `9229b8a`)
- Unit: **244 tests / 471 assertions**, integration: **11 tests / 58 assertions**, 0 skips.
- Line coverage (unit suite, 2026-08-28) **100% (1241/1241)**: LINE COVERAGE GATE MET.
  - Dead code removed: `SingleRoleUserProxy::$id` set hook; `OverviewMiddleware` null/empty-name guard +
    `normalizeRouteList()` flat branch; `Container\CommandHandlerMiddlewareFactory` DELETED.
  - Bug-blocked (5): RESOLVED - `Validator\Assertion` (issue #15) + `Entity\Rule::resolveType()` (issue #17) covered.
  - `ProcessRoleMiddleware` invalid-input branch now unit-covered (`processPostSkipsDispatchOnInvalidInput()`).
- Test helpers: `test/unit/Support/PhpDbAdapterMockTrait.php`, `test/unit/Support/InputFilterHelper.php`,
  `test/integration/Support/FilterManagerFactory.php`.
- Coverage must NOT be tied to integration tests (user directive).

---

## 2. Contracts moved to webware-core (this session)

The circular dep: `webware-acl` required `webware-usermanager` (for `UserInterface`);
`webware-usermanager` required `webware-acl` (for `AclInterface`). CI failed with
"webware/webware-usermanager 0.1.x-dev requires webware/webware-acl 0.1.x-dev … does not match".

### Decision (final)
- Move **both** contracts into webware-core:
  - `Webware\UserManager\UserInterface` → `Webware\Core\UserInterface`
  - `Webware\Acl\AclInterface` → `Webware\Core\AclInterface`
- Contracts MUST `extends` laminas interfaces (typing): `UserInterface extends
  RoleInterface, ResourceInterface, ProprietaryInterface, RowPrototypeInterface`;
  `AclInterface extends LaminasAclInterface` (`isAllowedRoute(?UserInterface, ResourceInterface)`).
- Therefore `laminas/laminas-permissions-acl` is a **runtime `require`** of webware-core (NOT require-dev).
- `laminas/laminas-permissions-acl` ALSO stays a direct `require` of webware-acl (its `Acl` extends
  `Laminas\Permissions\Acl\Acl` directly - implementation dep, not a contract).
- core + acl must bump laminas-permissions-acl in lockstep.

### Cross-repo ordering
1. webware-core: add `UserInterface` + `AclInterface` (DONE - on core `0.1.x`).
2. webware-usermanager: repoint refs, remove acl dep (DONE - tracked in webware-usermanager#5).
3. webware-acl: repoint refs, remove usermanager dep (IN PROGRESS - this repo).

### What was done in webware-acl (uncommitted)
- Deleted `src/AclInterface.php` (moved to core).
- Renamed all `use Webware\UserManager\UserInterface;` → `use Webware\Core\UserInterface;`
  and `use Webware\Acl\AclInterface;` → `use Webware\Core\AclInterface;` across `src/` + `test/`.
- Added `use Webware\Core\AclInterface;` to `src/Acl.php` and `src/ConfigProvider.php`
  (they used the short name via their own `Webware\Acl` namespace).
- `composer.json`: removed `"webware/webware-usermanager": "^0.1.x-dev"` from `require`.
- `composer.lock`: regenerated (removed usermanager + mailer + event + phpmailer;
  webware-core → `c36534f`, webware-admin → `b84d391`).
- `docs/integration-guide.md` updated by the rename (correct).
- Tests pass in the container after the rename: 244/471 + 11/58.

---

## 3. BROKEN / MUST FIX BEFORE CI IS GREEN

### 3a. Trait file NOT renamed (`mago lint` file-name error)
The trait was renamed in code `PhpDbAdapterMock` → `PhpDbAdapterMockTrait`, and all
23 usages + imports were updated. **BUT the file itself is still `test/unit/Support/PhpDbAdapterMock.php`.**

`mago lint` reports:
```
file-name: Trait `PhpDbAdapterMockTrait` should be in a file named `PhpDbAdapterMockTrait.php`
```

**Fix:** rename the file:
```
git mv test/unit/Support/PhpDbAdapterMock.php test/unit/Support/PhpDbAdapterMockTrait.php
```

### 3b. `mago analyze` cannot find the moved contracts (host vendor stale)
`mago analyze` (run on the Windows HOST) reports:
```
non-existent-use-import: Webware\Core\AclInterface does not exist
non-existent-use-import: Webware\Core\UserInterface does not exist
non-existent-class-like: Webware\Core\AclInterface not found
...
```

Root cause: `composer update` was run inside the **tooling container**
(`docker compose exec -T tooling composer update`), so the **container's** vendor
has the new webware-core `c36534f`, but the **host's** `vendor/` / autoloader does not.

**Fix (in WSL):** run `composer install` (or `composer update`) on the host/WSL so the
host vendor picks up webware-core `c36534f` (with `Webware\Core\UserInterface` + `AclInterface`).

### Also verify after the above
- `mago format --check` - clean (already applied `mago format`; 76 files).
- `mago lint` - should be clean once 3a is fixed (baseline was regenerated).
- `mago analyze` - should be clean once 3b is fixed.
- `composer test` + `composer test-integration` - already green in container (244/471, 11/58).

---

## 4. Remaining roadmap (after CI is green)

1. ~~**Line coverage → 100%**~~ - DONE (2026-08-28): dead code removed, `ProcessRoleMiddleware` invalid-input branch unit-tested.
2. ~~**Mutation coverage**~~ - DONE (2026-08-28): 100% MSI + 100% MCC (0 escaped, 0 timeouts).
3. ~~**TASK-004**: Mago Guard rules for the new layout.~~ - DONE (2026-08-28, PR #24).
4. ~~**Phase 2**: namespace moves (TASK-005/006).~~ - DONE (2026-08-27, PR #20).
5. ~~**Phase 3**: MessageBus read migration + unwiring (TASK-007/008/009).~~ - DONE (2026-08-28, PR #24).
6. ~~**Bug fixes** #13 - #18 (post-refactor, separate effort)~~ - DONE (2026-08-28).

**Remaining:**
- **TASK-010 (Phase 4)**: migrations/CLI - blocked on the IMS WSL branch (reference-only).
- **Phase 3 leftovers (webware-acl #22)**: remove unwired acl-local `MessageBus\*` classes; migrate `UpdateRuleTypeHandler` cascade reads to queries; drop `SaveRuleHandler` dead `RoleRepository` dep; honor `SaveRoleHandler::$command->id`.
- **RISK-007**: confirm/fix `ProcessRuleMiddleware` `CommandResult` + `RuleFilter` readonly bugs (separate effort).
- **DEP-005**: `roave/backward-compatibility-check` policy for moved/removed classes.

## 5. Open decisions

1. ~~`Container\CommandHandlerMiddlewareFactory` dead code~~ - DELETED (2026-08-28).
2. ~~Bugs #15 + #17 block 5 coverage lines~~ - RESOLVED: bugs fixed, lines covered (2026-08-28).
3. ~~`SingleRoleUserProxy::$id` set hook and `OverviewMiddleware` dead lines~~ - REMOVED (2026-08-28).

## 6. Bug issues (post-refactor)

- #13 `RuleRepository::findByRoleAndResource()` null guard (pre-existing)
- #14 `AccessDeniedException` extends `final` `RuntimeException` (fatal on autoload)
- #15 `Validator\Assertion` private `$missingAssertion`/`$invalidType` → `error()` throws
- #16 `Validator\Assertion::__construct` missing `nullable` null guard
- #17 `Entity\Rule` constructor promotes `RuleType` → `resolveType()` string branch dead
- #18 `RuleDataFilter` `type` ToEnum case-sensitivity → `TypeError`

## 7. Key files / references

- Plan: `plan/refactor-webware-acl-1.md`
- Session notes: `.github/instructions/memory.instructions.md`
- Cross-repo memory: user memory `/memories/webware-contracts-move.md`
- usermanager tracking issue: webware-usermanager#5
- PR: https://github.com/webinertia/webware-acl/pull/11
