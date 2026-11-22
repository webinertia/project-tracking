# Agent plan: admin create user (DeepSeek handoff)

> **STATUS: COMPLETED**: every task was executed and merged (PR #80 admin create-user, PR #82 render-only
> list handlers). Retained as a decision record; delete just before the `1.0.0` stable tag is cut.

Design is settled in [`plan-admin-create-user.md`](plan-admin-create-user.md). This file is the **work list**: do
exactly what a task says, nothing else. Section numbers below refer to that plan.

## Rules for every task (read first, apply always)

1. Read `vendor/webware/webware-tools/agent-working-agreements.md` in the repo you work in before starting.
2. **Scope:** touch only the files a task names. No cleanup, no refactors, no extra tests, no extra docs.
3. **Never delete, rename or move a file.** If you think one must go, stop and report it in one line.
4. **Git:** one branch and one PR per task, into `1.0.x`. Never push to `1.0.x`. Never create tags or releases.
   Commit with `git commit -s` as `Joey Smith <jsmith@webinertia.net>`; never override author or committer.
   Never use `--no-verify`. Do not merge the PR; report its URL.
5. **Code style:** `declare(strict_types=1)`, named arguments on every call, `match` for discrete selection,
   no `#[Override]` on bus handlers, no `Webware\Acl\…` imports in usermanager.
6. **Tests:** PHPUnit 13. Every test class has `#[CoversClass]` and `#[CoversMethod]` per method. `createStub()`
   when no expectations, `createMock()` only with `expects()`. Floors: 100% line coverage, MSI 95.
7. **Gates (run in this order; judge by exit code, not by the last line of output):**
   `mago fmt --check`, `mago lint`, `mago analyze`, `mago guard`, `composer test`, `composer mutation-test`.
   Fix findings **at the source**. Never add a baseline entry, `@mago-ignore`, or `@mago-expect`.
8. **Do not run application code** (server, `php -r` that boots the container, migrations) while a debugger may
   be attached. Check first: `ss -ltnp | grep -E ':(9000|9003)'`. A listener means stop and report.
9. **Stop and report** (do not improvise) when: a gate fails and the fix is not obvious at the source; the task
   text and the code disagree; a file needs deleting; a task needs a design decision; you need to touch a file
   outside the task's scope.

**Report format (and nothing else):**

```
Task: <id>
PR: <url>
Gates: fmt=<0|1> lint=<0|1> analyze=<0|1> guard=<0|1> test=<0|1> mutation=<0|1>
Changed files: <list>
Stopped because: <only if stopped, one line>
Noticed (max 3, one line each, no fixes proposed): <list or "none">
```

No recap, no summary, no suggestions.

## Dependency order

`T0 → T1`, then `T2`, `T3` independent of each other, `T3b` before `T4`, `T4` after the webware-tools tag (above) and independent of
the rest (all in usermanager, separate branches), `T5` needs
`T1`, `T3`, `T4`. `T6` is in the app repo and needs `T2`; it is done. `T7` starts after #76 is merged. `T8` is the
owner's session and must land before a user can actually be created. Do not start a task before its dependencies
are merged.

---

## T0: Onboard `webware-validator` (plan 16.1)

**Repo:** `webinertia/webware-validator`, clone to `~/github.com/webinertia/webware-validator`, branch
`chore/onboard-webware-validator` from `1.0.x`.

Steps:

1. Clone the repo. Read its `README.md` rename checklist and follow it **exactly**: composer `name`
   `webware/webware-validator`, description, `extra.laminas.config-provider`
   `Webware\Validator\ConfigProvider`, PSR-4 roots `Webware\Validator\` → `src/`,
   `WebwareTest\Validator\` → `test/unit/`, `WebwareTestIntegration\Validator\` → `test/integration/`,
   namespaces and file headers in `src/` and `test/`, `.github/copilot-instructions.md` title, badges,
   `LICENSE` holder text only if the checklist says so.
2. `composer require --dev webware/webware-tools:^1.0.0-beta.8` (adjusts the constraint and lock).
3. Run all gates (rule 7).

Done: all gates exit 0; `git grep -i skeleton` returns nothing outside `composer.lock`.
Stop if: the checklist is missing or contradicts the above.

## T1: `PasswordRequirement` (plan 16.2)

**Repo:** `webware-validator`, branch `feat/password-requirement`, after T0 is merged.
**Files (create):** `src/PasswordRequirement.php`, `src/Container/PasswordRequirementFactory.php`,
`test/unit/PasswordRequirementTest.php`, `test/unit/Container/PasswordRequirementFactoryTest.php`.
**Files (edit):** `src/ConfigProvider.php`, `test/unit/ConfigProviderTest.php`.

Reference (read only, do not copy its structure blindly): `~/github.com/axleus/axleus-validator/src/PasswordRequirement.php`.
Installed base class: `vendor/laminas/laminas-validator/src/AbstractValidator.php` (v3).

Required shape:

```php
final class PasswordRequirement extends AbstractValidator
{
    public const string INVALID_LENGTH_COUNT  = 'invalidLengthCount';
    public const string INVALID_UPPER_COUNT   = 'invalidUpperCount';
    public const string INVALID_LOWER_COUNT   = 'invalidLowerCount';
    public const string INVALID_DIGIT_COUNT   = 'invalidDigitCount';
    public const string INVALID_SPECIAL_COUNT = 'invalidSpecialCount';

