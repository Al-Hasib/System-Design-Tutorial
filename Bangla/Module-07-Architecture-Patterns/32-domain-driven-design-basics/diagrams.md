# ডায়াগ্রাম: Domain-Driven Design Basics

## ১. "Customer"-এর ভিন্ন ভিন্ন অর্থসহ Bounded Contexts

```mermaid
flowchart TB
    subgraph Sales["Sales Bounded Context"]
        SC[Customer: anyone who has shown purchase interest]
    end
    subgraph Support["Support Bounded Context"]
        SUC[Customer: anyone with an active account]
    end
    subgraph Billing["Billing Bounded Context"]
        BC[Customer: anyone with a valid payment method]
    end

    Sales -- Anti-Corruption Layer --> Billing
    Support -- Anti-Corruption Layer --> Billing
```
*প্রতিটি bounded context-এর "Customer"-এর নিজস্ব মডেল আছে; একটি মডেল শেয়ার করার পরিবর্তে anti-corruption layer এদের মধ্যে অনুবাদ করে।*

## ২. Bounded Contexts-কে Microservices-এ ম্যাপ করা

```mermaid
flowchart LR
    subgraph Domain["E-Commerce Business Domain"]
        direction LR
        OrderCtx[Order Management Bounded Context]
        InvCtx[Inventory Bounded Context]
        BillCtx[Billing Bounded Context]
    end

    OrderCtx --> OrderSvc[Order Service]
    InvCtx --> InvSvc[Inventory Service]
    BillCtx --> BillSvc[Billing Service]

    OrderSvc -- API/event --> InvSvc
    OrderSvc -- API/event --> BillSvc
```
*প্রতিটি bounded context স্বাভাবিকভাবেই তার নিজস্ব microservice-এ ম্যাপ করে, একটি শেয়ার্ড ডেটাবেসের পরিবর্তে সুনির্দিষ্ট API/ইভেন্টের মাধ্যমে যোগাযোগ করে।*

## ৩. Core, Supporting, এবং Generic Subdomains

```mermaid
flowchart TB
    Core["Core Domain\n(e.g., Search & Matching)\nInvest heavily, best engineers"]
    Supporting["Supporting Subdomain\n(e.g., Warehouse Fulfillment)\nBuild adequately, keep lean"]
    Generic["Generic Subdomain\n(e.g., Authentication, Payments)\nBuy/use existing solutions"]

    Core --- Supporting --- Generic
```
*subdomain-গুলোকে শ্রেণিবদ্ধ করা সিদ্ধান্ত নিতে সাহায্য করে কোথায় গভীর কাস্টম ইঞ্জিনিয়ারিং বিনিয়োগ করতে হবে বনাম কোথায় একটি বিদ্যমান সমাধান কিনতে হবে।*
</content>
