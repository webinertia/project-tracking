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

## Constraints

- **`tyrsson/inventory-management-system` is reference-only — no changes.** It is the source for seed data and for old component code: read it, do not edit, commit, or change its checked-out branch.
- Releases are immutable and are cut by the repository owner. Pending PRs must be merged and pulled into a branch **before** its PR goes up, otherwise the release excludes them and a second release is required.
