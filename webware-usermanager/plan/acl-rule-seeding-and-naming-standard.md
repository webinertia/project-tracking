# ACL rule seeding and the route-naming standard: landing plan

> **STATUS: COMPLETED**: retained as a decision record; delete just before the `1.0.0` stable tag is cut.
> The naming standard is live in usermanager (`user.*` on `/user`, `admin.user.*` on `/admin/user`) and seeding
> runs through `Acl\RuleSeeds`.

Written 2026-09-29. Scope: `webware-core`, `webware-acl`, `webware-usermanager`, `webware` (app);
touches `webware-log` and the future composer-plugin installer.

## Why this exists

`GET /` was denied, `ForbiddenHandler` redirected to `/user/login`, that route was denied too, and the
browser looped. The root cause is not the middleware:

- `acl_rule` contains no rows for usermanager's routes. Guest, Member and Administrator are therefore
  denied everything; the only working grant is `Acl::load()`'s blanket `Allow Developer`.
- acl's own seeded rows used `acl.manager.*`, which matches no live route name, so they granted nothing.

The underlying defect is that the same identifier was spelled in three independent places: the
`Configuration` constants, the write-only ACL config maps, and the seeded rows. The naming standard
below removes the duplication, and the seeding contract below makes the rows derive from the same
source the routes do.

### The naming standard

- Route names are dot-separated; URI segments are hyphen-separated.
- A component's public routes: `<COMPONENT_NAME>.*` on `/<segment>` where segment is
  `str_replace('.', '-', COMPONENT_NAME)`.
- A component's admin routes: `<adminName>.<COMPONENT_NAME>.*` on `/<adminName>/<segment>`.
- `COMPONENT_NAME` is the single canonical value; once set it does not change.
- Current values: core `webware`, admin `admin`, acl `acl`, usermanager `user`, app `app`.

## Model: how the two columns actually behave (measured)

- `Acl::load()` builds the Laminas resource tree **from the rows**. A row registers its `resourceId`
  with `parent = parentResourceId ?? null`, so a row without a parent is a **root** resource.
- Routes that have **no row at all** are walked up their dotted ancestry and attached to the nearest
  existing resource; if none exists they are registered bare.
- `Acl::isAllowed()` **fails closed**: `if (! $this->hasResource($resource)) return false;`.
- A `parentResourceId` naming a row that does not exist **throws** during `load()` - the source comment
  is explicit that a missing parent should surface rather than be masked. A stale `resourceId` only
  denies, silently.
- The columns are authorization topology **and** presentation topology:
  - `Acl::load()` uses them for inheritance (a grant on a node covers its subtree).
  - `OverviewMiddleware` builds `$parentResourceMap[$rule['resourceId']] = $rule['parentResourceId']`
    and renders every route with the rules of `$parentResourceMap[$name] ?? $name` - a child row
    displays its parent's grants, which is the expand-a-parent-see-children behaviour of the admin UI.
  - `UpdateRuleTypeHandler` cascades a type change to children with no explicit rule.
  - `RoleListHandler` refuses to delete a role that has children.
- `Acl::addResource()` is overridden to throw, so the tree can only ever come from the rows.
- Consequence: a child row that omits its parent simultaneously detaches the route from the subtree
  (authorization) and flattens it out of the tree (presentation).
- The anchor node for a module is `rtrim($prefix, '.')` - e.g. `admin.acl`, `admin.user`. It is a real
  route name (the index route registers it) and it is the unit of grant.

### IMS reference data

`inventory-management-system/data/schema/999_seed.sql` is the working precedent:

- `acl_role`: Guest → Member → (Warehouse, Sales, Collections) → … → Manager → Administrator →
  Developer, upserted with `ON DUPLICATE KEY UPDATE parentId = VALUES(parentId)`.
- `acl_rule` first insert (no `parentResourceId`): Guest allows on the seven public `user.manager.*`
  routes, Member allow `user.manager.logout.read`, Member denies on the same seven, Administrator
  allows on four `webware.admin.user.manager*` ids plus `webware.admin.dashboard.read`, Developer allow
  on `webware.admin.acl.manager`, plus IMS-specific grants.
- `acl_rule` second insert (`parentResourceId` set): Developer's ten acl-manager children, with the
  comment *"explicit rows so BuildAccessControlMiddleware can read the parent/child relationship
  directly without resource-tree traversal"*.
- Administrator's access is therefore **flat per-route rows plus role inheritance**; the only anchor
  usage is Developer's, and its children exist for the middleware's data path.

## Contract: Phase 1 (core)

