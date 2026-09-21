# Project Tracking

Central status tracking for org projects. Currently: the **Webware component stack** (18 repos).

## The tracker

**[Component status tracker #1](https://github.com/webinertia/project-tracking/issues/1)** — the living completion matrix.

It tracks four workstreams per component:

| axis | meaning |
|---|---|
| **tools** | aligned to the `webware/webware-tools` v1 preset |
| **refactor** | the component's own refactor workstream |
| **lint** | `mago lint` burndown |
| **analysis** | `mago analyze` burndown |

Per-component checkpoint issues attach to it as sub-issues, and GitHub derives progress from them.

## Rules

- **One source of truth.** The matrix lives in issue #1. Do not mirror it into files here — a second copy drifts, and the issue's edit history is itself the burndown record.
- **Measured, not estimated.** Burndown numbers are the `count`-weighted totals summed from each component's `lint-baseline.toml` / `analysis-baseline.toml` on its default branch.
- **Baselined is not fixed.** A baselined finding is suppressed. Burndown means driving it to zero and deleting the baseline file.
- **No milestones ahead of time.** A version milestone is created when a release is actually being cut, never planned in advance.

## Scope

This repo holds issues only — no code, no Composer package, nothing to publish. Component code lives in its own repo; org-wide templates and config live in [`webinertia/.github`](https://github.com/webinertia/.github). This repo is private, which is why tracking does not live there.