    /** Supported special characters. */
    public const string POSSIBLE_SPECIAL_CHARS = '/[].\'"+=\[\\\\@_!\#$%^&*()<>?|}{~:-]/';

    protected array $messageTemplates = [ /* same five texts as axleus, %length% %upper% %lower% %digit% %special% */ ];

    /** @var array<string, string> */
    protected array $messageVariables = [
        'length' => 'length', 'upper' => 'upper', 'lower' => 'lower', 'digit' => 'digit', 'special' => 'special',
    ];

    protected readonly int $length;
    protected readonly int $upper;
    protected readonly int $lower;
    protected readonly int $digit;
    protected readonly int $special;

    public function __construct(array $options = []) { /* read the five keys (default 0), cast to int,
                                                           then parent::__construct(options: $options) */ }

    public function isValid(mixed $value): bool { /* setValue((string) $value); check each rule; every
                                                     failed rule calls $this->error(messageKey: ...);
                                                     return true only when none failed */ }
}
```

Rules: length uses `mb_strlen`; upper/lower/digit/special count matches with `preg_match_all` using
`/[A-Z]/`, `/[a-z]/`, `/[0-9]/` and `POSSIBLE_SPECIAL_CHARS`; a rule with count `0` is skipped; a non-string
value is cast to string. All failed rules are reported, not just the first.

`ConfigProvider`: config key `PasswordRequirement::class => ['length' => 8, 'upper' => 1, 'lower' => 1,
'digit' => 1, 'special' => 1]` (**provisional defaults - owner to confirm**), `validators` → `factories` →
`PasswordRequirement::class => Container\PasswordRequirementFactory::class`. The factory merges
`config[PasswordRequirement::class]` with the call-site `$options` (call-site wins) and returns
`new PasswordRequirement(options: $merged)`; when no `config` service exists it uses `$options` only.

Tests: each rule alone; all together; boundary (exactly N and N−1); multibyte input; non-string input; option
override through the factory; every message template with its placeholder text; `ConfigProviderTest` updated.

Done: gates exit 0; coverage 100%; MSI ≥ 95. Stop if: a gate cannot pass without a baseline entry.

## T2: Verification listener wiring (plan F3, F4)

**Repo:** `webware-usermanager`, branch `fix/wire-verification-email-listener`.
**Files (edit):** `src/ConfigProvider.php`, `test/unit/ConfigProviderTest.php`. Read first:
`src/Listener/SendVerificationEmailListener.php`, `src/Listener/Container/SendVerificationEmailListenerFactory.php`,
`src/Container/Configuration.php`.

Steps:

1. In `ConfigProvider::getListeners()`, register `SendVerificationEmailListener::class` for
   `SendVerificationEmailEvent::class`, using **the same registration shape** the existing
   `RegisterWidgetListener` entry uses. Priority `1`.
2. Extend the config test to assert the event has exactly that listener.
3. Do **not** add config keys to the package defaults (`base_url` and `verification_email_subject` are
   app-owned; they come in T6).

Done: gates exit 0. Stop if the listener's factory needs a service no provider registers.

## T3: Assignable roles query (plan 7.1, 7.3)

**Repo:** `webware-usermanager`, branch `feat/assignable-roles-query`.
**Files (create):** `src/Query/FetchAssignableRolesQuery.php`, `src/QueryHandler/FetchAssignableRolesHandler.php`,
`src/QueryHandler/Container/FetchAssignableRolesHandlerFactory.php`, and one test per class.
**Files (edit):** `src/ConfigProvider.php`, `test/unit/ConfigProviderTest.php`.
Model on the existing query, handler and factory (`FetchUserByEmailQuery`, its handler and factory) - same
interfaces, same attributes, same registration, same constructor-argument style.

Behavior. Input: `string $actorRoleId`. Source: `Webware\Core\AclInterface::getRoles()` →
`array<string, string[]>` (role id → direct parent role ids). Output: `new QueryResult(...)` with status
Success and payload `list<string>`.

```
assignable(actor) = ({actor} ∪ ancestors(actor)) \ {Webware\Core\UserInterface::GUEST_ROLE}
```

`ancestors` is a depth-first walk over the parent lists with a visited set (must terminate on a cyclic
registry). Return roles in the registry's key order. An actor role that is not a key of the registry returns
`[]`. The handler has no `try/catch`.

Tests: Developer, Administrator, Member on the default registry (Guest → Member → Administrator → Developer);
an IMS-style registry with extra middle roles; Guest excluded; unknown actor returns `[]`; a cyclic registry
terminates; result order follows the registry.

Done: gates exit 0. Stop if: `Webware\Core\UserInterface::GUEST_ROLE` does not exist.

## T3b: Move to current `php-db/phpdb` (exception marker)

**Repo:** `webware-usermanager`, branch `chore/phpdb-exception-marker`. **Do this before T4.**
Upstream (`php-db/phpdb` 0.6.x, commit `01a9406b`, 2026-09-29) removed `PhpDb\Sql\Exception\ExceptionInterface` and
`PhpDb\TableGateway\Exception\ExceptionInterface`; **`PhpDb\Exception\ExceptionInterface` is now the only marker.**
**Files (edit):** `composer.lock` only for dependencies (see step 1), `src/Repository/UserRepository.php`,
`src/Repository/UserRepositoryFactory.php`. Touch nothing else.

Steps:

1. `composer update php-db/phpdb` (no other package named; **no `-W`**). Report the resulting `php-db/phpdb`
   commit and every other package whose version changed. Stop if anything other than `php-db/phpdb` and its own
   dependencies moves.
2. `UserRepository.php`: delete `use PhpDb\Sql\Exception\ExceptionInterface as SqlException;` and replace every
   `@throws SqlException` with `@throws ExceptionInterface` (the `PhpDb\Exception\ExceptionInterface` import already
   exists). Do not duplicate a `@throws` line that already names it.
3. `UserRepositoryFactory.php`: delete the `SqlException` and `TableGatewayException` imports; add
   `use PhpDb\Exception\ExceptionInterface as PhpDbException;` in alphabetical position; replace the
   `@throws SqlException` and `@throws TableGatewayException` lines with one `@throws PhpDbException`.
4. Do not change behavior, signatures or tests.

Done: gates exit 0 with **no new baseline entries** and the previous analyzer baseline unchanged; `git grep -n
'Sql\\Exception\\ExceptionInterface\|TableGateway\\Exception\\ExceptionInterface' src test` returns nothing. Stop if:
any analyzer finding remains in a file outside the two named files, or a gate fails for a reason not in this list.

## T4: Validators and the create input filter (plan 8.1, 16.3)

**Repo:** `webware-usermanager`, branch `feat/create-user-validators`. Needs webware-tools `1.0.0-beta.8` or later
(its guard allows `PhpDb\Validator\**`).
**Files (create):** `src/Validator/AssignableRoleValidator.php`, `src/Validator/Container/NoRecordExistsFactory.php`,
`src/Validator/ConfigProvider.php`, `src/InputFilter/CreateUserDataFilter.php`, and one test per class.
**Files (edit):** `composer.json` + `composer.lock` (add `php-db/phpdb-validator`), `src/ConfigProvider.php`,
`test/unit/ConfigProviderTest.php`, and `test/unit/Listener/Container/RegisterWidgetListenerFactoryTest.php` only
if step 6 applies.
Model the filter on `src/InputFilter/UpdateUserDataFilter.php` (same base class, `SystemMessageTrait`,
`init()` shape, docblock array shape).

Steps:

1. `composer require --dev webware/webware-tools:^1.0.0-beta.8`, then add `php-db/phpdb-validator`. It is not on
   Packagist: add a `repositories` entry `{"type": "vcs", "url": "https://github.com/php-db/phpdb-validator"}`
   and require `0.1.x-dev`. **Do not use `-W` / `--with-all-dependencies`** (T3b already moved `php-db/phpdb`;
   run `composer update php-db/phpdb-validator` only). `php-db/phpdb-validator` `0.1.x` now requires
   `laminas-translator ^2.0`
   (php-db/phpdb-validator#8), so `laminas/laminas-translator` must stay at 2.x; if composer proposes a
   downgrade, stop and report.
2. `NoRecordExistsFactory`: signature `__invoke(ContainerInterface $container, string $requestedName,
   ?array $options = null)`; builds `PhpDb\Validator\NoRecordExists` with
   `['adapter' => $container->get(PhpDb\Adapter\AdapterInterface::class), 'table' => 'user', 'field' => 'email']`
   merged with `$options` (call-site wins).
   **Registration:** the guard allows `PhpDb\Validator\**` only from `*\InputFilter\**` and `*\Validator\**`, so the
   root `ConfigProvider` must not name any `PhpDb\Validator` class. Put the `validators` config (the factory
   for `PhpDb\Validator\NoRecordExists::class` and the `AssignableRoleValidator` entry) in
   `Webware\UserManager\Validator\ConfigProvider` (`__invoke(): array` returning `['validators' => [...]]`), and
   have the root `ConfigProvider` merge it (`(new Validator\ConfigProvider())()`), exactly as it composes its other
   sections. If guard rejects that class name or place, stop and report.
3. `AssignableRoleValidator` (`AbstractValidator` v3, `final`): `isValid(mixed $value, array $context = []): bool`
   is valid only if `$value` is a string present in `$context['assignableRoles']`; one message template.
   Registered in `Validator\ConfigProvider` (step 2).
4. `CreateUserDataFilter` fields: `firstName`, `lastName` (required, `StringTrim`); `email` (required, `StringTrim`,
   `StringToLower`, `EmailAddress`, `NoRecordExists`); `roleId` (required, `StringTrim`,
   `AssignableRoleValidator`). **No password fields, no `active` field.** Register under `input_filters` the
   same way `UpdateUserDataFilter` is registered.
5. Tests as below.
6. **Formatting exception (the only one):** the mago bump to 1.51.0 makes whole-repo `mago fmt --check` flag
   `test/unit/Listener/Container/RegisterWidgetListenerFactoryTest.php`, which this task does not otherwise touch.
   Run `mago fmt` on that one file only. Its diff must be whitespace/line-wrapping only; if it is anything else,
   revert and report.

Tests: factory builds a validator with the adapter (stub the container); `AssignableRoleValidator` valid,
invalid, missing context; filter: each field, role outside the set, missing required fields.

Done: gates exit 0. Stop if: `mago guard` still rejects `PhpDb\Validator\**` after the webware-tools bump - report
it; do not work around it and do not edit `mago.toml`.

## T5: Create modal, route split, command changes (plan 6, 7.2, 9)

**Repo:** `webware-usermanager`, branch `feat/admin-create-user-modal`, after T1, T3, T4 are merged.
Read first, and follow as the pattern: `RouteProvider`, `Http/Admin/Middleware/ProcessUpdateUserMiddleware.php`,
`Http/Admin/RequestHandler/{UpdateUserHandler,UpdateUserModalHandler,CreateUserHandler}.php` and their
factories, `Command/CreateUserCommand.php`, `CommandHandler/CreateUserHandler.php`, `Acl/RuleSeeds.php`,
`templates/default/user/update-user-modal.phtml`.

**Held back - do not do in this task:** CSRF middleware, the shared `role-select` partial's use in the edit modal.
The password-set flag and the activation set-password step **landed already** (webinertia/webware-usermanager#78); do
not rebuild them.

Steps:

1. `RouteProvider`: add `admin.user.create.modal` (`GET`, path `/admin/user/create/modal`, pipeline
   `DisableBodyMiddleware`, `CreateUserModalHandler`) mirroring `admin.user.update.modal`. Make
   `admin.user.create` `POST`-only with pipeline `ProcessCreateUserMiddleware`, `NotificationMiddleware`,
   `CreateUserHandler`. **Remove the `navigation` option** from `admin.user.create`.
2. `Acl\RuleSeeds` and `ConfigProvider::getAclConfig()`: seed and register `admin.user.create.modal` exactly as
   `admin.user.update.modal` is. Update `RuleSeedsTest` and `ConfigProviderTest`.
3. `Http\Admin\RequestHandler\CreateUserModalHandler` (+ factory in `…\Container\`): reads the actor from the
   request attribute `Webware\Core\UserInterface::class` (never `Mezzio\Authentication\UserInterface`), dispatches
   `FetchAssignableRolesQuery` on the bus (no actor ⇒ empty list), renders `user::create-user-modal` with
   `assignableRoles`.
4. `Http\Admin\Middleware\ProcessCreateUserMiddleware` (+ factory in `…\Middleware\Container\`), using
   `HttpMethodProcessorTrait` like `ProcessUpdateUserMiddleware`: on POST fetch the assignable roles; build the
   filter data as `[...$body, 'assignableRoles' => $roles]` (server key **after** the spread); validate with
   `CreateUserDataFilter`; on failure attach a `Http\Admin\CreateUserState` (readonly: `list<string>
   $assignableRoles`, `array<string, list<string>> $errors`, `array<string, string> $old`, `?CommandResult
   $result`) and continue; on success dispatch `CreateUserCommand` built from the validated values plus
   `verificationToken => Uuid::uuid7()->toString()`, `active => false`, and an **unusable random password hash**
   (`password_hash(bin2hex(random_bytes(32)), PASSWORD_DEFAULT)`), attach the `CommandResult`, and log at `info`
   (actor id, new user id, role). A role outside the set is logged at `warning`.
5. `Command\CreateUserCommand`: implement `NotificationCapableInterface` (copy the mechanics from
   `UpdateUserCommand`), `successMessage` `User created.`, `failureMessage` `User could not be created. Please
   try again.`
6. `CommandHandler\CreateUserHandler`: dispatch `SendVerificationEmailEvent` **only when** the command is
   inactive. **No `try/catch`.**
7. `Http\Admin\RequestHandler\CreateUserHandler` (+ factory): on a failed validation re-render
   `user::create-user-modal` with status **422** and `old` input; on success delegate to `UserListHandler` and
   add the `HX-Trigger: closeModal` header, using the same header mechanism `UserListHandler` uses.
8. Template `templates/default/user/create-user-modal.phtml`: modal fragment, form `hx-post` to the create URL,
   fields `firstName`, `lastName`, `email`, and a single `<select name="roleId">` of `assignableRoles`
   (preserve the selected value on re-render), an error list, a Cancel button. Escape every output
   (`escapeHtml` / `escapeHtmlAttr`). **No password field.**
9. The users list template: the "Add User" control opens the modal through the same mechanism the edit button
   uses for `update.modal`.
10. Tests for every new or changed class, per plan section 13.

Done: gates exit 0. Stop if: any task step conflicts with an existing class; `UserListHandler` cannot be reused
for the success response; a guard rule rejects a new class name or location.

### T5 review fixes (PR #76): same branch, same PR, new commit

Found by review of the PR diff. Fix both; touch only the files named.

**R1 - the success and failure outcomes never reach the handler.**
`ProcessCreateUserMiddleware` attaches only the `CommandResult::class` request attribute (which
`NotificationMiddleware` reads). `Http\Admin\RequestHandler\CreateUserHandler` reads the outcome from
`CreateUserState::$result`, and no `CreateUserState` is attached on the dispatch path. Result: a successful
create re-renders the form with 200 instead of returning the list with `HX-Trigger: closeModal`, and a failed
command returns 200 with an empty role list instead of 500.

1. `src/Http/Admin/Middleware/ProcessCreateUserMiddleware.php`: after dispatching the command, attach **both**
   `CommandResult::class => $result` **and** `CreateUserState::class => new CreateUserState(assignableRoles:
   $assignableRoles, old: $old, result: $result)` (keep the `CommandResult` attribute; `NotificationMiddleware`
   needs it).
2. Tests: add `test/unit/Http/Admin/CreateUserFlowTest.php` (`#[CoversClass]` on both the middleware and the
   handler): run the real middleware into the real handler (stub the bus, filter, logger, template and list
   handler). Cases: success returns the list handler's response with `HX-Push-Url` and
   `HX-Trigger: {"closeModal":null}`; a failed command returns 500 with the assignable roles preserved; a
   validation failure returns 422 with errors and old input.