Layering (decided 2026-09-29): this package publishes the **vocabulary** only - `RuleType`,
`RuleSeed` and `RuleSeedProviderInterface`. The **behaviour**: `RuleSeedIndex`, `RuleSeedValidator`,
`RuleSeeder` and `SeedResult` - belongs to acl, which owns the rules table and already has the route
collector; core has no router dependency, so the route-name checks can only be done properly there.
The description below is the full shape; the last four classes land in acl during Phase 2.

```php
namespace Webware\Core\Acl;

enum RuleType: string { case Allow = 'Allow'; case Deny = 'Deny'; }   // moved from Webware\Acl

final readonly class RuleSeed
{
    /** @param list<string> $assertions */
    public function __construct(
        public RuleType $type,
        public string $roleId,
        public string $resourceId,
        public array $assertions = [],
        public ?string $parentResourceId = null,
    ) {}
}

interface RuleSeedProviderInterface
{
    /** @return list<RuleSeed> */
    public function ruleSeeds(string $adminName): array;
}

final readonly class RuleSeeder          // the only place that knows the table shape
{
    public function __construct(private AdapterInterface $adapter) {}
    public function ruleTableExists(): bool;
    public function seed(RuleSeed ...$seeds): SeedResult;   // validate, then upsert
}

final readonly class SeedResult
{
    public function __construct(public bool $tableMissing, public int $seeded) {}
}
```

- `$adminName` is a parameter, matching `AclSchema::ruleSeeds(string $adminRouteNamePrefix)`; the
  collector resolves it once through `AdminConfiguration::getAdminName()`.
- Discovery key (fleet contribution shape, inside the existing ACL section):

```php
Webware\Core\AclInterface::class => [
    'rule_seed_providers' => [
        Webware\UserManager\Acl\RuleSeeds::class,
        Webware\Acl\Acl\RuleSeeds::class,
    ],
],
```

- **Validate before writing** (the seeder is the only place that sees all providers):
  1. every emitted `resourceId` is either a registered route name or a node emitted by some provider;
  2. every non-null `parentResourceId` resolves to a node emitted by a provider or already present;
  3. no `(roleId, resourceId)` pair is emitted twice with different types;
  4. a provider's anchor descendants are exactly the routes that provider owns.
