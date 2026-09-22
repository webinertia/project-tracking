# Roadmap

The evolution plan for the webware component stack: from the current state (components mid-extraction from the IMS application, pre-1.0) through to rebuilding the IMS application as a host on a purpose-built skeleton.

**This file holds the plan. The tracker holds the state** — [issue #1](https://github.com/webinertia/project-tracking/issues/1).

## Phase 1

1. **Finish migrating all `webware*` components out of `tyrsson/inventory-management-system`** into their own repositories.
2. **Align all `webware*` components with `webware-tools`** — reusable workflow, shared mago config, central guard rules.
3. **Mago burndown for every component to green with NO baselines**, and `@mago-expect` suppressions kept to a bare minimum. Baselines are a temporary crutch, not a resting state.
4. **Refactor out the agent-generated code** introduced by Claude Sonnet, and/or align it with current webware implementation changes so it matches ecosystem patterns and usage.
5. **Add `webware-console` init commands where applicable**, extracting their seed data from the reference IMS application.
6. **Improve edge-case testing as gaps are identified.** The `webware-acl` seed → load → decision gap is the exemplar.
7. Once the webware components and the sparse IMS components are ready → Phase 2.

## Phase 2

### Composition doctrine (owner-stated, 2026-09-22)

> The skeleton is what will bind acl, usermanager, admin together. No component is required to
> require the other. The integration surface is the composer plugin installer and the console init
> commands.

What follows from it, and what governs the tracks below:

- **Composition belongs to the skeleton.** Nothing binds `acl`, `usermanager` and `admin` to each
  other, so no component gains a `require` or peer-dependency edge on another as the way that
  binding is expressed.
- **The installer and the init commands are the composition surfaces**: the installer decides what
  is installed and in what order; the init commands provision each package.
- **Config aggregation is the framework's binding mechanism, and it *is* an integration surface**
  (owner correction, 2026-09-22). Every component contributes default *and* runtime config through
  its `ConfigProvider`, and config-file aggregation merges all of it into the service manager's
  `config` service — routes, middleware, DI and the ACL declarations all compose that way. An
  earlier revision of this document claimed the opposite; that claim was invented to reconcile the
  doctrine against the plan, and it was wrong.

**Measured against the code today (2026-09-22), so the gap is explicit rather than assumed:**

| edge | how it exists now |
|---|---|
| `acl` → `admin` | `require: webware/webware-admin: 0.1.x-dev` |
| `usermanager` → `admin` | `require: webware/webware-admin: 0.1.x-dev` |
| `acl`, `usermanager` → `htmx` | `require: webware/webware-htmx: 0.1.x-dev` |

The code-level surface behind those edges is small — **three `admin` symbols and three `htmx`
symbols across both packages**:

- `Webware\Admin\Event\RegisterWidgetEvent`, `Webware\Admin\WidgetInterface`
- `Webware\Admin\Container\Configuration` (imported as `AdminConfiguration`) — its static
  `getAdminRouteSegment()` / `getAdminRouteNamePrefix()` helpers, so even a component's *route
  registration* currently consults admin's config
- `Webware\Htmx\Response\Header`, `Webware\Htmx\Http\Middleware\DisableBodyMiddleware`,
  `Webware\Htmx\Attribute`

Both packages also ship a duplicated `Admin/Dashboard/` trio — `Widget`, `RegisterWidgetListener`
and `Container/RegisterWidgetListenerFactory` — so the admin surface is repeated glue as well as an
edge.

### Before this leg — retire `Webware\Core\Configuration`

The `Configuration` / `ConfigurationInterface` static getters are already being phased out
(decided 2026-09-19: the interface **constants stay**, the getter methods and `Configuration`
classes go, replaced by mago `@type` shape aliases; order `webware-event` (pilot, applied) → `core` →
`acl` → `admin` → `usermanager`). That phase-out leaves a hole this leg depends on, because the
static getters are currently how a component's route segment and name prefix reach its route
provider.

**Owner direction (2026-09-22): find a better path than `Webware\Core\Configuration` before
starting, and prototype it.** The candidate under consideration is an **enum-backed provider for the
default required routes**, which double as ACL resources, expressed with `%s`-encoded strings so
segments can be substituted at runtime. Whether that is functional and flexible enough to meet the
requirements is **unresolved and needs a prototype** — do not treat it as decided.

**Measured requirements.** Two prefixes compose every route name, and the same string must be both
the registered route name and the ACL resource:

| kind | composition | example |
|---|---|---|
| public route | `route_name_prefix` | `user.manager.session.read` |
| admin route | `admin_base_name_prefix` + `module_admin_name_prefix` | `webware.admin.user.manager.create` |
| segment | `admin_base_segment` + `/` + `module_admin_segment` | `webware.admin/user.manager` |

Each component's `ConfigProvider` writes `route_segment` / `route_name_prefix` (defaults from
`ConfigurationInterface::*_VALUE`) into its **own** config section — `Webware\Core\UserInterface`,
`Webware\Admin\AdminInterface`, … — with the admin base coming from `webware-admin`'s section and the
module half from the component's own.

