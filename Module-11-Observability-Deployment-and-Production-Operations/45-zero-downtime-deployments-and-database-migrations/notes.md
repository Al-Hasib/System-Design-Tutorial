# Study Notes: Zero-Downtime Deployments & Database Migrations

## Definitions

- **Rolling deployment:** Replacing old instances with new ones a few at a time, keeping the total ready-instance pool roughly constant throughout.
- **Blue-green deployment:** Running a complete second environment with the new version, then atomically switching all traffic to it once verified.
- **Canary deployment:** Releasing the new version to a small percentage of traffic first, gradually increasing it while watching metrics.
- **Expand-contract pattern:** A multi-step database migration discipline — add new schema structure first, migrate code/data to use it, then remove old structure only once safe.

## Deployment Strategies Compared

| Strategy | Infra cost | Rollback speed | Blast radius of a bad deploy | Complexity |
|---|---|---|---|---|
| Rolling | Low (reuses existing capacity) | Slower (gradual re-rollout) | Grows as rollout proceeds | Low — often the platform default |
| Blue-green | High (temporarily 2x environments) | Fast (switch router back) | All-or-nothing per cutover | Medium |
| Canary | Low-medium | Fast (halt/roll back ramp) | Small, contained initially | Medium-high (needs metric-watching/automation) |

## Why Old and New Versions Must Coexist

- Every zero-downtime strategy has a window where old and new versions serve traffic simultaneously (rolling: throughout the rollout; blue-green: briefly during cutover verification; canary: throughout the ramp).
- This means API/message/schema changes during a deploy must be backward-compatible for that window — additive, not destructive (echoes Protocol Buffers schema evolution from Module 8).

## The Expand-Contract Pattern (Database Migrations)

| Step | What happens | Deployed as |
|---|---|---|
| 1. Expand | Add new column/table/index; old code ignores it | Migration-only deploy, no app code change |
| 2. Migrate | New app code writes to both old and new structures; backfill historical data | App code deploy |
| 3. Contract | Once fully rolled out and backfilled, stop writing to old structure; reads move fully to new one | App code deploy |
| 4. Cleanup | Drop the now-unused old structure | Migration-only deploy, later |

- **Never** rename/remove a column and switch application code to it in a single atomic step — old-version pods still running mid-rollout would break immediately.

## Key Numbers / Facts

- Kubernetes Deployments default to a rolling update strategy, configurable via `maxSurge`/`maxUnavailable` parameters controlling how many extra/unavailable pods are allowed mid-rollout.
- Canary rollouts are often automated via progressive-delivery tools (e.g., Argo Rollouts, Flagger) that watch metrics and auto-advance or auto-rollback the traffic percentage.

## Summary

- All zero-downtime strategies keep the ready-instance pool above zero but require old and new versions to coexist for some window — demanding backward-compatible changes during that window.
- Rolling = cheap but slower rollback; blue-green = fast rollback but costs double infrastructure briefly; canary = smallest blast radius but slowest, most operationally involved rollout.
- Database schema changes need the same coexistence discipline, formalized as expand-contract: add, migrate, then remove — never all at once.
