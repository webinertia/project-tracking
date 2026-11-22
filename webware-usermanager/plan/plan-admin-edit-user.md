# Admin Edit User: Implementation Plan

> **Partly superseded (2026-09-17):** role assignment described here as a JSON-encoded array (a
> multi-select, `wrap in [$value]`) becomes a single role string. See
> [`roleid-single-role-direction.md`](roleid-single-role-direction.md).

**Date:** 2026-06-17  
**Branch:** `webware-usermanager-update-user`  
**Target:** `0.1.x`

## Summary

Add admin edit-user modal (HTMX GET fragment) + PATCH update endpoint following the ACL
role-edit pattern. Reuse `SaveUserCommand` by adding `?int $id`; `SaveUserHandler` branches
on insert vs update. Modal closes on success via `HX-Trigger: closeModal` from
`UserListHandler`.

> **Status (2026-06-30):** The actual admin route segment is `webware.admin/user.manager`
> (route name prefix `webware.admin.user.manager.`), not the shorthand `admin/user.manager`
> used in earlier drafts of this plan - all path/route-name examples below have been
> corrected to match. Phase 0 (dashboard widget), the `EditUserModalHandler` GET modal
> (Step 11/12), and the templates (Step 14/15) are **already implemented and confirmed
> working**. `SaveUserCommand`'s upsert shape (Phase 2) is **superseded**: `SaveUserCommand`
> is being replaced by the `User` entity implementing `CommandInterface` directly
> (separate, in-progress refactor); Phase 2 of this plan should not be implemented as written.

## Design Decisions

| Decision | Rationale |
|---|---|
| **Modal dialog** (not full-page form) | Matches ACL role-edit admin pattern; consistent UX |
| **Reuse `SaveUserCommand`** + add `?int $id` | Matches `SaveRoleCommand` pattern (`?int $id` distinguishes create vs update) |
| **Editable fields:** firstName, lastName, email, roleId, active | Core profile fields + role assignment; password excluded (separate reset flow) |
| **Delete `EditUserHandler.php`** (duplicate class) | `EditUserHandler.php` defines `UpdateUserHandler` - same class as `UpdateUserHandler.php`; neither is wired for POST processing; replaced by modal + list |
| **PhpDb event-based extensibility** | PhpDb provides an event cycle at the SQL layer. External modules (e.g. `ims-store`) hook into PhpDb events on the `user` table to modify inflight queries (add store filters, augment columns, inspect results) - no usermanager code changes needed. |

---

## Pipeline Shape

```
GET  /webware.admin/user.manager                   → UserListHandler              (list)
GET  /webware.admin/user.manager/{id:\d+}/modal    → DisableBodyMiddleware
                                                      → EditUserModalHandler       (modal fragment)
PATCH /webware.admin/user.manager/{id:\d+}         → BodyParamsMiddleware
                                                      → ProcessUserMiddleware      (validates, dispatches SaveUserCommand)
                                                      → UserListHandler            (re-renders list, closeModal on success)
POST /webware.admin/user.manager/{id:\d+}/toggle   → ToggleUserActiveHandler      (unchanged)
```

---

## HTMX Interaction Flow

This mirrors the ACL role-edit pattern exactly. See
`src/webware-acl/templates/acl/partials/edit-role-modal.phtml` and
`src/webware-acl/src/Admin/RequestHandler/RoleListHandler.php` for the reference
implementation.

### Shared Modal Shell

The layout (`src/App/templates/layout/default.phtml`) provides a persistent modal
shell **outside `<main>`** that is never torn by HTMX swaps:

```html
<div class="modal fade" id="sharedModal" tabindex="-1" aria-hidden="true">
    <div id="sharedModalDialog"></div>
</div>
```

All modal fragments are swapped into `#sharedModalDialog`. The shell survives every
`<main>` replacement because it lives in the layout, not the body template.

### Loading the Edit Modal (GET)