The mechanism has to yield, **from one source**: the registered route name, the ACL resource id, and
the parent resource id. That last one is not cosmetic — `RouteResource::getResourceId()` returns the
matched route name, so a route name and its resource id must be byte-identical or the check fails
closed.

### Phase 2 breakdown — the install contract leg

What actually blocks *starting* the skeleton is that **the stack has no install contract**: a
consumer cannot resolve the application components from a release, and the two provisioning
commands that exist are unreachable and unordered. The work below is ordered by that, not by
component size. Track A is the only critical path.

#### Track A — the two integration surfaces (critical path)

Track A is the entire integration story: the installer, and the init commands. Per the composition
doctrine above, none of it is expressed by one component requiring another.

| # | Work | Repo | Done when |
|---|---|---|---|
| A0 | **Prerequisite** — retire the `Webware\Core\Configuration` route getters via the enum-backed route/resource provider; prototype before committing to it (see *Before this leg*) | `webware-core`, `webware-acl`, `webware-admin`, `webware-usermanager` | One source yields the route name, ACL resource id and parent id, and no component reads a segment through a static getter |
| A1 | Reconcile the ACL config contract with the decision engine (see *Decisions* below) | `webware-acl` | One source of truth; the declared policy reaches the tables the engine reads |
| A2 | Seed the framework's own gateway policy | `webware-usermanager`, `webware-acl` | A fresh install yields Guest→allow / Member→deny on the `user.manager.*` gateways, with zero IMS store roles |
| A3 | Composer-plugin installer: discovers per-package provisioners, orders them from the dependency graph | new | No component needs a peer dependency to be installed in the right order |
| A4 | Make the console reachable **from the skeleton**, not from the components. The commands already register by config, so the `require-dev` arrangement is correct and must *not* be promoted to a `require` — that would be exactly the component→component edge the doctrine forbids | skeleton / installer | `*:init-db` is reachable in a real consumer install, with no component requiring `webware-console` |
| A5 | Normalise the provisioning command surface (the `DB`/`Db` casing diverges in both class name and file name) | `webware-acl` | One naming convention across the fleet |
| A6 | Declare and enforce install order; add the missing `acl_rule.roleId → acl_role.roleId` guard or decide against it | `webware-acl` | A wrong order fails at seed time, not at first authorization |
| A7 | Keep the composition in the skeleton: the skeleton supplies `acl`, so `usermanager` never pulls it in (today `acl` is a `suggest` from `admin` and a require of nothing) | skeleton | The `acl` / `usermanager` / `admin` binding exists only in the skeleton |
| A8 | Log table provisioning command (parity with `acl:init-db` / `user:init-db`) | `webware-log` | `webware-log` is a provisioner too |
| A9 | Retire the component→`admin`/`htmx` edges so the doctrine holds in code (see the pinned edge table above) | `webware-acl`, `webware-usermanager`, `webware-admin` | Neither package `require`s `admin` (or `htmx`), and the duplicated dashboard trio exists once |

A9's three sub-questions, each smaller than the row implies:

1. **Admin route config** — components read admin's config statically for their own route segment and
   name prefix. Inverting this (each component reads its *own* config keys, as it already does
   elsewhere) removes the import outright.
2. **The dashboard widget contract** — `WidgetInterface` and `RegisterWidgetEvent` live in `admin`.
   Either the contract moves to the neutral foundation layer, or the skeleton owns widget
   registration and the components stop naming an admin symbol.
3. **`htmx` needs a classification** — `Webware\Htmx` is a response/attribute utility, not an
   application layer. If it is foundation (like `core`, which every component may require) the edge
   is legitimate and stays; if it is a component, the edge has to go. This is a one-line doctrine
   decision, not a refactor.

One mechanism note that A1, A2 and A8 depend on: a provisioning command reads the config
contributed by the *other* packages (`AclInterface::class`, the command maps). That is
config-provider aggregation — the framework's own self-registration — which the doctrine explicitly
leaves alone. What is out of bounds is one component binding to another **at runtime**; reading
merged config at provision time is not that.

#### Track B — make the stack resolvable

Gates handing the skeleton to anyone else; not required to begin local work.

| # | Work | Done when |
|---|---|---|
| B1 | Land the outstanding pre-release work on each default branch | Nothing pending is excluded from the next tag |
| B2 | Replace `N.N.x-dev` constraints in every `require` block with versions | No branch alias in a `require` |
| B3 | Publish the packages that are not on Packagist at all | `composer require` resolves from a tag |
| B4 | Cut the component tags (owner action) | A bare Mezzio app installs the stack with no VCS repository entry |

