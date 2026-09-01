# Study Notes: Containers & Orchestration

## Definitions

- **Container:** An OS-level virtualized unit sharing the host's kernel, with its own isolated filesystem, process namespace, and network stack (via Linux namespaces/cgroups).
- **Virtual machine (VM):** A fully virtualized computer including its own OS kernel, running on a hypervisor.
- **Container image:** An immutable, packaged snapshot of an application, its runtime, and dependencies, built from a Dockerfile and run identically anywhere.
- **Registry:** A storage/distribution service for container images (e.g., Docker Hub, private registries).
- **Orchestrator:** A system (e.g., Kubernetes) that schedules, scales, heals, and manages many containers across many machines.
- **Pod:** The smallest deployable unit in Kubernetes — one or more tightly-coupled containers scheduled together, sharing a network namespace.
- **Deployment:** A Kubernetes object describing desired state (e.g., "5 replicas of image X"); Kubernetes continuously reconciles reality toward this state.
- **Service:** A stable network identity/address routing to whichever healthy pods currently belong to a Deployment.
- **Liveness probe:** A health check determining whether a container should be killed and restarted.
- **Readiness probe:** A health check determining whether a pod should currently receive traffic.

## Containers vs. Virtual Machines

| Aspect | Virtual Machine | Container |
|---|---|---|
| Virtualizes | Entire computer + own OS kernel | OS-level only (shares host kernel) |
| Size | Gigabytes | Megabytes |
| Startup time | Tens of seconds to minutes | Milliseconds to a few seconds |
| Isolation strength | Strong (separate kernel) | Weaker (shared kernel) |
| Typical use | Running different OSs, strong multi-tenant isolation | Packaging and running many instances of trusted application code |

## Kubernetes Core Objects

| Object | Responsible for |
|---|---|
| Pod | Smallest deployable unit; one or more co-located containers |
| Deployment | Desired replica count/state; reconciles actual state to match |
| Service | Stable address load-balancing across healthy pods in a Deployment |
| Liveness probe | Detects a hung/crashed container → triggers restart |
| Readiness probe | Detects whether a pod is ready for traffic → controls Service routing, no restart |

## What an Orchestrator Actually Provides

- **Scheduling:** deciding which machine runs which container, based on resources.
- **Scaling:** adding/removing instances based on load (horizontal scaling, automated).
- **Self-healing:** restarting crashed containers; rescheduling if a machine fails.
- **Service discovery/load balancing:** stable addressing regardless of which specific instances exist.
- **Rolling updates:** gradually replacing old instances with new ones (the mechanism behind zero-downtime deploys).

## Key Numbers / Facts

- Docker popularized containers for mainstream backend use starting around 2013; the underlying Linux kernel features (namespaces, cgroups) existed earlier.
- Kubernetes originated at Google (based on internal systems like Borg) and was open-sourced in 2014.
- Container images are built in layers; unchanged layers are cached and reused, speeding up builds and pulls.

## Summary

- Containers are lighter, faster-starting alternatives to VMs, trading some isolation strength for efficiency — the default unit for deploying backend services today.
- Images make deployments reproducible and portable, eliminating "works on my machine."
- Kubernetes automates scheduling, scaling, healing, and traffic routing for containers via Pods, Deployments, and Services, using liveness/readiness probes to know what's actually healthy.
