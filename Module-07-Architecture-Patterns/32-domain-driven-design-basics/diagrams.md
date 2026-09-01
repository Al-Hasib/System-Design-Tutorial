# Diagrams: Domain-Driven Design Basics

## 1. Bounded Contexts with Different Meanings of "Customer"

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
*Each bounded context has its own model of "Customer"; anti-corruption layers translate between them instead of sharing one model.*

## 2. Mapping Bounded Contexts to Microservices

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
*Each bounded context maps naturally to its own microservice, communicating over well-defined APIs/events rather than a shared database.*

## 3. Core, Supporting, and Generic Subdomains

```mermaid
flowchart TB
    Core["Core Domain\n(e.g., Search & Matching)\nInvest heavily, best engineers"]
    Supporting["Supporting Subdomain\n(e.g., Warehouse Fulfillment)\nBuild adequately, keep lean"]
    Generic["Generic Subdomain\n(e.g., Authentication, Payments)\nBuy/use existing solutions"]

    Core --- Supporting --- Generic
```
*Classifying subdomains helps decide where to invest deep custom engineering versus where to buy an existing solution.*