- Writes upsert on `(roleId, resourceId)` (IMS's `ON DUPLICATE KEY UPDATE`), so the seed stays
  authoritative when an installer re-runs it.
- Missing `acl_rule` ⇒ `tableMissing = true`, no writes, warn and continue - never a hard failure.
- Unverified detail to settle during implementation: how phpdb 0.6 exposes table presence (metadata
  service versus `SHOW TABLES LIKE`).

## Phases

### Phase 0: close what is open

- [ ] Merge acl PR #67 **before any re-seeding**: until it lands, `1.0.x` writes `acl.manager.*` while
      the branch writes `admin.acl.*`, so one command name produces two schemes.
- [ ] Re-seed any database carrying `acl.manager.*` or IMS's `webware.admin.*` anchors.
- [ ] Park each clone back on its default branch after its PR lands.
- [ ] The phpunit MySQL-host PRs (core #45, log #88, acl #66) are independent of this work and gate
      nothing.

### Phase 1: core: the seeding contract, released first

- [x] `RuleType` moved up to `Webware\Core\Acl`; acl switches to it in Phase 2.
- [x] `RuleSeed` (with structural `equals()`) and `RuleSeedProviderInterface` added, with unit tests.
      All four gates green, 68 unit tests.
- [x] Delivered on `feat/acl-rule-seed-contract` - **core PR #47**, open.
- [ ] Merge, then **release** (owner). acl and usermanager cannot finish without a tag to pin.
- The writer and the validator are deliberately not in core: they move to acl, below.

### Phase 2: acl: provider, seed command, corrected seeds

- [ ] Move the validate-then-write machinery in from core - `RuleSeedIndex`, `RuleSeedValidator`,
      `RuleSeeder`, `SeedResult` and their tests. Parked at `/tmp/webware-acl-seeding/` until core's
      contract is released; the seeder already takes a `TableIdentifier` rather than naming a table,
      so in acl that indirection collapses to `Schema::Rules` through `SchemaFactory`.
- [ ] Import core's `RuleType` in place of `Webware\Acl\RuleType` (which then becomes unused - removal
      needs the owner's approval).
- [ ] `AclSchema::ruleSeeds()` returns `RuleSeed`s; acl implements `RuleSeedProviderInterface` - anchor
      `admin.acl` with `parentResourceId => null`, ten children with the anchor as parent.
- [ ] `acl:seed`, registered under `ConsoleInterface::class`, non-interactive: collects every provider
      from `rule_seed_providers`, validates, writes, warns when `acl_rule` is absent.
- [ ] `acl:init-db` keeps DDL + the role chain and reuses the same collector.
- [ ] Switch the `RuleType` import to core.
- [ ] Gate: green, invariant test that seeded ids equal registered route names, PR, release.

### Phase 3: usermanager: naming first, then seeds

- [ ] 3a - naming migration (fold in the two existing uncommitted files, `composer.lock` and
      `src/ConfigProvider.php`, as part of this, not as their own commit):
  - `COMPONENT_NAME = 'user'`; delete the four legacy constants in `src/Container/Configuration.php`;
  - `src/Container/RouteProviderFactory.php` onto `getRouteSegment()`, `getRouteNamePrefix()`,
    `getAdminRouteSegment($adminName)`, `getAdminRouteNamePrefix($adminName)` with
    `AdminConfiguration::getAdminName()` - it currently calls the removed container-taking signatures;
  - rename `user.manager.*` → `user.*` and `/user.manager/…` → `/user/…` across `src`, `test`,
    `templates` (102 literal occurrences); admin routes become `admin.user.*` on `/admin/user`;
  - `getDefaultConfig()` loses the four naming keys; `getAuthenticationConfig()` uses
    `getRouteSegment()`; `getAclConfig()` ids are recomputed from the accessors.
- [ ] 3b - `require php-db/phpdb-mezzio-session` and wire session storage; the session table DDL ships
      in `user:init-db` (idempotent, `ifNotExists`) and is marked provisional: delete it, do not
      migrate it, if the package later provides its own init-db.
- [ ] 3c - provider `src/Acl/RuleSeeds.php`, registered under `rule_seed_providers`:
  - anchors (nodes, `parentResourceId => null`): `user` (public), `admin.user` (admin);
  - Guest `Allow` on the `user` anchor; Member `Deny` on it; Member `Allow user.logout.read`;
    Administrator `Allow` on the `admin.user` anchor **plus** its children with the anchor as parent,
    so they nest in the admin UI as well as inherit.
  - Precedence question to settle from Laminas before finalising: whether a child `Allow` overrides a
    parent `Deny`, which decides whether the seven public/Member pairs collapse into anchor rows plus
    one exception.
- [ ] 3d - `user:init-db` calls core's `RuleSeeder` (warn and skip when `acl_rule` is absent). The
      installer path is `acl:seed`, not this command, because `user:init-db` requires interactive
      arguments.
- [ ] 3e - retire the write-only `allow`/`deny` maps. `resources` is a separate decision: it is read
      by acl's `ResourceListHandler` for `admin-resources.phtml`; the rules screen is DB-driven, so it
      costs nothing there.
- [ ] Gate: gates green, invariant test that seeded ids equal registered route names, PR, release.

### Phase 4: webware (app): the acceptance test

- [ ] `App\Container\Configuration` with `COMPONENT_NAME = 'app'`; home route renamed `app.home`.
- [ ] App provider for app-owned ids (`app.home` for Guest, plus whatever else the app owns).
- [ ] Seed through `acl:init-db` + `acl:seed`, then request `/` as Guest: **no redirect loop** and
      `/user/login` reachable. This is the bug the whole effort exists to fix.

### Phase 5: table owners with nothing today

- [ ] `webware-log`: no `src/Console` and no DDL at all - needs a table builder, an init-db command and
      a provider if any of its routes are gated.
- [ ] Every DDL/seed command registers under `ConsoleInterface::class` so one installer finds them.
- [ ] The session table is solved on our side (usermanager), to be removed if upstream provides it.

### Phase 6: installer readiness

- [ ] Commands are non-interactive and idempotent; `SeedResult` is reported as seeded/skipped/
      tableMissing so the plugin can log it.

## Must not drop

- Renames land in `test` and `templates`, not only `src`.
- `parentResourceId` is written for presentation as well as authorization.
- `rtrim($prefix, '.')` becomes one core accessor instead of the five copies currently in acl's widget
  factory, acl's schema, acl's route provider and usermanager's route provider.
- Stale anchors are invisible until something is denied - re-seed after Phase 2.
- Dependents pin the previous **release**; releases are the owner's, so Phases 2 and 3 each stall at a
  tag boundary by design.

## Open decisions

- Stale rows: report only, or prune rows whose `resourceId` matches no registered route.
- Exit code on integrity violations: recommended hard failure for unknown resource and bad parent,
  warning for everything else.
- Whether the ACL admin's `resources` config array is replaced by a DB-derived list.
- Whether an app-level `db:seed` aggregate exists in addition to `acl:seed`.
