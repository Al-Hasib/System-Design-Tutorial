# Practice & Interview Questions

**1. Why does simply stopping the old version and starting the new one cause downtime, even briefly?**
There's an inevitable gap — however small — where no instance is available to serve requests, because the old ones have stopped and the new ones haven't finished starting yet. At real scale, even a very short gap causes a visible number of failed in-flight requests.

**2. How does a rolling deployment avoid a zero-instance gap?**
It replaces instances a few at a time — starting a new-version instance, waiting for its readiness probe to pass, then terminating one old-version instance, and repeating — so the total pool of ready instances never drops to zero throughout the process.

**3. Why must old and new application versions be backward-compatible with each other during a rolling deployment?**
Because both versions serve live traffic simultaneously for the duration of the rollout. If the new version changes an API response shape or a schema element in a way the old version can't handle (or vice versa), real requests fail during that overlap window, not just in theory.

**4. Compare blue-green and canary deployments on rollback speed and blast radius.**
Blue-green offers very fast rollback (just switch the router back to the untouched blue environment) but an all-or-nothing blast radius per cutover — a bad deploy affects 100% of traffic the moment it's switched. Canary limits blast radius by ramping the new version to a small percentage of traffic first, catching problems before they affect everyone, but its rollback/ramp-down is more gradual and it requires ongoing metric-watching to decide when to proceed.

**5. What's the main cost trade-off of a blue-green deployment compared to a rolling deployment?**
Blue-green requires running a complete, fully-scaled second environment alongside the current one, at least temporarily — roughly doubling infrastructure cost during the switch — whereas a rolling deployment reuses existing capacity, replacing instances gradually without needing double the infrastructure.

**6. Why is renaming a database column and switching the application to the new name in a single deploy dangerous under a rolling deployment?**
During the rollout, old-version pods are still running and querying the column under its original name. If that deploy renames the column outright, those old-version pods immediately start failing because the column they expect no longer exists — a direct consequence of old and new versions needing to coexist safely during the rollout window.

**7. Describe the four steps of the expand-contract pattern for a database migration.**
Expand: add the new schema element without touching the old one. Migrate: deploy application code that writes to both old and new structures and backfills historical data. Contract: once fully rolled out and backfilled, stop writing to the old structure and switch reads fully to the new one. Cleanup: in a later, separate migration, drop the now-unused old structure.

**8. Why is expand-contract done as multiple separate deploys rather than one atomic change?**
Because each step needs to be safe for any mix of old and new application instances running simultaneously during a rollout. Combining steps (e.g., renaming and switching reads in one deploy) would break whichever instances are still running the version that expects the old structure, exactly the failure mode expand-contract is designed to avoid.

**9. Scenario: You need to deploy a critical fix as fast and safely as possible, want a very fast rollback option, and can tolerate temporarily higher infrastructure cost. Which deployment strategy fits best, and why?**
Blue-green — it lets you deploy the fix into a fully separate environment, verify it, and cut over atomically, with a near-instant rollback available (switching the router back to the still-running old environment) if something's wrong — exactly matching the priorities of speed and safe, fast rollback over cost efficiency.

**10. True or False: A canary deployment guarantees that a bad new version will never affect any real users.**
False. A canary deployment limits the blast radius by exposing the new version to only a small percentage of traffic first, but that percentage of real users is still affected if the new version has a problem — the goal is to catch and stop the rollout before it reaches 100% of traffic, not to eliminate all risk to any user.
