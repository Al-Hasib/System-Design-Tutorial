# Zero-Downtime Deployments & Database Migrations

**Difficulty:** Intermediate/Advanced
**Estimated length:** 14-18 min
**Prerequisites:** [44 - Containers and Orchestration: Docker and Kubernetes Fundamentals](../44-containers-and-orchestration-docker-and-kubernetes-fundamentals/README.md), [07 - Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md)

## Learning Objectives

- Explain why shipping a new version of a service without downtime is harder than "just replacing the old code."
- Compare rolling, blue-green, and canary deployment strategies and the trade-offs between them.
- Explain why old and new versions of a service must coexist briefly during any zero-downtime deployment, and what that implies for API compatibility.
- Describe the expand-contract pattern for changing a database schema without breaking a running application.
- Design a safe rollout plan for a change that touches both application code and the database schema together.

## Script

### Hook / Intro

We spent the last video on how Kubernetes schedules and heals containers. Today's question is narrower and, in a lot of real incidents, more dangerous: how do you actually replace a running version of your service with a new one, serving live production traffic the entire time, without a single dropped request — and how do you do the same thing to your database's schema, which can't just be "restarted" the way a container can? Get this wrong and you get the classic 2am incident: a deploy that looked fine in staging takes down checkout for eight minutes in production.

### Why "Just Replace It" Doesn't Work

If you simply stopped all instances of the old version and started the new version, there's an inevitable gap — however small — where no instance is available to serve requests, and every in-flight request at that exact moment fails. At any real scale, "however small" is still enough requests failing to be a real, visible incident. Zero-downtime deployment strategies exist specifically to eliminate this gap by never having a moment where the total count of *healthy, ready* instances drops to zero — building directly on the readiness probes and Service abstraction from last video.

### Rolling Deployments

A **rolling deployment** — the default in Kubernetes Deployments — replaces old instances with new ones a few at a time: start one new-version pod, wait for its readiness probe to pass, then terminate one old-version pod, and repeat until all instances are on the new version. At every point in this process, the total pool of ready instances stays roughly constant, so the Service keeps routing traffic successfully throughout. The real subtlety, and the thing that catches teams unprepared: for the whole duration of the rollout, *both the old and new versions are serving live traffic simultaneously*. If the new version changes an API response shape, a database column's meaning, or a message format on a queue in a way the old version doesn't understand (or vice versa), you get real, live errors during the rollout window — which is exactly why backward-compatible, additive changes (new optional fields, not renamed or removed ones) are the standard discipline for any service that gets rolling-deployed, echoing the schema-evolution principles from Module 8's Protocol Buffers video.

### Blue-Green and Canary Deployments

A **blue-green deployment** takes a different approach: run a complete, fully-scaled second environment ("green") alongside the current one ("blue"), deploy the new version entirely into green while blue keeps serving all live traffic, and once green passes health checks, switch the router/load balancer to send all traffic to green in one atomic cut-over. If something's wrong, rolling back is just switching the router back to blue, which is still fully running and untouched — a fast, clean rollback compared to a rolling deployment, which requires rolling back gradually or re-deploying the old version pod by pod. The cost is running two full production-scale environments simultaneously, at least briefly, which roughly doubles infrastructure cost for the duration of the switch.

A **canary deployment** goes further in the cautious direction: release the new version to a small fraction of traffic first — say, 5% — and closely watch error rates, latency, and other metrics (recall Module 11's observability tools) before gradually increasing that percentage toward 100%. This limits the blast radius of a bad deploy to a small slice of users rather than everyone, at the cost of a slower, more gradual rollout and the operational overhead of actually watching and deciding when to proceed at each stage — often automated today via progressive-delivery tooling that watches metrics and advances (or automatically rolls back) the canary percentage on its own.

### Database Migrations: The Expand-Contract Pattern

Application code can be blue-green'd or canaried relatively cleanly because you can run two versions side by side. A database schema is trickier — there's normally only one database, shared by both the old and new application versions during any rollout — so a schema change has to be safe for *both* versions to read and write against simultaneously, for at least the duration of the deployment. The standard discipline for this is the **expand-contract pattern**, done in separate, sequential deploys rather than one atomic change:

**Expand**: add the new schema element (a new column, table, or index) without removing or repurposing anything old — the old application code simply ignores the new column it doesn't know about, exactly like the additive schema-evolution principle from Protocol Buffers. Deploy this migration alone, with zero application code changes yet. **Migrate**: deploy a new version of the application that writes to *both* the old and new columns/tables (and reads from whichever is authoritative for now), backfilling historical data into the new structure as needed. **Contract**: once you've confirmed every instance is running the version that uses the new schema, and historical data is fully backfilled, deploy a final change that stops writing to the old column/table, and only then, in a later cleanup step, actually drop it. Trying to skip straight to "rename this column and deploy the new code that expects the new name" in one shot is exactly the mistake that breaks a rolling deployment — the old-version pods still running during the rollout window would immediately start failing against a column that no longer exists under the old name.

### Real-World Example

Imagine renaming a `user.name` column to `user.full_name` while a rolling deployment is standard practice for this service. Attempting this as one change — migrate the column and deploy new code that only reads `full_name` — breaks immediately, because old-version pods still running mid-rollout keep querying `name`, which no longer exists. Using expand-contract instead: first deploy adds a new `full_name` column (expand) alongside the untouched `name` column; a second deploy ships application code that writes to both columns on every update and backfills existing rows' `full_name` from `name`; once fully rolled out and backfilled, a third deploy switches all reads to `full_name` and stops writing to `name`; only after that's been running safely for a while does a final cleanup migration actually drop the now-unused `name` column. Four separate, individually-safe steps instead of one deploy that would have caused a real incident.

### Recap

Zero-downtime deployment strategies exist to keep the pool of healthy, ready instances from ever hitting zero, but all of them (rolling, blue-green, canary) require the old and new application versions to coexist and correctly serve traffic during at least part of the rollout — which means changes need to be backward-compatible for that window. Rolling deployments replace instances gradually with minimal extra infrastructure cost; blue-green swaps an entire environment atomically, trading infrastructure cost for a fast, clean rollback; canary limits a bad deploy's blast radius by ramping traffic gradually while watching metrics. Database schema changes need the same "old and new must coexist" discipline, formalized as the expand-contract pattern: add new structure first, migrate application code and data next, and only remove the old structure once nothing depends on it anymore.

### What's Next

We've now covered deploying changes safely. Next video asks a related but distinct question: how do you actually *prove*, deliberately and proactively, that your system survives real load and real failure — rather than finding out for the first time during an actual incident?

## Key Takeaways

- Zero-downtime deployment strategies work by never letting the pool of healthy, ready instances drop to zero, building on readiness probes and load-balanced service discovery.
- Rolling deployments replace instances gradually and cheaply, but mean old and new versions serve traffic simultaneously during the rollout — requiring backward-compatible changes.
- Blue-green deployments run a full second environment and cut over atomically, giving a fast, clean rollback at the cost of temporarily doubled infrastructure.
- Canary deployments ramp new-version traffic gradually while watching metrics, limiting a bad deploy's blast radius at the cost of a slower rollout.
- The expand-contract pattern makes database schema changes safe under any of these strategies: add new structure first (expand), migrate code/data to use it, then remove the old structure only once nothing depends on it (contract) — never rename/remove in one atomic step.
