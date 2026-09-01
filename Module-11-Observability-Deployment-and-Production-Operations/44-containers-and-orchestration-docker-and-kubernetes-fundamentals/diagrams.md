# Diagrams: Containers & Orchestration

## 1. Virtual Machines vs. Containers

```mermaid
flowchart TB
    subgraph VMs["Virtual Machines"]
        direction TB
        HW1[Physical Hardware] --> HV[Hypervisor]
        HV --> VM1["VM 1<br/>Full Guest OS + App"]
        HV --> VM2["VM 2<br/>Full Guest OS + App"]
    end

    subgraph Containers["Containers"]
        direction TB
        HW2[Physical Hardware] --> Host[Host OS Kernel]
        Host --> CT1["Container 1<br/>App only, shares kernel"]
        Host --> CT2["Container 2<br/>App only, shares kernel"]
    end
```
*Each VM carries its own full OS kernel, adding significant overhead. Containers share the host's kernel and isolate only the application layer, making them far lighter and faster to start.*

## 2. Kubernetes Core Objects Working Together

```mermaid
flowchart TB
    Dep["Deployment<br/>desired state: 5 replicas of image X"] --> P1[Pod 1]
    Dep --> P2[Pod 2]
    Dep --> P3[Pod 3]
    Dep --> P4[Pod 4]
    Dep --> P5[Pod 5]

    Svc["Service: payments-service<br/>(stable address)"] --> P1
    Svc --> P2
    Svc --> P3
    Svc --> P4
    Svc --> P5

    Caller[Other services] --> Svc
```
*A Deployment keeps the desired number of Pods running; a Service gives callers one stable address that routes to whichever Pods are currently healthy, regardless of which specific ones exist at any moment.*

## 3. Liveness vs. Readiness Probes

```mermaid
flowchart LR
    Pod[Running Pod] --> Live{"Liveness probe:<br/>still alive?"}
    Live -->|No, hung/crashed| Restart[Kubernetes kills and restarts the container]
    Live -->|Yes| Ready{"Readiness probe:<br/>ready for traffic?"}
    Ready -->|No, still warming up| Wait["Removed from Service routing<br/>(not restarted, just excluded)"]
    Ready -->|Yes| Route[Included in Service routing - receives traffic]
```
*A failed liveness probe triggers a restart; a failed readiness probe just excludes the pod from receiving traffic until it recovers, without killing it — two different problems, two different responses.*
