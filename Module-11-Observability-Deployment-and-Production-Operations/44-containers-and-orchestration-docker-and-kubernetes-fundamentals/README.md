# Containers & Orchestration: Docker & Kubernetes Fundamentals

**Difficulty:** Intermediate
**Estimated length:** 16-20 min
**Prerequisites:** [34 - Web Server Internals: Concurrency, Threading & Content Serving](../../Module-08-Protocols-Formats-and-Security/34-web-server-internals-concurrency-and-content-serving/README.md), [07 - Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md)

## Learning Objectives

- Explain what a container actually is, and how it differs from a virtual machine.
- Describe what a container image is and why it solves the "works on my machine" problem.
- Explain what an orchestrator (Kubernetes) actually does: scheduling, scaling, self-healing, and service discovery for containers.
- Describe the core Kubernetes building blocks — Pods, Deployments, Services — and what each one is responsible for.
- Explain liveness and readiness probes and why they matter for a container's actual availability.

## Script

### Hook / Intro

Throughout this course we've talked about "servers," "services," and "instances" somewhat abstractly — a box on a diagram that runs your code. In modern backend engineering, that box is almost always a **container**, scheduled and managed by an **orchestrator** — usually Kubernetes. This isn't incidental infrastructure trivia; it directly shapes how load balancing, service discovery, and scaling (all covered earlier in this course) actually get implemented in practice. Today we open up what a container actually is, why it beat the alternative (virtual machines) for this job, and what Kubernetes is actually doing when you tell it to "just deploy this."

### Containers vs. Virtual Machines

A **virtual machine** virtualizes an entire computer, including its own full operating system kernel, running on top of a hypervisor. This gives strong isolation, but each VM carries the overhead of a complete OS — gigabytes of disk, real memory overhead, and a slow boot time (often tens of seconds to minutes). A **container**, by contrast, virtualizes at the operating system level: containers running on the same host share the host's OS kernel, but each gets its own isolated filesystem, process namespace, and network stack, enforced by kernel features like Linux namespaces and cgroups. Because containers don't carry their own kernel, they're dramatically lighter — megabytes instead of gigabytes, and they start in milliseconds to a few seconds instead of minutes. The isolation is weaker than a full VM's (a kernel-level vulnerability could theoretically cross container boundaries in a way it can't cross VM boundaries), but for running many instances of your own trusted application code, that trade-off overwhelmingly favors containers, which is exactly why they became the default unit of deployment for backend services.

### Images: Solving "Works on My Machine"

A **container image** is a packaged, immutable snapshot containing your application code, its runtime (e.g., a specific Node.js or Python version), its libraries, and its configuration — everything needed to run it, built once from a `Dockerfile` and then run identically anywhere a container runtime is installed. This directly kills the classic "works on my machine" problem: if it runs correctly inside the image on your laptop, it runs identically on a teammate's laptop, in CI, and in production, because it's the literal same filesystem and dependencies every time — not "hopefully the same versions got installed." Images are typically stored in a **registry** (Docker Hub, or a private equivalent) and are built in layers, where unchanged layers are cached and reused across builds, keeping rebuilds and image pulls fast.

### What an Orchestrator Actually Does

Running one container on one machine is easy — running hundreds of containers, across dozens of machines, resiliently, is the actual hard problem, and that's what **Kubernetes** (or a similar orchestrator) solves. Concretely, an orchestrator handles: **scheduling** — deciding which physical machine each container instance actually runs on, based on available resources; **scaling** — automatically adding or removing container instances based on load (directly implementing the horizontal scaling concept from Module 1); **self-healing** — automatically restarting a container that crashes, and rescheduling it elsewhere if the machine it was running on fails entirely; **service discovery and load balancing** — giving a stable name/address to a group of container instances so other services can reach "the payments service" without knowing or caring which specific instances currently exist or where they're running (recall service discovery from Module 7); and **rolling updates** — replacing old container instances with new ones gradually, which is the actual mechanism underneath the zero-downtime deployment strategies we'll cover next video.

### Kubernetes' Core Building Blocks

Kubernetes organizes all of this around a small set of core objects. A **Pod** is the smallest deployable unit — one or more tightly-coupled containers that always get scheduled together on the same machine, sharing a network namespace (usually just one application container per pod, in practice). A **Deployment** describes the *desired state* of a set of pods — "I want 5 replicas of this container image running at all times" — and Kubernetes continuously works to make reality match that desired state, restarting or rescheduling pods as needed to maintain the count. A **Service** provides that stable network identity we just mentioned — a single, unchanging address that routes to whichever pods are currently healthy and part of a Deployment, even as individual pods are replaced, rescheduled, or scaled up and down underneath it — functioning as an internal load balancer.

Kubernetes also needs to know whether a pod is actually healthy enough to receive traffic, which is where **liveness and readiness probes** come in — periodic health checks the orchestrator runs against each pod. A **liveness probe** answers "is this container still alive, or should it be killed and restarted?" (catching a hung or deadlocked process). A **readiness probe** answers "is this container ready to receive traffic right now?" (a pod might be alive but still warming up a cache or waiting on a dependency, and shouldn't receive requests yet) — a pod that fails its readiness probe is automatically removed from a Service's routing until it passes again, without being restarted the way a failed liveness probe would trigger.

### Real-World Example

Consider deploying our recurring example of a payments microservice. You build a Docker image containing the service and its exact dependencies, push it to a registry, and define a Kubernetes Deployment requesting 6 replicas. Kubernetes schedules those 6 pods across your available machines, and a Service gives the rest of your system one stable address — `payments-service` — to call, regardless of which specific pods are currently running or where. If traffic spikes, an autoscaler (built on the same scaling mechanism) increases the replica count to 12; if one of the underlying machines crashes, Kubernetes notices the pods it was running are gone and reschedules them elsewhere, restoring the desired 12 replicas without a human involved. And when a new version needs deploying, readiness probes ensure a new pod doesn't receive live traffic until it's actually confirmed ready — directly preventing the "new instance receives traffic before it's actually initialized" failure mode.

### Recap

Containers virtualize at the OS level, sharing the host kernel but isolating filesystem, processes, and network — dramatically lighter and faster than full virtual machines, at the cost of slightly weaker isolation. Container images package an application and everything it needs into an immutable, portable unit, solving "works on my machine" for good. Kubernetes (an orchestrator) handles scheduling, scaling, self-healing, service discovery, and rolling updates for many containers across many machines — organized around Pods (the smallest deployable unit), Deployments (desired-state management), and Services (stable network identity), with liveness and readiness probes ensuring only genuinely healthy pods receive traffic.

### What's Next

Now that we know how containers are scheduled and kept healthy, next video covers exactly how a new version actually replaces an old one in production without downtime — building directly on the rolling-update and readiness-probe mechanisms we just introduced — plus the trickier problem of changing a database's schema underneath a system that can't stop running.

## Key Takeaways

- Containers virtualize at the OS level (sharing the host kernel, isolating via namespaces/cgroups), making them far lighter and faster to start than full virtual machines, at the cost of somewhat weaker isolation.
- A container image packages code, runtime, and dependencies into an immutable, portable unit built once and run identically anywhere — solving "works on my machine."
- Kubernetes (an orchestrator) handles scheduling, scaling, self-healing, service discovery, and rolling updates for containers across many machines.
- Pods are the smallest deployable unit; Deployments manage desired replica state; Services provide a stable network identity that load-balances across healthy pods.
- Liveness probes decide whether to restart a hung container; readiness probes decide whether a pod should currently receive traffic — distinct checks for distinct problems.