3. Update `ProcessCreateUserMiddlewareTest` to assert both attributes on the success and failure paths.

**R2 - a 422 or 500 re-render lands in the wrong element.**
The form posts with `hx-target="main"`, so a failure response (the modal fragment) would be swapped into `main`.
Also, `CreateUserModalHandler` renders with `'layout' => false, 'body' => false`, but the POST handler's failure
render does not.

1. `src/Http/Admin/RequestHandler/CreateUserHandler.php`: on the 422 and 500 responses add the headers
   `HX-Retarget: #sharedModalDialog` and `HX-Reswap: innerHTML` (use the `Webware\Htmx\Response\Header` enum
   cases if they exist; if they do not, stop and report which cases exist), and pass `'layout' => false,
   'body' => false` to the template exactly as `CreateUserModalHandler` does. The success response is unchanged.
2. Update `CreateUserHandlerTest` for the headers and the template variables on 422 and 500.

Done: gates exit 0 with the existing coverage and MSI floors. Report in the standard format, plus one line saying
whether `Header` had `Retarget` and `Reswap` cases. Do not touch the held-back items.

## T6: App configuration (plan F3, F4)

**Repo:** `webinertia/webware` (app), branch `chore/verification-email-config`, after T2 is merged.
**Files (edit):** the app's `config/autoload/` global config (create `user.global.php` only if no suitable file
exists, and report which you chose).

