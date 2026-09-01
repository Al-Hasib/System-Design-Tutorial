# Practice & Interview Questions

**1. What's the fundamental architectural difference between a container and a virtual machine?**
A virtual machine virtualizes an entire computer, including its own full OS kernel, running on a hypervisor. A container virtualizes only at the OS level — it shares the host's kernel but gets its own isolated filesystem, process namespace, and network stack via kernel features like namespaces and cgroups.

**2. Why are containers dramatically faster to start than virtual machines?**
A VM has to boot an entire guest operating system, including its own kernel, which takes real time. A container doesn't carry its own kernel at all — it shares the host's already-running kernel — so starting a container is closer to starting a regular process, taking milliseconds to a few seconds instead of minutes.

**3. What problem does a container image solve, and how?**
It solves "works on my machine" — the situation where an application behaves differently across environments due to inconsistent dependency versions or configuration. An image packages the application, its exact runtime, and all its dependencies into one immutable snapshot, built once and run identically wherever a container runtime exists, so there's no "hopefully this environment matches" guessing.

**4. List three responsibilities of an orchestrator like Kubernetes.**
Any three of: scheduling (deciding which machine runs which container), scaling (adding/removing instances based on load), self-healing (restarting crashed containers, rescheduling on machine failure), service discovery/load balancing (stable addressing across changing instances), or rolling updates (gradually replacing old instances with new ones).

**5. What is a Kubernetes Pod, and why is it described as the "smallest deployable unit" rather than a single container?**
A Pod is one or more tightly-coupled containers scheduled together on the same machine, sharing a network namespace. It's the smallest deployable unit because Kubernetes schedules and manages at the Pod level — even though most pods in practice contain just one application container, the abstraction allows tightly-coupled helper containers to be co-located when needed.

**6. What does a Kubernetes Deployment actually do?**
It describes the desired state of a set of pods — e.g., "5 replicas of this container image should be running" — and Kubernetes continuously reconciles the actual running state toward that desired state, restarting or rescheduling pods as needed if reality drifts from it (e.g., a pod crashes or a machine fails).

**7. What does a Kubernetes Service provide, and why is it necessary given that Pods can be created and destroyed frequently?**
A Service provides a single, stable network address that routes to whichever pods are currently healthy and part of a Deployment. Since individual pods can be rescheduled, restarted, or scaled up/down at any time (each potentially getting a new internal address), other services need a stable address to call that doesn't change as the underlying pods do — this is exactly what a Service provides.

**8. Explain the difference between a liveness probe and a readiness probe, and what happens when each one fails.**
A liveness probe checks whether a container is still functioning (not hung or deadlocked); if it fails, Kubernetes kills and restarts the container. A readiness probe checks whether a pod is currently able to handle traffic (e.g., it might be alive but still warming up); if it fails, the pod is simply removed from the Service's routing until it passes again — no restart occurs, just a temporary exclusion from traffic.

**9. Scenario: A new pod finishes starting up but needs 10 seconds to warm an in-memory cache before it can serve requests correctly. Which Kubernetes mechanism from this video ensures it doesn't receive traffic during that warm-up, and how?**
A readiness probe — configured to only report "ready" once the cache warm-up is complete (e.g., checking an internal health endpoint that reflects this). Until then, the pod fails its readiness check and is excluded from the Service's routing, so no traffic reaches it prematurely, without the pod being restarted.

**10. True or False: Because containers share the host's kernel, they provide exactly the same level of security isolation as virtual machines.**
False. Sharing the host kernel means containers have inherently weaker isolation than VMs — a kernel-level vulnerability could theoretically be exploited to cross container boundaries in a way that's much harder across VM boundaries, which have entirely separate kernels. This is an accepted trade-off for running many instances of your own trusted application code, not a claim of equivalent isolation strength.