```
User clicks Edit (pencil icon)
  → hx-get="/webware.admin/user.manager/42/modal"
  → hx-target="#sharedModalDialog"
  → hx-swap="innerHTML"
  → hx-push-url="false"
  → hx-on::after-request="if(event.detail.successful)
       bootstrap.Modal.getOrCreateInstance(
         document.getElementById('sharedModal')
       ).show()"

Server:
  → DisableBodyMiddleware (sets layout=false, body=false)
  → EditUserModalHandler
      → finds user by route param 'id'
      → renders user::edit-user-modal
      → returns HtmlResponse (modal fragment only - .modal-dialog > .modal-content > ...)
```

### Submitting the Form (PATCH)

```
User clicks Save (button with form="editUserForm")
  → form hx-patch="/webware.admin/user.manager/42"
  → hx-target="main"                          ← always targets main, never #sharedModalDialog
  → hx-swap="innerHTML"
  → hx-push-url="false"

Server:
  → BodyParamsMiddleware (parses form body)
  → ProcessUserMiddleware::processPatch()
  → UserListHandler::handle()
```

### Validation Failure (modal stays open)

```
ProcessUserMiddleware returns invalid:
  → $messenger->warning(...)                  ← toast queued
  → $handler->handle($request)                ← passes to UserListHandler

UserListHandler:
  → fetches all users
  → renders user::list-users (full list page)
  → NO CommandResult attribute → no closeModal header
  → returns HtmlResponse

Browser:
  → <main> replaced with list page (behind the modal)
  → Modal stays open (no closeModal trigger, form still visible in #sharedModalDialog)
  → Toast warning appears in #systemMessage (above modal backdrop)
  → User sees toast, dismisses modal, clicks Edit to retry
```

### Validation Success (modal closes)

```
ProcessUserMiddleware returns valid:
  → $this->messageBus->handle(new SaveUserCommand(...))
  → $messenger->success(...)                  ← toast queued
  → attaches CommandResult::class to $request
  → $handler->handle($request)

UserListHandler:
  → fetches all users
  → renders user::list-users (full list page)
  → detects MessageStatus::Success
  → adds header: HX-Trigger: {"closeModal": null}
  → returns HtmlResponse

Browser:
  → <main> replaced with updated list page
  → HTMX processes HX-Trigger: closeModal
  → #sharedModal hides
  → showPendingToasts() fires success toast from swapped body
```

### Key Rules

| Rule | Reason |
|---|---|
| Form always targets `main`, not `#sharedModalDialog` | The response is the full list page; the modal is closed or left open by trigger |
| Modal is never swapped by the PATCH response | It lives in the layout outside `<main>` |
| `closeModal` trigger only fires on `MessageStatus::Success` | On failure the modal stays open so the user can fix and retry |
| `BodyParamsMiddleware` is only on the PATCH route, not the GET modal route | Modal GET routes use `DisableBodyMiddleware` instead |
| Toast uses `$messenger->success(...)` / `$messenger->warning(...)` in middleware | Never in the handler - handlers only render |
| Shared modal shell is Bootstrap 5 `.modal.fade` | Use `getOrCreateInstance` to show, `data-bs-dismiss="modal"` for Cancel |
| Use `#sharedModalLabel` as the modal title `id` | Consistent with all other admin modals |

### Reference Files (ACL implementation)

| Component | File |
|---|---|
| Edit modal GET handler | `src/webware-acl/src/Admin/RequestHandler/EditRoleModalHandler.php` |
| Processing middleware (PATCH) | `src/webware-acl/src/Admin/Middleware/ProcessRoleMiddleware.php` |
| List handler (PATCH response) | `src/webware-acl/src/Admin/RequestHandler/RoleListHandler.php` |
| Modal template | `src/webware-acl/templates/acl/partials/edit-role-modal.phtml` |
| List template | `src/webware-acl/templates/acl/admin-roles.phtml` |
| Route definitions | `src/webware-acl/src/RouteProvider.php` |
| Shared modal shell | `src/App/templates/layout/default.phtml` |

---
## Phase 0: Dashboard Widget (Entry Point) ✅ DONE

The admin dashboard widget provides the UI entry point for the user management
workflow. It appears on `/admin` alongside other module widgets (ACL, etc.).
The pattern mirrors `webware-acl/src/Admin/Dashboard/` exactly.

### Widget Architecture