Values, under the key `Webware\Core\UserInterface::class`: `base_url` `http://localhost:8080`,
`verification_email_subject` `Verify your Farmers IMS account`, `verification_token_ttl` `86400`. Mailer adapter
(`Webware\Mailer` adapter config key - read `webware-mailer/src/ConfigProvider.php` for the exact key):
`useSmtp` true, host `127.0.0.1`, port `1025`, no auth (Mailpit, `compose.yml`).

Done: gates exit 0. **Do not run the app.** Stop if: the exact mailer config key is not obvious.

*T6 was done by the owner's session: `webinertia/webware#23` (keys) and `#24` (`php-db/phpdb-validator` VCS entry
and install). Do not repeat it.*

## T7: Assignable roles on the edit flow (plan F2) - **DONE** (webinertia/webware-usermanager#80)

**Repo:** `webware-usermanager`, branch `fix/edit-user-assignable-roles`, from `1.0.x` after #76 is merged.

Found in the browser test and by reading `ProcessUpdateUserMiddleware`: the edit flow has **no role-assignment
check**. `UpdateUserDataFilter` only requires a `roleId` string, so any caller who can reach
`PATCH /admin/user/update/{id}` can set any role, including a role above their own (privilege escalation). The edit
modal also posts a hard-coded multi-select named `roleId[]`. **Rule (owner): a role the actor cannot assign is
not displayed and is rejected server-side, on create and on edit alike.**

