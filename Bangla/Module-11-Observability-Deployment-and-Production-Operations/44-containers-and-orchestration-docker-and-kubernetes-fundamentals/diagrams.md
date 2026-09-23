# ডায়াগ্রাম: Containers & Orchestration

## ১. Virtual Machines বনাম Containers

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
*প্রতিটি VM তার নিজস্ব পূর্ণাঙ্গ OS kernel বহন করে, যা উল্লেখযোগ্য overhead যোগ করে। Container-গুলো host-এর kernel share করে এবং শুধুমাত্র application layer isolate করে, যা এগুলোকে অনেক বেশি হালকা এবং দ্রুত শুরু হওয়ার উপযোগী করে তোলে।*

## ২. Kubernetes-এর মূল Object একসাথে কাজ করছে

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
*একটি Deployment কাঙ্ক্ষিত সংখ্যক Pod চলমান রাখে; একটি Service caller-দের একটি স্থিতিশীল ঠিকানা দেয় যা বর্তমানে যে Pod-গুলো healthy তাদের দিকে route করে, যেকোনো মুহূর্তে কোন নির্দিষ্ট Pod বিদ্যমান তা নির্বিশেষে।*

## ৩. Liveness বনাম Readiness Probe

```mermaid
flowchart LR
    Pod[Running Pod] --> Live{"Liveness probe:<br/>still alive?"}
    Live -->|No, hung/crashed| Restart[Kubernetes kills and restarts the container]
    Live -->|Yes| Ready{"Readiness probe:<br/>ready for traffic?"}
    Ready -->|No, still warming up| Wait["Removed from Service routing<br/>(not restarted, just excluded)"]
    Ready -->|Yes| Route[Included in Service routing - receives traffic]
```
*একটি ব্যর্থ liveness probe একটি restart trigger করে; একটি ব্যর্থ readiness probe শুধু pod-টিকে ট্রাফিক গ্রহণ থেকে বাদ দেয় যতক্ষণ না এটি সুস্থ হয়, একে kill না করেই — দুটি ভিন্ন সমস্যা, দুটি ভিন্ন সাড়া।*
</content>