```
DashboardMiddleware (webware-admin)
  → dispatches RegisterWidgetEvent (PSR-14 collector event)
  → each module's RegisterWidgetListener calls $event->registerWidget($widget)
  → AclWidgetFilterIterator filters by ACL
  → DashboardHandler renders admin::dashboard
    → foreach $widgets → $this->partial($widget->template, $widget)
```

Modules do not create `WidgetInterface`, `RegisterWidgetEvent`, `DashboardMiddleware`,
or `DashboardHandler` - those all live in `webware-admin`. Modules only create:
the widget class, the listener, the listener factory, and the template.

### Widget Contract (`WidgetInterface`: in webware-admin)

```php
interface WidgetInterface extends ResourceInterface
{
    public string $title     { get; }
    public string $resourceId { get; }
    public string $privilege  { get; }
    public string $template   { get; }
    public int    $order      { get; }
}
```

### Step A: Create `Widget`

**File:** `src/webware-usermanager/src/Admin/Dashboard/Widget.php`
**Namespace:** `Webware\UserManager\Admin\Dashboard`

Extra public readonly properties (`$totalUsers`, `$activeUsers`, `$inactiveUsers`)
are passed directly to the template via `$this->partial()`.

### Step B: Create `RegisterWidgetListener`

**File:** `src/webware-usermanager/src/Admin/Dashboard/RegisterWidgetListener.php`
**Namespace:** `Webware\UserManager\Admin\Dashboard`

Injects `UserRepositoryInterface` to fetch real user counts. Calls
`$event->registerWidget(new Widget(...))`.

### Step C: Create `RegisterWidgetListenerFactory`

**File:** `src/webware-usermanager/src/Admin/Dashboard/Container/RegisterWidgetListenerFactory.php`

Builds the `resourceId` by concatenating two config prefixes:

```php
$resourceId = rtrim(
    AdminConfiguration::getAdminRouteNamePrefix($container, self::class)
    . Configuration::getAdminRouteNamePrefix($container, self::class),
    '.'
);
// Result: 'webware.admin.user.manager'
```

### Step D: Create widget template

**File:** `src/webware-usermanager/templates/user/admin-widget.phtml`

Renders a Bootstrap card with user stats (total, active, inactive) and a
"Manage Users" link to the user list route.

### Step E: Register in `ConfigProvider`

Add to `getDependencies()`:
```php
RegisterWidgetListener::class => RegisterWidgetListenerFactory::class,
```

Add `getListeners()` method, wire to `RegisterWidgetEvent::class`.

Add `'listeners' => $this->getListeners()` to `__invoke()` return array.

Add ACL resource `webware.admin.user.manager` with `read` privilege for
`Administrator` role in `getAclConfig()` (if not already present).

### Files Summary (Phase 0)

| Action | File |
|---|---|
| Create | `src/webware-usermanager/src/Admin/Dashboard/Widget.php` |
| Create | `src/webware-usermanager/src/Admin/Dashboard/RegisterWidgetListener.php` |
| Create | `src/webware-usermanager/src/Admin/Dashboard/Container/RegisterWidgetListenerFactory.php` |
| Create | `src/webware-usermanager/templates/user/admin-widget.phtml` |
| Modify | `src/webware-usermanager/src/ConfigProvider.php` |

---

## Phase 1: Clean Up Dead Code

### Step 1: Delete `EditUserHandler.php`

**File:** `src/webware-usermanager/src/Admin/RequestHandler/EditUserHandler.php`

Defines class `UpdateUserHandler` in `Webware\UserManager\Admin\RequestHandler` - duplicate
of the same class in `UpdateUserHandler.php`. Not registered in `ConfigProvider`. Dead code.

### Step 2: Delete `UpdateUserHandler` + factory

**Files:**
- `src/webware-usermanager/src/Admin/RequestHandler/UpdateUserHandler.php`
- `src/webware-usermanager/src/Admin/RequestHandler/Container/UpdateUserHandlerFactory.php`

Replaced by `EditUserModalHandler` (GET modal) + `UserListHandler` (PATCH response).

**Remove from ConfigProvider:** Delete the entry
`UpdateUserHandler::class => UpdateUserHandlerFactory::class` from `getDependencies()`.

---

## Phase 2: Data Layer

### Step 3: Add `?int $id` to `SaveUserCommand`