**Files (create):** `src/Http/Admin/AssignableRolesProvider.php`,
`src/Http/Admin/Container/AssignableRolesProviderFactory.php`, `templates/default/user/partials/role-select.phtml`,
and one test per new class.
**Files (edit):** `src/Http/Admin/Middleware/ProcessCreateUserMiddleware.php` (+ factory),
`src/Http/Admin/Middleware/ProcessUpdateUserMiddleware.php` (+ factory),
`src/Http/Admin/RequestHandler/CreateUserModalHandler.php` (+ factory),
`src/Http/Admin/RequestHandler/UpdateUserModalHandler.php` (+ factory), `src/InputFilter/UpdateUserDataFilter.php`,
`src/InputFilter/CreateUserDataFilter.php`, `src/ConfigProvider.php`, `templates/default/user/update-user-modal.phtml`,
`templates/default/user/create-user-modal.phtml`, and the matching existing tests.

Steps:

1. `AssignableRolesProvider` (`final readonly`, constructor `MessageBusInterface $messageBus`), one public method
   `forRequest(ServerRequestInterface $request): array` returning `list<string>`. It is the code that is now
   duplicated in `ProcessCreateUserMiddleware` and `CreateUserModalHandler` (`actorRoleId()` and
   `assignableRoles()`), moved verbatim: read the actor from `Webware\Core\UserInterface::class`, no actor or no
   role returns `[]`, otherwise dispatch `FetchAssignableRolesQuery` and assert the payload with
   `Type\vec(Type\string())`. Register it with its factory in `ConfigProvider` dependencies. Replace the duplicated
   private methods in the two create classes with this provider (constructor argument `assignableRoles:`).
