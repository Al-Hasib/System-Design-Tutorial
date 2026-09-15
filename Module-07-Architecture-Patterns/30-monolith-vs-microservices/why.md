# Why This Topic Matters: Monolith vs Microservices

> **In one sentence:** This is the most consequential and most frequently botched architectural decision in modern software — teams adopt microservices for technical reasons when the real justification is organizational, and pay enormous complexity costs for a problem they didn't have.

## The World Before This Idea

**The monolith that outgrew its team.** One codebase, 400 engineers. Every deploy requires coordinating across teams. The test suite takes 90 minutes and is flaky, so merges queue up. A memory leak in the reporting module takes down checkout, because it's the same process. The whole application must scale together, so you run 200 instances of everything just to handle load on one hot endpoint. Nobody can upgrade a shared library because it would break forty other places.

**The microservices migration that made it worse.** So the team splits into 60 services. Now a single user action traverses eight services. A bug means reading eight repositories and correlating eight sets of logs. Local development requires running a dozen containers. Services share a database anyway, so they still can't deploy independently — you've built a distributed monolith, which has all the complexity of distribution and none of the autonomy.

Both are real, and both are common. The failure mode isn't picking wrong; it's picking without knowing what problem you're solving.

## The Problems It Solves

### 1. Deployment coupling that slows every team
**What you see:** Shipping a one-line change means waiting for a release train, a 90-minute pipeline, and everyone else's changes to be green.

**Why it happens:** One deployable unit means one deployment cadence, set by the slowest and riskiest change in the batch.

**How microservices solve it:** Each service deploys on its own schedule, with its own pipeline and its own risk. A team can ship ten times a day without asking anyone. **This is the primary, and arguably only compelling, reason to adopt microservices** — and note that it's an organizational benefit, not a technical one.

### 2. Scaling everything to scale one thing
**What you see:** The image-processing endpoint needs 64 GB of RAM, so every instance of the entire application gets 64 GB, including the ones only serving static JSON.

**Why it happens:** A monolith scales as a unit. Resource requirements are the maximum across all its functions.

**How microservices solve it:** Scale the image service to 20 memory-heavy instances and the API service to 100 cheap ones. Independent resource profiles, independently tuned — a real and measurable cost win at scale.

### 3. Blast radius of a single failure
**What you see:** A bug in an obscure background feature exhausts the heap and takes down the whole application, including revenue-critical paths.

**Why it happens:** One process, one memory space, one set of threads. There is no isolation between modules.

**How microservices solve it:** Process and network boundaries are hard boundaries. The recommendation service OOM-ing cannot consume the checkout service's memory. Combined with circuit breakers, failure stays contained.

### 4. Technology lock-in across the whole codebase
**What you see:** A machine learning feature that would be trivial in Python has to be written in Java because that's what the monolith is.

**Why it happens:** One codebase, one runtime, one dependency tree.

**How microservices solve it:** Each service picks its own stack. Real benefit — and also the most over-used justification, because polyglot fleets multiply the operational surface enormously. Most organizations should deliberately restrict themselves to two or three stacks.

## The Price You Pay

Microservices are a trade of *local* simplicity for *global* complexity, and the bill is large:

- **Every function call becomes a network call.** It can fail, time out, retry, and duplicate. Latency goes from nanoseconds to milliseconds. Every one of these calls now needs timeouts, retries, circuit breakers, and fallbacks.
- **Transactions disappear.** Operations spanning services need sagas or 2PC. Data that used to be joined is now denormalized and eventually consistent.
- **Observability becomes mandatory infrastructure.** Without distributed tracing, correlation IDs, and centralized logs, you cannot debug anything. This must exist *before* the split, not after.
- **Operational cost multiplies.** 60 services means 60 pipelines, 60 sets of dashboards and alerts, 60 on-call surfaces, 60 dependency-upgrade streams.
- **Local development gets hard.** Running the system on a laptop may become impossible.
- **Getting boundaries wrong is expensive.** Wrong boundaries mean chatty cross-service calls and changes that require coordinated deploys across three teams — exactly the coupling you split to escape, now with network latency added.

**The strongly recommended path:** start with a well-modularized monolith. Enforce internal boundaries in code. Extract a service only when a specific, concrete pain — a team blocked on deploys, a component with a wildly different scaling profile — justifies it. This "modular monolith first" approach gives you most of the structural benefit at a fraction of the cost, and it lets you learn where the boundaries actually are before making them expensive to change.

## When You Need It — and When You Don't

| Microservices when | Monolith when |
|---|---|
| Many teams blocked by a shared deploy pipeline | Fewer than ~20-30 engineers |
| Components have genuinely divergent scaling needs | The domain isn't well understood yet |
| You need fault isolation between critical and non-critical | You lack mature CI/CD and observability |
| Different parts genuinely need different tech | Development speed matters more than independence |
| You already have strong DevOps maturity | Strong consistency across the domain is required |

## Why This Shows Up in Interviews

This question is a judgment test more than a knowledge test. Reciting microservice benefits scores poorly; interviewers have seen too many failed migrations. The high-signal answer starts with "it depends on team size and organizational structure, not on traffic," recommends a modular monolith as the default, names the specific triggers that would justify splitting, and is candid about the costs — distributed transactions, observability requirements, operational overhead. Mentioning Conway's Law and the distributed-monolith anti-pattern shows you've thought about this beyond the blog-post level.

## How It Connects

Choosing microservices is what makes most of Module 6 mandatory rather than optional: **distributed transactions** (topic 28), **idempotency** (topic 29), and **circuit breakers** (topic 26) all become required. It creates the need for **service discovery** and inter-service **communication** (topic 31), an **API gateway** (topic 9), **message queues** (topic 20), and **observability** (topic 43). **Domain-driven design** (topic 32) is the discipline for drawing the boundaries, and **containers and Kubernetes** (topic 44) are how you operate the result.

**Next:** [Microservices Communication & Service Discovery](../31-microservices-communication-and-service-discovery/why.md) — how the services you just created find and talk to each other.