**File:** `src/webware-usermanager/src/Command/SaveUserCommand.php`

Add `public ?int $id = null` as the last constructor parameter (trailing optional).
`null` = create (existing behaviour); non-null = update.


**File:** `src/webware-usermanager/src/CommandHandler/SaveUserHandler.php`

Branch on `$command->id !== null`:

**Update path:**
- Call `$this->users->update($command->id, [...])` with only: `firstName`, `lastName`,
  `email`, `roleId` (JSON-encoded array), `active`
- Do **not** set password, `verificationToken`, or `tokenCreatedAt`
- Do **not** dispatch `SendVerificationEmailEvent`

Dependency: Add `EventDispatcherInterface` if not already injected (already present for
`SendVerificationEmailEvent`).

Return `CommandResult` with appropriate status in both branches.

### Step 5: Update `SaveUserHandlerTest`
**File:** `test/unit/UserManager/CommandHandler/SaveUserHandlerTest.php` (create if absent)

Add test cases for update path:
- `$id` non-null → `update()` called
- `$id` non-null → verification token not generated
- `$id` null (insert) → `insert()` called

Existing insert tests remain unchanged.

---

## Phase 3: Validation

### Step 7: Create `UserDataFilter`

**File:** `src/webware-usermanager/src/InputFilter/UserDataFilter.php`  
**Namespace:** `Webware\UserManager\InputFilter`  
**Extends:** `Laminas\InputFilter\InputFilter`

Fields (reference `RoleDataFilter` in `src/webware-acl/src/InputFilter/RoleDataFilter.php`):

| Field | Required | Filters | Notes |
|---|---|---|---|
| `id` | No (`allow_empty: true`) | `ToInt` → `ToNull` | Empty string → null |
| `firstName` | Yes | `StringTrim` | |
| `lastName` | Yes | `StringTrim` | |
| `email` | Yes | `StringTrim`, email validator | |
| `roleId` | Yes | `StringTrim`, callback: wrap in `[$value]` | Matches JSON array format in DB |
| `active` | No | `Boolean` | `fallback_value: false`; converts `'1'`/`'on'` → `true` |

### Step 8: Register `UserDataFilter`

**File:** Input filter plugin manager config (follow `RoleDataFilter` registration pattern).

Register `UserDataFilter::class` as an invokable so `InputFilterPluginManager` can resolve
it by FQCN.

---

## Phase 4: Processing Middleware

### Step 9: Create `ProcessUserMiddleware`

**File:** `src/webware-usermanager/src/Admin/Middleware/ProcessUserMiddleware.php`  
**Namespace:** `Webware\UserManager\Admin\Middleware`  
**Implements:** `MiddlewareInterface`  
**Uses:** `HttpMethodProcessorTrait`

**Dependencies:** `MessageBusInterface`

**`processPatch()`:**

1. Get `SystemMessengerInterface` from request attributes
2. Get `InputFilterPluginManager` from request attributes
3. Retrieve `UserDataFilter`; set validation group:
   `['id', 'firstName', 'lastName', 'email', 'roleId', 'active']`
4. Merge route param `id` into parsed body (so InputFilter can validate it)
5. `$filter->setData($parsedBody)`
6. If invalid → messenger warning → pass to handler (re-renders list)
7. If valid → `$filteredData = $filter->getValues()`
8. Build `SaveUserCommand` from filtered data (password is a dummy for the update path)
9. Dispatch via `$this->messageBus->handle($command)`
10. Set messenger success or warning based on `MessageStatus`
11. Attach `CommandResult::class` as request attribute → pass to handler

**`processPost()`:** Falls through to handler (no admin create POST processing in this plan).

### Step 10: Create `ProcessUserMiddlewareFactory`

**File:** `src/webware-usermanager/src/Admin/Middleware/Container/ProcessUserMiddlewareFactory.php`

Resolves `MessageBusInterface` from container.

---

## Phase 5: Request Handlers

### Step 11: Create `EditUserModalHandler` ✅ DONE

**File:** `src/webware-usermanager/src/Admin/RequestHandler/EditUserModalHandler.php`  
**Namespace:** `Webware\UserManager\Admin\RequestHandler`  
**Implements:** `RequestHandlerInterface`

