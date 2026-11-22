# `roleId` Usage Audit

> **Superseded in part (2026-09-17):** the array shape recorded below is the *current* state, not the
target. See [`roleid-single-role-direction.md`](roleid-single-role-direction.md) - `roleId` becomes a
hard `string`, the column becomes `varchar(50)`, and `getRoles()` wraps the string.

> Generated 2026-06-16 from live database (PhpDb MCP) + full codebase grep.

---

## 1. Database Schema

### `user.roleId`

| Attribute | Value |
|---|---|
| Type | `JSON` |
| Nullable | ❌ NOT NULL |
| Default | - |
| Content | JSON array of role name strings, e.g. `["Developer"]` or `["Member","Warehouse"]` |

Source: `data/schema/002_user.sql:47` - `(new Json('roleId', nullable: false))`.
Live DB confirms: `roleId json NOT NULL`.

### `acl_role.roleId`

| Attribute | Value |
|---|---|
| Type | `VARCHAR(50)` |
| Nullable | ❌ NOT NULL |
| Unique | `uq_role_id` |
| Content | Single role name string, e.g. `"Developer"` |

Source: `data/schema/016_acl_role.sql` (migration `Migration016AclRole`).

### `acl_rule.roleId`

| Attribute | Value |
|---|---|
| Type | `VARCHAR(50)` |
| Nullable | ❌ NOT NULL |
| Unique | `uq_rule(roleId, resourceId)` |
| Content | Single role name string, e.g. `"Member"` |

Source: `data/schema/017_acl_rule.sql` (migration `Migration017AclRule`).

### Schema note (historical)

`data/schema/002_user.sql:4` contains a comment: *"role_id is a plain VARCHAR; roles are managed in config, not a DB table."* This comment is stale - the column was changed from `VARCHAR(50)` to `JSON`, and roles are now managed in the `acl_role` DB table, not config.

---

## 2. `User` entity (`webware-usermanager`)

### Property definition (lines 48-53)

```php
public private(set) string|array|null $roleId = null {
    get => $this->roleId ?? null;
    set(string|array|null $value) {
        $this->roleId = $this->parseRoleId($value);
    }
},
```

- **Type:** `string|array|null` - `array` represents `GenericRole[]`, `string` is a raw role name, `null` for guest/unauthenticated
- **Get hook:** returns the raw stored value or `null`
- **Set hook:** delegates to `parseRoleId()`

### `parseRoleId()` (lines 184-203)

```php
private function parseRoleId(string|array|null $value): array|string|null
{
    if (is_string($value)) {
        $decoded = json_decode($value, true);
        if (is_array($decoded)) {
            $roles = [];
            foreach ($decoded as $item) {
                if (is_string($item)) {
                    $roles[] = new GenericRole($item);
                }
            }
            $roleId = $roles;
        } else {
            $roleId = [$value];
        }
    } else {
        $roleId = $value;
    }
    return $roleId;
}
```

- JSON string (from DB) → decoded → each element wrapped in `new GenericRole()`
- Non-JSON string → wrapped in `[$value]`
- Array/null → passed through as-is

### `getRoleId()` (line 152)

```php
public function getRoleId(): array|string|null
{
    return $this->roleId;
}
```

**⚠ Type mismatch:** `RoleInterface::getRoleId()` has no return type declaration (only `@return string` in PHPDoc). `User::getRoleId()` returns `array|string|null`. PHP does not enforce compatibility because the parent interface has no native return type. If `RoleInterface` ever adds `: string`, this would be a fatal error - PHP supports covariant return types (narrowing) only, not contravariant (widening).

### `getRoles()` (line 158)

```php
public function getRoles(): ?array
{
    return $this->roleId;
}
```

**⚠ Return type mismatch with PHPDoc:** PHPDoc says `@return RoleInterface[]|null`, but the method body can also return `string` (if `$this->roleId` is a string). The return type is `?array`, which excludes `string`.

### `withRoleId()` (lines 296-312)

```php
public function withRoleId(array $roleId): self
{
    if (is_string($roleId)) {      // Dead check - $roleId is typed `array`
        $roleId = [$roleId];
    }
    return new self(
        // ...
        roleId: array_merge($this->roleId, array_values($roleId)),
        // ...
    );
}
```

**⚠ Dead code:** `is_string($roleId)` can never be true because the parameter is typed `array`.

**⚠ Type mismatch in PHPDoc:** `@param RoleInterface[]|string[]|string $roleId` describes a union that the signature doesn't accept.

