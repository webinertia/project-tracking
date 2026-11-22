# WEBWARE COORDINATION HANDOFF: written 2026-09-30 before user's firmware restart
User restarted their machine; NOTHING was in flight. User said: "Do not start this work yet" - wait for their go.
I am the COORDINATING session for all webinertia/webware-* repos (symlinked into webware/webware via path repos).
Notify the user LOUDLY when a release/tag is needed - they cut ALL tags. Never tag, never `gh release`.

## NEXT TASK (user's words): "handle the missing log table now and then test again"
- webware-log `Handler/PhpDbHandler` writes to table `log` (ConfigProvider default `'table' => 'log'`, `PhpDbHandlerFactory` reads `$config['table']`).
- webware-log has NO `src/Console`, NO DDL, NO init command. Nothing can create the table.
- Symptom: a failed-login POST to /user/login -> 500 `PhpDb\...InvalidQueryException 42S02 1146 Table 'webware_app.log' doesn't exist`
  (Monolog handler writes during the request). GET pages are 200.
- Plan (my proposal, not yet approved/started): in webware-log add a table builder (mirror acl: `src/Console/Schema/*Schema.php` with
  PhpDb Ddl CreateTable, see webware-acl/src/Console/Schema/AclSchema.php) + `log:init-db` command (final class *Command, #[AsCommand],
  in `Webware\Log\Console\`, factory in `Console\Container\`), register under `Webware\Console\ConsoleInterface::class => ['commands'=>[...]]`,
  table name via `PhpDb\SchemaFactory` (`Schema` enum implementing `PhpDb\SchemaInterface`, like acl's Repository\Schema). Idempotent (ifNotExists).
  webware-log default branch is `2.0.x` (NOT 1.0.x); check `git -C webware-log branch -a` first. Needs webware-console + symfony/console require.
  Check the log table columns from PhpDbHandler's INSERT (read it first) - do not guess columns.
- After that: run `php vendor/bin/webware log:init-db` in webware/webware, re-test failed login POST (expect the "Invalid email or password." toast),
  then session table.
- USER NOTE: for sessions "We may just provide our own session handler via a webware component" (instead of / besides php-db/phpdb-mezzio-session).
  Session table is REQUIRED (user 2026-09-30: "those tables are required"). Currently native file sessions (Mezzio\Session\Ext). Nothing requires
  php-db/phpdb-mezzio-session anywhere (verified). Its repo: ~/github.com/php-db/phpdb-mezzio-session (not under webinertia/). DDL belongs in
  a component init-db; plan item 3b.

## OPEN PRs (nothing merged-pending-CI except these)
- webinertia/webware-usermanager #69 `feat!: ship the user templates as the default theme` (branch feat/default-theme-templates, commit fe0ea0e) - 12/12 checks
  PASS, OPEN, NOT MERGED. User wants to see the pages in the browser first; merge only when told. Merge commit preferred (single commit -> state reason).
- webinertia/webware-htmx #22 `fix: name the body and layout with one template address each` (branch prototype/theme-config, commit 58fa4ff) - user marked READY
  (not draft), CLEAN, 12 checks pass, **NOT MERGED (user confirmed 2026-09-30 "never merged")**. It is the FR-012 prerequisite of webware-theme. Do not merge unless asked.
  The webware-htmx clone is currently checked out on prototype/theme-config (user's checkout, 1 dirty file) - do not switch it.
- No PR for webware/webware (standing rule 2026-09-28: none until the app loads in a browser - it now does; ASK before any PR).

## MERGED TODAY (all default branches parked clean except where noted)
- webware-tools #41 (PhpDb\** added to its own guard perimeter) -> tag 1.0.0-beta.6 (user)
- webware-phpdb #3 (schema abstraction, namespace stays PhpDb\ - may move into php-db upstream; do NOT rename) -> tag 1.0.0-alpha.1 (Packagist registered)
- webware-core #47 (RuleSeed contract), #48 (schema moved out) -> tag 1.0.0-alpha.8
- webware-acl #68 (PhpDb schema), #69 (acl:seed, Acl\RuleSeeds, SeedRunner; squash)
- webware-usermanager #67 (PhpDb schema + naming standard, COMPONENT_NAME 'user'), #68 (Acl\RuleSeeds provider, allow/deny maps removed)
- webware-theme + webware-phpdb Packagist-registered; theme has NO tag, still prototype (branch prototype/resolver, 0/39 tasks).

## STATE OF webware/webware (the host app): branch feat/app-acl-seeds, UNCOMMITTED (26 dirty files, nothing committed, no PR)
- Phase 4 done in tree: `App\Container\Configuration` (COMPONENT_NAME 'app'), route `app.home`, `App\Acl\RuleSeeds` (Guest Allow app.home),
  published under `Webware\Core\AclInterface::class => rule_seed_providers`. Tests updated/added (unit OK).
- composer.json core floor ^1.0.0-alpha.8; lock adds webware/webware-phpdb; config/config.php has `\PhpDb\WebwareProvider::class` (injected).
- compose.yml: app mysql DB/user/password = `webware_app` (NOT `webware`: component test suites target `webware` and phpdb's fixture loader
  DROPs it - that is how the app's DB was lost 2026-09-30). config/autoload/mysql.local.php (untracked) matches. DB + user created in running container.
- config/development.config.php and .dist both: `Tracy\Debugger::class => ['enable' => Debugger::Development]` (Development === false in Tracy 2.12!).
  Without it Traccio's TracyDebuggerMiddleware never calls Debugger::enable -> empty 500s.
- config/pipeline.php: `MessageMiddleware::class` piped right after SessionMiddleware (messenger never attached otherwise).
- src/App/templates/default/layout/default.phtml: + bootstrap-icons css + `htmx.onLoad` script showing `.toast` (bootstrap.Toast).
- src/App/templates/ims/user/{login,registration,resend-verification,verify-email}.phtml = byte-identical copies of the IMS pages; theme.global.php `ims` map has
  `user::login|registration|resend-verification|verify-email`. active theme = `default` (ims dormant). ims/body/default.phtml has imsMessenger() UNCOMMENTED
  (== IMS original except one formatter blank line).
- DB webware_app tables now: acl_role, acl_rule (21 rules from 3 providers: acl 11, usermanager 9, app 1), user. MISSING: log, session (both required).
- Run commands with `php vendor/bin/webware <cmd>` (list: acl:init-db, acl:seed, user:init-db, dev:mode, menu). acl:init-db WITHOUT --drop is safe.
- Dev mode is ON (no config cache). Dev server on :8080 is the USER's (`composer serve`), mysql container webware-mysql-1 on 127.0.0.1:3306 (user's, healthy).
  webware-acl-mysql-1 is stopped. Never start a server of my own on 8080.
- Reproduce a 500: Tracy bluescreen is in the response body; or scratch script /tmp/repro-login.php (outside repos; may be gone after reboot).

## RULES THE USER HAS GIVEN (follow exactly)
- tyrsson/inventory-management-system is READ-ONLY: no edits, no branch switches, nothing. Original IMS code must never be lost.
- No IMS helper/class/copy in webware components; IMS code lives in the `ims` theme / IMS components. "Default" theme = plain Bootstrap + htmx
  (hx-boost on <body>, hx-target main), palette/logo TBD (webware org GitHub logo) - work through later.
- MAKE NO CHANGES TO FORM FIELD NAMES when rewriting templates (verified via grep signature diff vs git originals).
- Never delete a file without explicit approval (renames/moves count). git mv of templates under templates/default/<ns>/ WAS approved. I deleted
  only stubs/dups the user approved.
- Every change through a PR; never push to default/version branches; sign-off commits (`git commit -s`), identity Joey Smith <jsmith@webinertia.net>.
- Merge commits by default; state the reason before squashing (acl #69 was squashed because commit 1 was red alone).
- Do not run anything that boots app code while user's debug session is attached: check `ss -ltnp | grep -E ':(9000|9003)'` first.
- Do not probe Packagist. Do not run composer install/update in webware/webware unless asked (user approved two targeted updates today).
- Reply style: findings first, no closing recap/summary, no option menus; say "measured" vs "inferred" honestly; ask ONE narrow question if blocked.

## GOTCHAS LEARNED
- bash history expansion: `feat!:` inside double quotes aborts the command line -> `set +H` and single-quoted messages.
- create_file/edit tools write ASYNCHRONOUSLY to disk in this workspace: `mago fmt` right after creating a file can leave a 1-byte file. `sleep 3-5` and
  check `wc -c` before formatting. A new class also may need `composer dump-autoload`. multi_replace_string_in_file input needs `"key": "value"` (I twice typed `>`).
- New test files + mago: kan-defect lint fires on test classes with many loops -> use data providers / helper class.
- acl/usermanager integration tests run `acl:init-db --drop` against 127.0.0.1:3306 - only run against a throwaway DB (acl compose mysql), never the app's.
- Shared DB name `webware` on 3306 across all component test suites; phpdb MysqlFixtureLoader CREATE/DROP DATABASE `webware`.
- core tag alpha.8 removed Webware\Core\Schema*; route accessors now take `$adminName` string (AdminConfiguration::getAdminName()).
- mago `.phtml` files get reformatted by editor format-on-save sometimes (blank-line diffs).

## THEME (webware-theme) FACTS: measured 2026-09-30
- Code is MAP-based: ThemeResolver looks up `theme.themes[<active>][<address>]` then `theme.themes['default'][<address>]` else false. AggregateResolverFactory order:
  ThemeResolver(100) -> TemplateMapResolver(50) -> NamespacedPathStackResolver(1). `theme.active` defaults 'default'. Tests: unit 7, integration 5 pass.
- Spec docs (specs/001-theme-resolution/) still describe `theme.roots` + PSR-4 derivation (plan.md, data-model.md, quickstart.md, tasks T010/T014/T024) - STALE vs code.
  User will supply corrections (hand-typed) or ask me to list passages. Do not build on `roots`.
- Convention agreed: components ship `templates/default/<namespace>/` and publish path under their namespace; host app overrides via theme map. No webware-theme
  dependency in components. usermanager DONE (PR #69). TODO next rounds: usermanager admin-widget.phtml + list-users.phtml (5 `ims-*` classes each, admin views -
  keep, strip ims classes, keep function), webware-acl templates (8, `templates/acl`), webware-admin (1, `templates/admin`), htmx body (PR #22), app.
- `imsMessenger` is defined ONLY in the IMS repo (App\View\Helper\ImsMessenger + Middleware\ImsMessengerMiddleware). Neutral replacement: webware-message `messenger`
  view helper (+ MessageMiddleware). SystemMessenger helper returns toast markup without container -> templates wrap in `.toast-container`.
- webware-htmx shell (154 lines) preserved: htmx git history (858c5a0), IMS repo original, and app ims copy.

## OTHER PENDING / OPEN ITEMS
- webware-acl `src/RuleType.php` + `test/unit/RuleTypeTest.php` now unused (core RuleType used) - deletion needs user approval.
- plan doc `webware-usermanager/plan/acl-rule-seeding-and-naming-standard.md` is UNTRACKED and STALE (core #47, acl #67-69, usermanager #67/#68 merged; 3d dropped because RuleSeeder lives in acl).
- webware-console clone has 2 dirty files (dev:mode help text) from before - not mine.
- webware-phpdb composer.json description is misleading ("shadows ... replace classes") - user aware, unchanged.
- webware-phpdb had no CI until user set org property `ci-target=true` (it is now set). First CI run there not yet observed.
- Plan phases remaining: 3b session (see above), Phase 5 log table (NEXT), Phase 6 installer readiness, admin/acl template migration, palette/logo for default theme.
- Stashes: none left in usermanager (dropped with approval). webware-phpdb has stash@{0} (byte-identical copy of untracked schema files) - safe to drop with approval.