**Dependencies:** `TemplateRendererInterface`, `UserRepositoryInterface`

1. Read `id` from route attribute (use `filter_var` with `FILTER_VALIDATE_INT`, not `(int)`)
2. Call `$this->users->findById($id)`
3. If null → `HtmlResponse` with 404
4. Render `user::edit-user-modal` with `['user' => $user, 'layout' => false, 'body' => false]`
5. Return `HtmlResponse`

### Step 12: Create `EditUserModalHandlerFactory` ✅ DONE

**File:** `src/webware-usermanager/src/Admin/RequestHandler/Container/EditUserModalHandlerFactory.php`

Resolves `TemplateRendererInterface`, `UserRepositoryInterface` from container.

### Step 13: Update `UserListHandler` for PATCH response

**File:** `src/webware-usermanager/src/Admin/RequestHandler/UserListHandler.php`

After rendering the list template, check for `CommandResult::class` request attribute:

- If `CommandResult` is present and `MessageStatus::Success`:
  add response header `HX-Trigger: {"closeModal": null}`

Pattern reference: `RoleListHandler` in `src/webware-acl/src/Admin/RequestHandler/RoleListHandler.php`.

---

## Phase 6: Templates ✅ DONE

### Step 14: Create `list-users.phtml` ✅ DONE

**File:** `src/webware-usermanager/templates/user/list-users.phtml`  
**Reference:** `src/webware-acl/templates/acl/admin-roles.phtml`

Breadcrumb (Admin → Users), page header with user count badge, table with columns:
Name, Email, Role, Status, Actions.

Actions column:
- Edit button with `hx-get` to `webware.admin.user.manager.edit.modal` route, target `#sharedModalDialog`
- Toggle active button with `hx-post` to `webware.admin.user.manager.toggle.update` route

Conventions: `$this->escapeHtml()`, `$this->escapeHtmlAttr()`, `$this->adminUrl()`,
no inline styles.

### Step 15: Create `edit-user-modal.phtml` ✅ DONE

**File:** `src/webware-usermanager/templates/user/edit-user-modal.phtml`  
**Reference:** `src/webware-acl/templates/acl/partials/edit-role-modal.phtml`

Modal dialog fragment:
- `.modal-dialog` → `.modal-content` → header (Edit User + close), body (form), footer (cancel + save)
- Form: `hx-patch` to `webware.admin.user.manager.update` route with `{id}` param, target `main`
- Hidden `<input type="hidden" name="id" value="">`
- Fields: firstName (text), lastName (text), email (email), roleId (select multiple), active (checkbox)
- Pre-populate values with `$this->escapeHtmlAttr($user->...)`
- No inline styles

---

## Phase 7: Route Wiring

### Step 16: Update `RouteProvider`

**File:** `src/webware-usermanager/src/RouteProvider.php`

Remove the old route:
```
GET+POST /{adminSegment}/{id:\d+} → UpdateUserHandler (route name: ...update)
```

Add two new routes:

```php
// GET modal
$routeCollector->get(
    '/' . $this->adminRouteSegment . '/{id:\d+}/modal',
    $middlewareFactory->prepare([
        DisableBodyMiddleware::class,
        EditUserModalHandler::class,
    ]),
    $this->adminRouteNamePrefix . 'edit.modal'
);

// PATCH update
$routeCollector->patch(
    '/' . $this->adminRouteSegment . '/{id:\d+}',
    $middlewareFactory->prepare([
        BodyParamsMiddleware::class,
        ProcessUserMiddleware::class,
        UserListHandler::class,
    ]),
    $this->adminRouteNamePrefix . 'update'
);
```

### Step 17: Update `ConfigProvider`

**File:** `src/webware-usermanager/src/ConfigProvider.php`

**Remove:**
- `UpdateUserHandler::class => UpdateUserHandlerFactory::class` from `getDependencies()`

**Add factories:**
- `EditUserModalHandler::class => EditUserModalHandlerFactory::class`
- `ProcessUserMiddleware::class => ProcessUserMiddlewareFactory::class`

**Update ACL config:**
- Add resource: `webware.admin.user.manager.edit.modal`
- Allow `Administrator` role on `webware.admin.user.manager.edit.modal` with `read` privilege