### All `with*` methods pass `roleId: $this->roleId`

Every immutable setter (`withFirstName`, `withLastName`, `withEmail`, `withPasswordHash`, `withActive`, `withDetail`) passes the current `$this->roleId` through unchanged. Lines: 210, 227, 244, 265, 282, 321.

### `GuestUser::getRoleId()` (line 141)

```php
public function getRoleId(): string
{
    return $this->roleId;
}
```

Returns `string` - matches `RoleInterface` contract.

---

## 3. `ims-store` User (extends webware User)

**File:** `src/ims-store/src/Entity/User.php`

```php
final class User extends WebwareUser implements StoreProprietaryInterface
{
    // ... store-owned behavior only; roleId is inherited unchanged ...
}
```

Inherits all roleId behavior from `webware-usermanager`. No roleId overrides.

---

## 4. ACL layer (`webware-acl`)

### `AclInterface::DEVELOPER_ROLE_ID`

```php
final public const string DEVELOPER_ROLE_ID = 'Developer';
```

Used in:
- `Acl.php:172-173` - auto-allows Developer role on all resources
- `AclWidgetFilterIteratorTest.php` - test assertions

### `Acl::addRole()` (lines 37-55)

Extracts `$role->getRoleId()` for persistence:

```php
$roleId = $role instanceof RoleInterface ? $role->getRoleId() : $role;
$this->roleRepository->save($roleId, ...);
```

### `Acl::getRoles()` (lines 101-108)

Iterates the Laminas registry by `roleId` keys:

```php
foreach (array_keys($registry->getRoles()) as $roleId) { ... }
```

### `SingleRoleUserProxy` (`webware-acl/src/Role/`)

Wraps a `UserInterface` with a single `roleId` for ACL iteration:

```php
final class SingleRoleUserProxy implements UserInterface
{
    public function __construct(
        private readonly UserInterface $user,
        private readonly string $roleId,
    ) {}
    public function getRoleId(): string { return $this->roleId; }
    public function getRoles(): ?array { return [$this->roleId]; }
}
```

Used to iterate multi-role users: one ACL `isAllowed` check per role.

### `Role` entity (`webware-acl/src/Entity/Role.php`)

```php
public private(set) int|string|null $roleId = null,
// ...
public function getRoleId(): int|string|null
```

### `Rule` entity (`webware-acl/src/Entity/Rule.php`)

```php
public private(set) ?string $roleId = null,
public function getRoleId(): ?string
```

### `RuleRepository`

Every CRUD method keys on `(roleId, resourceId)`:

| Method | roleId usage |
|---|---|
| `fetchAll()` | `$row['roleId']` in result map |
| `findByRoleAndResource()` | `WHERE roleId = :roleId AND resourceId = :resourceId` |
| `save()` | upsert on `roleId + resourceId` unique key |
| `updateType()` | `WHERE roleId = :roleId AND resourceId = :resourceId` |
| `delete()` | `WHERE roleId = :roleId AND resourceId = :resourceId` |

### `RoleRepository`

| Method | roleId usage |
|---|---|
| `fetchAclRoleRegistry()` | Indexes by `$role->getRoleId()` |
| `fetchDirectChildren()` | `JSON_CONTAINS(parentId, JSON_QUOTE(:roleId))` |
| `save()` | upsert on `roleId` unique key |
| `delete()` | `WHERE roleId = :roleId` |
| `removeFromParents()` | `JSON_CONTAINS(parentId, JSON_QUOTE(:roleId))` |

### ACL Admin Commands

| Command | roleId field |
|---|---|
| `SaveRoleCommand` | `public string $roleId` |
| `DeleteRoleCommand` | `public string $roleId` |
| `SaveRuleCommand` | `public string $roleId` |
| `DeleteRuleCommand` | `public string $roleId` |
| `UpdateRuleTypeCommand` | `public string $roleId` |

### ACL Input Filters

| Filter | roleId field |
|---|---|
| `RoleDataFilter` | `'name' => 'roleId'` |
| `RuleDataFilter` | `'name' => 'roleId'` |

---

## 5. Routes (URL parameters)

**File:** `src/webware-acl/src/RouteProvider.php`

| Route | Parameter |
|---|---|
| `acl.manager.rule.edit` | `{roleId:[^/]+}/{resourceId:[^/]+}` |
| `acl.manager.rule.delete` | `{roleId:[^/]+}/{resourceId:[^/]+}` |
| `acl.manager.rule.type.modal` | `{roleId:[^/]+}/{resourceId:[^/]+}/modal` |
| `acl.manager.role.edit` | `{roleId:[^/]+}/modal` |
| `acl.manager.role.delete` | `{roleId:[^/]+}` |
| `acl.manager.role.update` | `{roleId:[^/]+}` |

