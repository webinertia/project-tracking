# `roleId`: Single-Role Direction

> Recorded 2026-09-17. **Direction only - not implemented.** Every other document in this
> folder that describes `roleId` as an array predates this decision.

---

## Goal

Force the invariant to a `string` and keep it there. The array is never an input, never stored, and
never mutated - it exists at exactly one point: the outbound `getRoles()` return.

```php
public function getRoles(): iterable
{
    return [$this->roleId];
}
```

That return is the **only** array in the role/identity contract surface, because it is the only member
of that surface which deviates from Laminas permissions ACL's shape:

| Contract member | Laminas permissions ACL | Mezzio authentication |
|---|---|---|
| `getRoleId()` | `@return string` | - |
| `getResourceId()` | `@return string` | - |
| `getOwnerId()` | `@return mixed` | - |
| `getIdentity()` | - | `: string` |
| `getRoles()` | *no equivalent - Laminas handles multiple roles by inheritance, not arrays* | `: iterable` |
| `getDetail()` / `getDetails()` | - | `mixed` / `: array` |

Every role-shaped member Laminas defines is a string. The array is Mezzio's; it arrives through
`getRoles()` alone. Wrapping at that one boundary - and nowhere else - is the whole direction.

(`getDetails(): array` also returns an array, but it is Mezzio-only, has no Laminas counterpart, and
is not part of the role identity surface.)

## The problem with the current approach

Approaching `roleId` as an array put the array at the centre and treated the single role as a
derived value. In order, that produced:

- a nullable `array|string|null` property, because `null` had to be representable
- an `array_shift` / index-0 collapsing hook, because only one element was ever legitimate
- a `withRoleId()` that merged arrays and then had to be changed to replace them
- a `getRoleId()` that could not satisfy Laminas' `string` contract while returning the raw property
- `Webware\Acl\Role\SingleRoleUserProxy`, which exists only to present one role to Mezzio's
  `iterable` contract

The direction is inverted: a string, wrapped once, at the contract boundary.

## The direction

- `$roleId` is typed hard to **`string`**. That is all it is.
- `getRoles()` **wraps that string in an array**: nothing more.
- The `roleId` DB column becomes **`varchar(50)`**.
- `SingleRoleUserProxy` is **never needed**: build to the string.
- `getRoles()` is only reached at **boundaries we never touch**, e.g. Laminas permissions
  calling `getRoles()`.

## What changes

| Where | Now | Direction |
|---|---|---|
| `Entity\User::$roleId` | `array\|string` + `array_shift` index-0 hook | `string` |
| `Entity\User::getRoles()` | normalises the property into a list | `return [$this->roleId];` |
| `Entity\User::withRoleId()` | wraps and merges arrays | takes and stores a plain string |
| `Console\UserSchema` - `roleId` | `Json`, `nullable: false` | `Varchar(50)` |
| acl `Role/SingleRoleUserProxy.php` | presents one role as a Mezzio `iterable` | deleted |
| acl `Role/UserRoleIterator.php` | iterates the role list | removed - no longer needed |

Unchanged by this direction:

- `Entity\User::getRoleId(): string`
- ACL role names - proper nouns: `'Guest'`, `'Member'`, `'Administrator'`, `'Developer'`
- `getIdentity()` - the email address, or `'Guest'` for a guest principal
- the authorization boundary; Laminas ACL resolves roles through `getRoleId()`

> **Note (2026-09-17):** `Role/UserRoleIterator.php` in `webware-acl` was previously frozen
> ("make no changes to UserRoleIterator"). That freeze is released - it is to be **removed** along
> with `SingleRoleUserProxy`, since one role needs no iterator.

## Precedent

`acl_role.roleId` is already `VARCHAR(50)` holding a single role name
(see `roleid-audit-results.md`). `user.roleId` is the outlier, not the model.

## Existing data

`roleid-audit-results.md` records live rows holding more than one role, e.g.
`["Member","Warehouse"]`. Moving the column to `varchar(50)` requires deciding what happens to
those rows - the direction assumes exactly one role per user.