**Command map:** `SaveUserCommand::class => SaveUserHandler::class` already exists - no change.

---

## Phase 8: Verification

### Step 18: Write Unit Tests

**`ProcessUserMiddlewareTest`:**
- `processPatch()` with valid data → command dispatched, success messenger set
- `processPatch()` with invalid data → warning messenger, handler called
- `processPatch()` with command failure → warning messenger

**`EditUserModalHandlerTest`:**
- Renders modal with user pre-populated
- Returns 404 when user not found

**`UserDataFilterTest`:**
- Valid data passes all filters
- Missing required fields fail
- Invalid email fails
- `id` field: empty → null, numeric → int

### Step 19: Manual Verification

1. Start dev server: `php -S 0.0.0.0:8080 -t public/`
2. Navigate to `/webware.admin/user.manager` → verify user list renders
3. Click Edit on a user → verify modal opens with pre-populated data
4. Change fields, click Save → verify:
   - Modal closes (`closeModal` HTMX trigger)
   - List refreshes with updated data
   - Success toast appears
5. Verify ACL: non-admin users denied access to edit modal and PATCH endpoint
6. Verify invalid data (empty name, invalid email) → warning toast, modal stays open

---

## File Summary

### Delete
- `src/webware-usermanager/src/Admin/RequestHandler/EditUserHandler.php`
- `src/webware-usermanager/src/Admin/RequestHandler/UpdateUserHandler.php`
- `src/webware-usermanager/src/Admin/RequestHandler/Container/UpdateUserHandlerFactory.php`

### Modify
- `src/webware-usermanager/src/Command/SaveUserCommand.php` - add `?int $id`
- `src/webware-usermanager/src/CommandHandler/SaveUserHandler.php` - upsert logic
- `src/webware-usermanager/src/RouteProvider.php` - replace route
- `src/webware-usermanager/src/ConfigProvider.php` - add widget, listeners, ACL, factories
- `src/webware-usermanager/src/Admin/RequestHandler/UserListHandler.php` - closeModal trigger

### Create
- `src/webware-usermanager/src/InputFilter/UserDataFilter.php`
- `src/webware-usermanager/src/Admin/Dashboard/Widget.php`
- `src/webware-usermanager/src/Admin/Dashboard/RegisterWidgetListener.php`
- `src/webware-usermanager/src/Admin/Dashboard/Container/RegisterWidgetListenerFactory.php`
- `src/webware-usermanager/templates/user/admin-widget.phtml`
- `src/webware-usermanager/src/Admin/Middleware/ProcessUserMiddleware.php`
- `src/webware-usermanager/src/Admin/Middleware/Container/ProcessUserMiddlewareFactory.php`
- `src/webware-usermanager/src/Admin/RequestHandler/EditUserModalHandler.php`
- `src/webware-usermanager/src/Admin/RequestHandler/Container/EditUserModalHandlerFactory.php`
- `src/webware-usermanager/templates/user/list-users.phtml`
- `src/webware-usermanager/templates/user/edit-user-modal.phtml`
- `test/unit/UserManager/InputFilter/UserDataFilterTest.php`
- `test/unit/UserManager/Admin/Middleware/ProcessUserMiddlewareTest.php`
- `test/unit/UserManager/Admin/RequestHandler/EditUserModalHandlerTest.php`
- Update `test/unit/UserManager/CommandHandler/SaveUserHandlerTest.php` (if exists)

---

## Excluded from Scope

- Admin create-user POST processing (stub exists but no middleware)
- Password reset / change flow
- Store ownership data is a downstream extension concern, not this component's
- `GuestUser` entity implementation (missing file - tracked in `roleid-audit-results.md`)
- Non-admin user self-service profile editing

---

## Update: 2026-09-10

The `GuestUser` entity listed under "Excluded from Scope" was never implemented and is
no longer part of the design: a guest is a `User` carrying
`UserInterface::GUEST_ROLE`, attached by `IdentityMiddleware` when the session payload
is absent or the account fails the active check. It is the only `GuestUser` reference in
this plan, so nothing else here is affected. See `docs/identity-flow.md` for the current
session and identity contract.
