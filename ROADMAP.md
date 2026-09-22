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

- Build a custom base skeleton using `webware*` components, via `tyrsson/mezzio-bleeding-edge`.
- Build a composer plugin for installing optional packages for M2.
- Start rebuilding the IMS application as a host from M2.

### Phase 2 breakdown — the install contract leg

What actually blocks *starting* the skeleton is that **the stack has no install contract**: a
consumer cannot resolve the application components from a release, and the two provisioning
commands that exist are unreachable and unordered. The work below is ordered by that, not by
component size. Track A is the only critical path.

#### Track A — provisioning contract (critical path)

| # | Work | Repo | Done when |
|---|---|---|---|
| A1 | Reconcile the ACL config contract with the decision engine (see *Decisions* below) | `webware-acl` | One source of truth; the declared policy reaches the tables the engine reads |
| A2 | Seed the framework's own gateway policy | `webware-usermanager`, `webware-acl` | A fresh install yields Guest→allow / Member→deny on the `user.manager.*` gateways, with zero IMS store roles |
| A3 | Composer-plugin installer: discovers per-package provisioners, orders them from the dependency graph | new | No component needs a peer dependency to be installed in the right order |
| A4 | Make the console reachable — `webware-console` is `require-dev` in `acl` and `usermanager` while both register commands into it | `webware-acl`, `webware-usermanager` | `*:init-db` exists in a real consumer install |
| A5 | Normalise the provisioning command surface (the `DB`/`Db` casing diverges in both class name and file name) | `webware-acl` | One naming convention across the fleet |
| A6 | Declare and enforce install order; add the missing `acl_rule.roleId → acl_role.roleId` guard or decide against it | `webware-acl` | A wrong order fails at seed time, not at first authorization |
| A7 | Decide which layer owns the `usermanager → acl` edge (acl is a `suggest` and a require of nothing) | `webware-acl` / skeleton | Components stay decoupled; the edge lives in the framework layer |
| A8 | Log table provisioning command (parity with `acl:init-db` / `user:init-db`) | `webware-log` | `webware-log` is a provisioner too |

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

#### Track D — Phase 1 burndown (parallel, not blocking)

Baseline burndown to zero and the PSR-14 consolidation are Phase 1 exit criteria and are tracked
in [issue #1](https://github.com/webinertia/project-tracking/issues/1). They run alongside this
leg and do not gate it.

### Decisions

Taken by the agent while the owner was unavailable, **proposed pending owner ratification** —
revise freely:

1. **ACL source of truth: the database stays the runtime engine; the config contract becomes the
   seed input.** `AclSchema::ruleSeeds()` merges the `AclInterface::class` config contributed by
   every package and materialises it into `acl_role` / `acl_rule`. Rationale: it is the smallest
   reversal, it keeps the documented integration contract meaningful, and it gives the installer a
   natural per-package provisioning unit without any component depending on another.
2. **The installer is a new framework-layer package** (a composer plugin), not a feature of an
   individual component.
3. **The skeleton is scaffolded from the current tooling artifacts plus the application layout**
   of the existing custom skeleton, rather than aligning that skeleton's older toolchain first.

### Findings to action outside this leg

- **The ACL integration contract is broken against the implementation.** The decision engine takes
  no config: it builds from the message bus and sources resources *and* rules from the rule table,
  and direct resource registration throws by design. The documented `roles` / `resources` /
  `allow` / `deny` array therefore has readers for `resources` only (a display path). Consequence:
  the gateway policy that `webware-usermanager` declares in `getAclConfig()` is inert. Tracked
  against `webware-acl`.
- **`webinertia/.github` → `.github/rulesets/org-required-ci.json` no longer describes the deployed
  ruleset.** The file scopes coverage with a `webware-*` name pattern, but the live ruleset also
  covers repositories that pattern cannot match. Re-applying the file as written would silently
  drop required CI from them. Tracked against `webinertia/.github`.

## Constraints

- **`tyrsson/inventory-management-system` is reference-only — no changes.** It is the source for seed data and for old component code: read it, do not edit, commit, or change its checked-out branch.
- Releases are immutable and are cut by the repository owner. Pending PRs must be merged and pulled into a branch **before** its PR goes up, otherwise the release excludes them and a second release is required.