#### Track C — the skeleton repository

| # | Work | Note |
|---|---|---|
| C1 | Scaffold from the current tooling artifacts plus the app layout — do not revive a stale toolchain | See *Decisions* |
| C2 | Repo identity: org, name, package name, default branch `N.N.x` | The organization's required CI applies unless the repo is explicitly excluded, so `webware-ci.json` must be in the first commit or the first PR is blocked by a red required workflow |
| C3 | First commit set: canonical tooling artifacts, `public/`, `config/`, `bin/`, `webware-ci.json`, container config, empty baselines | Four mago gates plus required CI green on the first PR |
| C4 | Global pipeline order: the container alias for the user contract, and ACL middleware after identity resolution | Pipeline ordering is currently unverified against login |
| C5 | Compose the application layer: the skeleton supplies `acl`, `usermanager`, `admin` and `console`, and is the only place that binding exists | The doctrine's positive obligation — see *Composition doctrine* above |

#### Track D — Phase 1 burndown (parallel, not blocking)

Baseline burndown to zero and the PSR-14 consolidation are Phase 1 exit criteria and are tracked
in [issue #1](https://github.com/webinertia/project-tracking/issues/1). They run alongside this
leg and do not gate it.

### Decisions

**Owner-stated, 2026-09-22 — not an agent proposal:** the composition doctrine above. The skeleton
binds `acl`, `usermanager` and `admin`; no component requires another; the integration surface is
the composer plugin installer plus the console init commands.

Taken by the agent while the owner was unavailable, **proposed pending owner ratification** —
revise freely:

1. **ACL source of truth: the database stays the runtime engine; the config contract becomes the
   seed input.** `AclSchema::ruleSeeds()` merges the `AclInterface::class` config contributed by
   every package and materialises it into `acl_role` / `acl_rule`. Rationale: it is the smallest
   reversal, it keeps the documented integration contract meaningful, and it gives the installer a
   natural per-package provisioning unit without any component depending on another.
   **Retraction (2026-09-22).** An earlier revision argued this was also the only reading consistent
   with the doctrine, because option (A) would add a third integration surface. That rested on the
   incorrect claim that config aggregation is not an integration surface. It is, so the doctrine does
   not exclude option (A), and the choice in
   [webware-acl#59](https://github.com/webinertia/webware-acl/issues/59) is open on its own merits.
   The route/resource finding below now argues *for* runtime derivation, which is option (A)'s shape.
2. **The installer is a new framework-layer package** (a composer plugin), not a feature of an
   individual component.
3. **The skeleton is scaffolded from the current tooling artifacts plus the application layout**
   of the existing custom skeleton, rather than aligning that skeleton's older toolchain first.

### Findings to action outside this leg

- **The ACL integration contract is broken against the implementation.** The decision engine takes
  no config: it builds from the message bus and sources resources *and* rules from the rule table,
  and direct resource registration throws by design. The documented `roles` / `resources` /
  `allow` / `deny` array therefore has readers for `resources` only (a display path). Consequence:
  the gateway policy that `webware-usermanager` declares in `getAclConfig()` is inert.
  Tracked in [webware-acl#59](https://github.com/webinertia/webware-acl/issues/59).
- **`webinertia/.github` → `.github/rulesets/org-required-ci.json` no longer describes the deployed
  ruleset.** The file scopes coverage with a `webware-*` name pattern, but the live ruleset also
  covers repositories that pattern cannot match. Re-applying the file as written would silently
  drop required CI from them.
  Tracked in [webinertia/.github#20](https://github.com/webinertia/.github/issues/20).
- **`webware-acl`'s `ruleSeeds()` composes resource ids from the module prefix only, so all 11 seeded
  rows are inert.** It uses `Configuration::ADMIN_ROUTE_NAME_PREFIX_VALUE` (`acl.manager.`) where the
  registered route name is `webware.admin.acl.manager.role.read`. `Acl::load()` then registers that
  route resource with **no parent** — the walk up the dot path never finds a seeded ancestor, because
  the seeded resources are one level short — so no seeded rule can ever reach it. Masked in practice
  because `load()` also adds an allow-all for `Developer`, the only role the seed covers. Proof it is
  an extraction error rather than a naming choice: the reference IMS seed carries the same 11 rows
  with the same 10 suffixes under the full composed name and parent.
  Tracked in [webware-acl#60](https://github.com/webinertia/webware-acl/issues/60).

## Constraints

- **`tyrsson/inventory-management-system` is reference-only — no changes.** It is the source for seed data and for old component code: read it, do not edit, commit, or change its checked-out branch.
- Releases are immutable and are cut by the repository owner. Pending PRs must be merged and pulled into a branch **before** its PR goes up, otherwise the release excludes them and a second release is required.
