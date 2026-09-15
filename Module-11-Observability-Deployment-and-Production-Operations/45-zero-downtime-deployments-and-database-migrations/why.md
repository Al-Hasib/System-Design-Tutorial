# Why This Topic Matters: Zero-Downtime Deployments & Database Migrations

> **In one sentence:** Every deploy is a moment when two versions of your code exist at once, and if your schema change assumes only one version is running, you have scheduled an outage.

## The World Before This Idea

**The maintenance window.** Deploys happen Sunday at 2 a.m. behind a "we'll be back soon" page. Because deploys are painful, they are rare; because they are rare, each one bundles a month of changes; because each one is large, failures are frequent and hard to diagnose. The cost of deploying makes deploying worse, which makes it more costly. Teams stuck in this loop ship slowly and are afraid of their own release process.

**The migration that took the site down.** A developer renames `user_name` to `username` in one migration. It runs. Every instance of the old code — still serving traffic during the rolling update — instantly starts throwing errors because the column it queries no longer exists. The deploy is rolled back, but the migration is not reversible, and now the *new* code cannot run either. The site is down in both directions.

## The Problems It Solves

### 1. Downtime as a cost of shipping
**What you see:** Scheduled outages, releases confined to low-traffic hours, and a release process that requires several people awake at night.

**Why it happens:** Replacing all instances at once means a window with nothing serving.

**How rolling deploys solve it:** Replace instances a few at a time behind a load balancer. Take one out of rotation, drain its in-flight requests, update it, health-check it back in, repeat. Capacity dips slightly; availability does not. This is what makes deploying a normal daytime activity rather than an event.

### 2. Bad releases reaching everyone at once
**What you see:** A subtle bug — a memory leak, a slow query, a broken edge case — that testing missed, now affecting 100% of users.

**Why it happens:** No production exposure until full exposure.

**How canary and blue-green solve it:** A canary sends a small percentage of traffic to the new version and compares error rates and latency against the old one. A problem is caught at 1% of users instead of 100%. Blue-green runs the full new environment alongside the old and switches traffic at once — giving a near-instant rollback by switching back, at the cost of running double capacity briefly. Both buy you the same thing: a bounded blast radius and a fast way out.

### 3. Schema changes that assume one version of the code
**What you see:** The rename scenario above. Or adding a `NOT NULL` column with no default, so old code's inserts fail. Or dropping a column that the previous version still selects.

**Why it happens:** During any rolling deploy, old and new code run **simultaneously against the same database**. A migration that is only compatible with the new code breaks the old one instantly, and a migration only compatible with the old code makes rollback impossible.

**How expand-and-contract solves it:** Split every breaking change into backward-compatible steps deployed separately. To rename a column: **expand** — add the new column and write to both; **migrate** — backfill existing rows; **transition** — deploy code that reads the new column; **contract** — stop writing the old one and drop it, in a later release. Each step is independently safe to roll back. It is more work and more deploys, and it converts a guaranteed-outage operation into a routine one.

### 4. Migrations that lock a large table
**What you see:** A migration on a 200-million-row table takes an exclusive lock, queries pile up behind it, connection pools exhaust, and the application is down for twenty minutes even though nothing "failed."

**Why it happens:** Some DDL operations lock the table for their full duration, and duration scales with table size.

**How this topic solves it:** It makes you check what each operation actually does in your specific database version, and use the safe variants — creating indexes concurrently, adding nullable columns rather than defaulted ones, backfilling in small batches with pauses, and setting a short lock timeout so the migration fails fast rather than blocking traffic. This is one of the few areas where knowing your database's exact behavior is genuinely required.

## The Price You Pay

- **Backward compatibility is ongoing discipline.** Every schema change becomes a multi-release sequence. Someone has to remember to run the contract step, and in practice the cleanup is what gets forgotten, leaving dual-write code and dead columns for years.
- **More deploys, more coordination.** A rename that was one migration becomes four releases spread over days.
- **Temporary complexity in the code.** Dual-writing and reading-with-fallback is ugly code that exists only during a transition, and it is a source of bugs if the two paths diverge.
- **Infrastructure cost.** Blue-green doubles the environment during a switch; canaries require traffic-splitting and automated comparison of metrics between versions.
- **Long-running backfills are their own operational task.** They must be resumable, throttled so they do not saturate the database, and monitored — sometimes for days.
- **Rollback is not always possible.** Once you have dropped a column or transformed data destructively, there is no going back. Ordering steps so that the irreversible one comes last, and only after the new version has been stable, is the discipline that matters.

## When You Need It — and When You Don't

| Full zero-downtime discipline when | Simpler is acceptable when |
|---|---|
| The system has real users across time zones | Internal tools with a tolerant, known audience |
| You deploy frequently (which you should) | Genuinely low-traffic systems with agreed windows |
| An outage has financial or contractual cost | A pre-launch product with no users |
| Tables are large enough that migrations take real time | Tables are small and migrations are instant |

**Backward-compatible migrations, though, are worth it as a default habit** — the cost is small, and the failure they prevent is severe and abrupt.

## Why This Shows Up in Interviews

"How do you deploy this without downtime?" and "how do you change this schema safely?" are practical questions that reliably distinguish candidates with production experience. The high-signal answer names rolling deploys with health checks and connection draining, canary or blue-green for risk control, and — most importantly — explains that old and new code run concurrently, so schema changes must be backward compatible via expand-and-contract. Mentioning that a large-table migration can lock and take the site down, and how you would avoid it, is a detail that only comes from having been through it.

## How It Connects

Zero-downtime deploys are built on **load balancer** health checks and draining (topic 7) and are largely automated by **Kubernetes** rolling updates and readiness probes (topic 44). Canary analysis depends entirely on **observability** (topic 43) to compare versions. The dual-version problem is a form of the API compatibility concern in **message formats** (topic 35) and **microservices communication** (topic 31). Safe migrations interact with **replication** lag (topic 13) and **isolation levels** (topic 37), and the whole practice is what makes frequent, low-risk delivery compatible with the **availability** targets from topic 5.

**Next:** [Testing Distributed Systems](../46-testing-distributed-systems-load-testing-and-chaos-engineering/why.md) — finding out whether any of this actually works before your users do.
