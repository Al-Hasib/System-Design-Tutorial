# Why This Topic Matters: Load Balancing

> **In one sentence:** Running ten servers is worthless unless something intelligently decides which one each request goes to — and that "something" is also what keeps traffic away from the servers that are broken.

## The World Before This Idea

You've scaled horizontally: five application servers, all identical, all healthy. Now what? The user's browser only knows one hostname. Somebody has to translate "one hostname" into "one of these five machines."

The naive answer is DNS round-robin: publish five A records and let clients pick. It works badly. DNS results are cached aggressively by resolvers and clients, so distribution is lumpy. Worse, DNS has no idea whether a server is alive — when one dies, a fifth of your users keep being sent to a dead box for as long as the TTL lasts, and there's nothing you can do about it in real time.

## The Problems It Solves

### 1. Dead servers that keep receiving traffic
**What you see:** One backend crashes. 20% of requests start failing. Nothing automatically stops it.

**Why it happens:** Whatever is distributing traffic has no notion of backend health.

**How load balancing solves it:** Active health checks. The load balancer probes each backend on an interval, and a node that fails is removed from rotation within seconds. This is arguably the load balancer's *most* valuable job — more valuable than the load spreading itself, because it converts a total failure for some users into no failure for anyone.

### 2. Uneven load that wastes the fleet
**What you see:** Two servers are pinned at 95% CPU and requests are queuing, while three sit at 15%. Average utilization looks fine on the dashboard.

**Why it happens:** Naive distribution ignores what each request actually costs. If requests have wildly varying work (a search query versus a health ping), pure round-robin will pile heavy requests unevenly.

**How load balancing solves it:** Algorithms that account for reality — least-connections routes to the backend currently handling the fewest requests; weighted variants account for heterogeneous hardware; least-response-time factors in observed latency. Picking the right algorithm for your request profile is what turns raw capacity into usable capacity.

### 3. Deploys that require downtime
**What you see:** Every release has a maintenance window.

**Why it happens:** If traffic goes straight to servers, you can't update a server without interrupting the users on it.

**How load balancing solves it:** Connection draining and rolling deploys. Take one node out of rotation, let its in-flight requests finish, deploy, health-check it back in, repeat. The load balancer is the mechanism that makes zero-downtime deployment, blue-green, and canary releases possible at all.

### 4. Routing decisions that need to see inside the request
**What you see:** You want `/api/*` to go to the API fleet, `/static/*` to the asset servers, and 5% of traffic to a canary build — but the traffic distributor only understands IP addresses and ports.

**Why it happens:** A Layer 4 load balancer operates on TCP/UDP connections. It's fast and protocol-agnostic, but it cannot read paths, headers, or cookies.

**How this topic solves it:** Understanding the L4/L7 distinction. Layer 7 balancers parse HTTP, enabling path- and header-based routing, TLS termination, per-request retries, and content-aware rules — at the cost of more CPU and higher latency per request. Knowing which layer you need is the actual design decision.

## The Price You Pay

- **It's a new single point of failure.** The thing that protects you from server failure can itself fail. Real deployments need redundant load balancers with failover (a floating IP, an anycast address, or a managed cloud LB).
- **It's a bottleneck.** All traffic flows through it. It must be sized for peak, and L7 inspection costs meaningfully more CPU than L4 forwarding.
- **Sticky sessions are a trap.** Session affinity is the tempting shortcut for stateful apps, but it undermines even distribution, breaks when a node dies, and makes draining harder. The better fix is to make the application stateless.
- **Health checks can lie in both directions.** A check that only confirms the process is alive will keep routing to a node whose database connection pool is exhausted. A check that's too aggressive will eject healthy nodes during a transient blip and amplify an incident.

## When You Need It — and When You Don't

| You need one when | You can skip it when |
|---|---|
| You run more than one instance of anything | Genuinely single-instance, low-stakes systems |
| You want zero-downtime deploys | Downtime windows are acceptable |
| You need TLS termination in one place | — |
| You want canary or blue-green releases | — |
| You need traffic shaped by path, header, or geography | — |

## Why This Shows Up in Interviews

A load balancer appears in essentially every system design diagram, which means drawing one earns you nothing. What earns credit is the follow-up: which algorithm and why, L4 or L7 and why, how health checks work, what happens when the load balancer itself fails, and how you'd avoid sticky sessions. Interviewers often probe the last one specifically, because it separates people who've operated a fleet from people who've read about one.

## How It Connects

Load balancing is the practical enabler of **horizontal scaling** and a core mechanism of **fault tolerance**. It's closely related to the **reverse proxy** (a load balancer is one specialized kind), sits next to the **API gateway** (which adds auth, rate limiting, and aggregation on top), and uses **consistent hashing** when backends hold state and you need stable key-to-node mapping. **Zero-downtime deployments** are built directly on its draining and health-check behavior.

**Next:** [Forward Proxy vs Reverse Proxy](../08-forward-proxy-vs-reverse-proxy/why.md) — the broader family of intermediaries that load balancers belong to.