2. `ProcessUpdateUserMiddleware`: take the provider; build the filter data as
   `[...$body, 'id' => ..., 'assignableRoles' => $provider->forRequest($request)]` (server key after the spread).
3. `UpdateUserDataFilter`: add `AssignableRoleValidator` to `roleId`, exactly as `CreateUserDataFilter` has it.
4. `UpdateUserModalHandler`: take the provider; pass `'assignableRoles' => $provider->forRequest($request)` to the
   template, keep every existing variable.
5. `templates/default/user/partials/role-select.phtml`: one `<select id="{$id}" name="roleId" required>` with a
   `Select a role` empty option and one option per role in `$roles`, `selected` when it equals `$selected`, every
   value escaped. Variables: `$id`, `$roles`, `$selected`. Use it in both modals (render it with the same mechanism the
   templates already use for partials; if none exists, `include` it with an explicit variable array).
6. `update-user-modal.phtml`: delete the hard-coded options, the `multiple` attribute, the `roleId[]` name and the
   "Hold Ctrl/Cmd" text. `$selected` is the user's current role (first element of `$user->getRoles()`).
   `create-user-modal.phtml`: use the partial, same variables as today.
7. `CreateUserDataFilter`: set the `NoRecordExists` message so an existing email reads
   `An account with this email already exists.` (the validator option `messages` keyed by
   `NoRecordExists::ERROR_RECORD_FOUND`).