---

## 6. Templates (`webware-acl/templates/`)

| Template | roleId usage |
|---|---|
| `admin-acl.phtml` | Displays role columns, passes `roleId` in HTMX delete URLs, toggle values |
| `admin-resources.phtml` | Checkbox IDs, values, labels via `$role->roleId` |
| `add-role-modal.phtml` | `<input name="roleId">`, parent selector via `$role->getRoleId()` |
| `delete-rule-modal.phtml` | `$roleId` variable, HTMX delete URL |
| `protect-route-wizard.phtml` | `@var` doc for role tree |

---

## 7. Tests

| File | roleId usage |
|---|---|
| `test/AppTest/Acl/RuleFilterTest.php` | Validates `roleId` field in input filter |
| `test/AppTest/Acl/StoreOwnershipAssertionPrototypeTest.php` | Mock roles with `getRoleId(): string` returning role names |
| `src/webware-acl/test/integration/ProcessRoleMiddlewareTest.php` | POST body `role_id` field |

---

## 8. Seed data (`data/schema/999_seed.sql`)

```sql
-- user table: JSON array
INSERT INTO `user` (roleId, ...) VALUES ('["Developer"]', ...);

-- acl_role table: plain string
INSERT INTO `acl_role` (roleId, parentId) VALUES ('Guest', JSON_ARRAY()), ...;

-- acl_rule table: plain string
INSERT INTO `acl_rule` (type, roleId, resourceId, ...) VALUES ('Allow', 'Developer', ...);
```

---

## 9. Issues & Discrepancies Summary

| # | Issue | Location |
|---|---|---|
| 1 | `User::getRoleId()` returns `array\|string\|null` but `RoleInterface` has no return type (only `@return string` PHPDoc) - no PHP enforcement, but contract mismatch | `User.php:152` |
| 2 | `User::getRoles()` returns `$this->roleId` which can be `string` (not `array`) - type mismatch with PHPDoc `RoleInterface[]` | `User.php:158` |
| 3 | `withRoleId()` has dead `is_string($roleId)` check - parameter is typed `array` | `User.php:298` |
| 4 | `withRoleId()` PHPDoc says `RoleInterface[]\|string[]\|string` but signature is `array` | `User.php:295` |
| 5 | `user.roleId` is `JSON` (array of strings) but `acl_role.roleId` and `acl_rule.roleId` are `VARCHAR(50)` (single string) - no FK or referential integrity possible | Schema |
| 6 | Stale schema comment: `002_user.sql:4` says role_id is a plain VARCHAR and roles are in config - both are now false | `data/schema/002_user.sql` |
| 7 | `parseRoleId()` creates `GenericRole` objects but `getRoles()` PHPDoc still says `string[]` (was never updated) | `User.php:158` |

---

## Update: 2026-09-10

This audit was taken 2026-06-16 against the pre-extraction entity, before the move to
the RowPrototype implementation. Re-checked against the current package:

- **`GuestUser` does not exist.** A guest is a `User` carrying
  `UserInterface::GUEST_ROLE`; `IdentityMiddleware` attaches one when the session
  payload is absent or the account fails the `CheckUserActiveQuery` check.
- **`User::getRoleId()`** (`src/Entity/User.php`, previously `User.php:152/158`) is
  typed `array|string|null` and returns `$this->roleId` verbatim. The promoted-property
  setter normalizes a JSON string to an array and wraps a bare string, so
  `new User(roleId: 'Guest')` reports `['Guest']`.
- **Single-role consumers are served on the ACL side, not by changing this entity.**
  `Webware\Acl\Role\UserRoleIterator` yields one `Webware\Acl\Role\SingleRoleUserProxy`
  per role (`new SingleRoleUserProxy($this->user, $this->roles[$this->position])`), so
  each yielded object's `getRoleId()` returns a single role id for Laminas
  `RoleInterface` consumers. Issues 1 and 2 above are therefore by design: the entity
  stays multi-role, and the ACL view is one role per proxy.
- Entries 3, 4 and 7 above are resolved in the current entity: `withRoleId()` accepts
  `array|string` (so its string branch is live) and `parseRoleId()` / `GenericRole` no
  longer exist - roleId normalization happens in the setter. All line numbers in the
  tables above refer to the pre-extraction file and have shifted.
