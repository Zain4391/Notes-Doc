# Service Boundaries

How to decide what belongs inside one service and what belongs in another: bounded contexts, data ownership, the signals that tell you where to cut, and the splits that look reasonable but hurt later.

**Contents**

1. [Overview](#overview)
2. [Split by business capability](#split-by-business-capability)
3. [Bounded contexts](#bounded-contexts)
4. [The same word means different things](#the-same-word-means-different-things)
5. [Relationships between contexts](#relationships-between-contexts)
6. [Data ownership](#data-ownership)
7. [Where to cut: signals](#where-to-cut-signals)
8. [Granularity](#granularity)
9. [Common mistakes](#common-mistakes)

Related: [Monolith vs Modular Monolith vs Microservices](monolith-vs-microservices.md) · [Reference Architecture](reference-architecture.md) · [NestJS Modules](../Nest/nest-module.md) (one module per boundary)

---

## Overview

A service boundary is a line with three jobs:

| Job | Meaning |
|---|---|
| **Own a capability** | One service is the single place where a business rule lives |
| **Own its data** | Only that service reads and writes its tables |
| **Hide its internals** | Others see an API or events, never the implementation |

Get the lines right and most changes touch one service. Get them wrong and every feature needs coordinated releases across several.

The examples use a food-delivery domain (customers, restaurants, orders, drivers), the same one used in the NestJS notes.

---

## Split by business capability

Cut along what the business does, not along technical layers.

```mermaid
flowchart LR
    subgraph BAD["Split by technical layer (avoid)"]
        direction TB
        UIL["UI service"]:::bad
        BLL["Business logic service"]:::bad
        DAL["Data access service"]:::bad
        UIL --> BLL --> DAL
    end
    subgraph GOOD["Split by business capability"]
        direction TB
        ORG["Ordering<br/>UI, logic and data"]:::good
        RE["Restaurants<br/>UI, logic and data"]:::good
        DE["Delivery<br/>UI, logic and data"]:::good
    end
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
```

With a layer split, adding one field touches all three services. With a capability split, the same change stays inside one.

---

## Bounded contexts

A **bounded context** (from Domain-Driven Design) is an area of the business where every term has one precise meaning. Each context usually becomes one service, or one module in a modular monolith.

```mermaid
flowchart TB
    subgraph CORE["Core domain: what makes the business different"]
        ORD["Ordering<br/>cart, order lifecycle, pricing"]:::core
        DEL["Delivery<br/>dispatch, tracking, driver assignment"]:::core
    end
    subgraph SUP["Supporting: needed, but not the differentiator"]
        RES["Restaurants<br/>menus, availability, preparation"]:::sup
        CAT["Catalog and search"]:::sup
    end
    subgraph GEN["Generic: could be bought or reused"]
        IDN["Identity<br/>accounts, login, roles"]:::gen
        PAY["Payments"]:::gen
        NOT["Notifications<br/>email, SMS, push"]:::gen
    end

    ORG -->|"asks to prepare"| RES
    ORD -->|"asks to dispatch"| DEL
    ORD -->|"charges"| PAY
    ORD -.->|"sends events to"| NOT
    DEL -.->|"sends events to"| NOT
    ORD -->|"reads menu from"| CAT
    RES -->|"publishes menu changes to"| CAT
    IDN -.->|"identity of the caller"| ORD
    IDN -.-> DEL

    classDef core fill:#c4b5fd,stroke:#6d28d9,color:#0f172a
    classDef sup fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef gen fill:#e5e7eb,stroke:#4b5563,color:#0f172a
```

| Kind | Meaning | Investment |
|---|---|---|
| Core | The reason the business wins | Build in-house, best people, most care |
| Supporting | Necessary but not unique | Build simply |
| Generic | Solved problem | Buy or use a library or SaaS where you can |

---

## The same word means different things

"Order" is not one concept. Each context models only what it needs, so there is **no single shared `Order` class**.

```mermaid
classDiagram
    class OrderingOrder {
        id
        customerId
        items
        totalAmount
        status
        deliveryAddress
    }
    class RestaurantTicket {
        orderId
        items
        specialInstructions
        preparationStatus
        estimatedReadyAt
    }
    class DeliveryJob {
        orderId
        pickupAddress
        dropoffAddress
        assignedDriverId
        deliveryStatus
    }
    class PaymentCharge {
        orderId
        amount
        method
        paymentStatus
    }
    OrderingOrder ..> RestaurantTicket : order.placed
    OrderingOrder ..> DeliveryJob : order.ready
    OrderingOrder ..> PaymentCharge : charge request
```

Contexts link by **identifier** (`orderId`) and **events**, not by sharing the object. That is why the event DTOs in [RabbitMQ: Configuration and Usage](../Nest/rabbitmq-config-use.md) carry only the fields each consumer needs.

---

## Relationships between contexts

How two contexts relate decides who has to change when something changes.

| Relationship | Meaning | Example |
|---|---|---|
| **Customer / supplier** | Downstream asks, upstream agrees to deliver what it needs | Ordering (downstream) negotiates menu fields with Restaurants |
| **Conformist** | Downstream accepts the upstream model as is | Using a payment provider's data shape directly |
| **Anti-corruption layer** | Downstream translates the upstream model into its own | An adapter that converts the payment provider's response into `PaymentCharge` |
| **Published language** | A documented shared contract (events or API schema) | The event names and payloads on the message broker |
| **Shared kernel** | A small piece of model both teams own together | A tiny shared `Money` or `Address` type (keep it very small) |

```mermaid
flowchart LR
    EXT["External payment provider<br/>(its own model)"]:::ext
    ACL["Anti-corruption layer<br/>translates"]:::acl
    PAYC["Payments context<br/>(our model)"]:::svc
    ORDC["Ordering context"]:::svc
    EXT -->|"provider schema"| ACL -->|"our schema"| PAYC
    ORDC -->|"published language:<br/>charge request / payment events"| PAYC
    classDef ext fill:#e5e7eb,stroke:#4b5563,color:#0f172a
    classDef acl fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
```

---

## Data ownership

Each service is the **only** reader and writer of its tables. Other services get data through its API or its events.

```mermaid
flowchart LR
    subgraph NO["Not allowed: reaching into another service's database"]
        direction TB
        OS1["Orders service"]:::bad
        RDB1[("Restaurants DB")]:::db
        OS1 -.->|"direct SQL join"| RDB1
    end
    subgraph YES["Allowed: through the owner"]
        direction TB
        OS2["Orders service"]:::good
        RS2["Restaurants service"]:::good
        RDB2[("Restaurants DB")]:::db
        OS2 -->|"API call or event"| RS2
        RS2 --> RDB2
    end
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
```

Consequences you design for:

| Need | Without shared tables |
|---|---|
| A report joins orders with restaurants | Keep a small **local copy** of the fields you need, updated by events, or build a read model |
| A foreign key to a customer | Store the `customerId` only; no database-level foreign key across services |
| Cross-service consistency | Use a saga (a sequence of local steps with compensations) instead of one transaction |

---

## Where to cut: signals

```mermaid
flowchart TD
    Q0(["Two pieces of functionality:<br/>same service or different?"]):::entry
    Q0 --> A{"Do they change together<br/>for the same business reason?"}
    A -->|"Yes"| SAME1["Keep together"]:::good
    A -->|"No"| B{"Do they need strongly consistent<br/>updates in one transaction?"}
    B -->|"Yes"| SAME2["Keep together, or rethink the rule<br/>so it can be eventually consistent"]:::good
    B -->|"No"| C{"Can each own its data<br/>without joining the other's tables?"}
    C -->|"No"| SAME3["Keep together until the data can be separated"]:::good
    C -->|"Yes"| D{"Different teams, release cadence<br/>or scaling needs?"}
    D -->|"Yes"| SPLIT["Separate services"]:::split
    D -->|"No"| MOD["Separate modules in one deployable.<br/>Split later if it hurts."]:::warn
    classDef entry fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef split fill:#c4b5fd,stroke:#6d28d9,color:#0f172a
    classDef warn fill:#fde68a,stroke:#b45309,color:#0f172a
```

| Signal that two things belong together | Signal that they can be apart |
|---|---|
| They change in the same commits | They change at different times for different reasons |
| One cannot work without the other's data | They exchange only identifiers and small messages |
| They must commit atomically | They can tolerate a delay between steps |
| The same small team owns both | Separate teams own them |

---

## Granularity

| Too coarse | Right-sized | Too fine ("nano-services") |
|---|---|---|
| Many unrelated rules in one service | One business capability, one team can own it | One function or table per service |
| Hard to change safely, large deploys | Changes mostly stay local | A single use case touches many services |
| Behaves like a monolith | Few, meaningful network calls | Chatty calls, heavy operations overhead |

A practical check: **can one team understand, change and deploy the service without asking another team?** If not, it is too entangled. If a use case needs five services to move, it is too fine.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Splitting by layer or by entity (`UserService`, `OrderService`, `ItemService`) | Every feature crosses all of them | Split by capability and bounded context |
| One shared `Order` model for the whole company | Any change ripples everywhere | Each context models its own version and links by id and events |
| Foreign keys or joins across services | Hidden coupling at the database level | Own the data; copy what you need via events |
| A "common" library full of domain classes | Becomes a shared kernel that forces lock-step releases | Share only tiny stable types or contracts |
| Cutting before the language of the domain is clear | The lines move constantly | Begin with modules in one deployable and extract later |
| Ignoring team structure | Services end up owned by nobody or by everybody | Align each service with one owning team |