8. **Reject array input on every string field** (both filters). Measured 2026-10-02 with the real filters: a posted
   `roleId[]` (the edit modal's multi-select) passes `UpdateUserDataFilter` **valid and still an array**, because
   `StringTrim` returns non-strings unchanged and `NotEmpty` accepts a non-empty array; the value then reaches
   `new UpdateUserCommand(roleId: ...)`, whose parameter is `string`. On create, a posted `email[]` makes
   `NoRecordExists` throw `InvalidArgumentException: Value must be string, integer or null` (a 500), because the
   validator chain runs every validator unless told to break. In `UpdateUserDataFilter` and `CreateUserDataFilter`
   add, **first** in each field's `validators` list, `['name' => Validator\StringLength::class,
   'break_chain_on_failure' => true, 'options' => ['min' => 1, 'max' => N]]` with N from the column sizes:
   `firstName` 75, `lastName` 75, `email` 255, `roleId` 50. (`StringLength` in laminas-validator 3.18 rejects every
   non-string with "Invalid type given. String expected"; verified.) Do not touch the `id` or `active` fields.
9. Tests: provider (no actor, no role, a role set, payload shape); update middleware (a posted `roleId` outside the
   set is rejected and no command is dispatched; a posted `assignableRoles` is overwritten by the server value);
   both filters: an array for each of `firstName`, `lastName`, `email`, `roleId` is invalid and **no later validator
   runs** (assert no exception); a string longer than the column size is invalid; update filter; both modal
   handlers pass the list; templates need no test beyond the handler tests.

Done: gates exit 0 with the coverage and MSI floors. Stop if: a guard rule rejects `Http\Admin\AssignableRolesProvider`
(report which rule; do not work around it), or the user's current role is not in the assignable set and you think
the modal should behave differently - **do not decide that**, report it (the select then simply lacks the current
role and the form requires choosing one).

## T8: Validator false positive (owner's session, not the agent)

Browser test, 2026-10-02: `PhpDb\Validator\NoRecordExists` reports "found" for a nonexistent email, so no user can
be created. `AbstractDbValidator::query()` returns `$statement->execute()?->current()` and the PDO result returns
`false` (not `null`) for no rows; the validators test `!== null`. Verified by running the validator against the dev
database (existing email returns the row, nonexistent returns `false`). The fix is upstream in
`php-db/phpdb-validator` (normalise `false` to `null`); then `composer update php-db/phpdb-validator` in the app.

---

## Not for the agent (held for the design session)

- webware-tools guard changes (plan 5.1).
- CSRF wiring with `mezzio-csrf` and session settings.
- The password-set flag, `UserSchema` change and activation set-password step (done: webinertia/webware-usermanager#78);
  a real migration for installs that predate the column; password reset (plan 8.3).
- The member list and the update route's list and close behavior (F7).
- Email uniqueness on update (`NoRecordExists` with `exclude` the user's own id): an edit to another user's email
  currently reaches the database `UniqueKey`.
- Any change to `php-db/phpdb-validator` (T8).
- Opening the session-length, response-construction and CSRF-tracking issues.
