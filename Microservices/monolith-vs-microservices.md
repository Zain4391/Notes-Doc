# Monolith vs Modular Monolith vs Microservices

The three common ways to structure a back end, what each one looks like at runtime, how a single request differs between them, and how to decide which one to use (and how to move from one to the next).

**Contents**

1. [Overview](#overview)
2. [The monolith](#the-monolith)
3. [The modular monolith](#the-modular-monolith)
4. [Microservices](#microservices)
5. [Side-by-side comparison](#side-by-side-comparison)
6. [How one request differs](#how-one-request-differs)
7. [Scaling](#scaling)
8. [Choosing an architecture](#choosing-an-architecture)
9. [Migrating step by step](#migrating-step-by-step)
10. [Common mistakes](#common-mistakes)

Related: [Service Boundaries](service-boundaries.md) (where to cut) · [Reference Architecture](reference-architecture.md) (the full picture) · [NestJS Modules](../Nest/nest-module.md) (modules as the building block of a modular monolith)

---

## Overview

| Style | Deployables | Database | Module-to-module calls | Best when |
|---|---|---|---|---|
| **Monolith** | 1 | 1 shared | In-process, unrestricted | Small team, small or unclear domain |
| **Modular monolith** | 1 | 1 (schema or tables per module) | In-process, through each module's public API or events | Most products; the default starting point |
| **Microservices** | Many | One per service | Network (HTTP, gRPC, messages) | Many teams, independent release and scaling needs |

The axis that matters is not "old vs new", it is **how strongly the parts are isolated** and what you pay for that isolation.

```mermaid
flowchart LR
    A["Monolith<br/>one codebase, one deploy,<br/>no enforced boundaries"]:::weak
    B["Modular monolith<br/>one deploy,<br/>enforced module boundaries"]:::mid
    C["Microservices<br/>many deploys,<br/>boundaries enforced by the network"]:::strong
    A -->|"add boundaries"| B -->|"split deployment and data"| C
    classDef weak fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mid fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef strong fill:#bbf7d0,stroke:#15803d,color:#0f172a
```

---

## The monolith

Everything is built, deployed and scaled as one unit. Any class can call any other class, and every feature reads and writes the same database.

```mermaid
flowchart TB
    C["Clients<br/>web and mobile"]:::client
    subgraph APP["One process, one deployment"]
        UI["Controllers / HTTP layer"]:::gw
        subgraph BL["Business logic (no enforced boundaries)"]
            O["Orders"]:::svc
            R["Restaurants"]:::svc
            D["Delivery"]:::svc
            U["Users"]:::svc
        end
        DA["Data access layer"]:::svc
    end
    DB[("One shared database")]:::db

    C --> UI
    UI --> O & R & D & U
    O -.->|"direct method calls"| R
    O -.-> D
    D -.-> U
    O & R & D & U --> DA
    DA --> DB

    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
```

| Strengths | Weaknesses |
|---|---|
| Simple to build, run, debug and test | Everything is deployed together: a small change ships the whole app |
| One transaction can span any data | One bug or memory leak can take down all features |
| No network between features, so it is fast | Scaling means copying the whole app, even if one feature is hot |
| One repo, one pipeline, one thing to monitor | Without discipline, code becomes tangled ("big ball of mud") |

---

## The modular monolith

Still **one deployable and one database server**, but the code is divided into modules with explicit boundaries. A module exposes a small public API (or publishes events) and hides everything else. Other modules never touch its tables directly.

```mermaid
flowchart TB
    C["Clients"]:::client --> HTTP["HTTP layer"]:::gw
    subgraph APP["One process, one deployment"]
        subgraph MO["Orders module"]
            OA["Public API"]:::gw
            OI["Internal logic"]:::svc
            OA --> OI
        end
        subgraph MR["Restaurants module"]
            RA["Public API"]:::gw
            RI["Internal logic"]:::svc
            RA --> RI
        end
        subgraph MD["Delivery module"]
            DAPI["Public API"]:::gw
            DI["Internal logic"]:::svc
            DAPI --> DI
        end
        BUS(["In-process event bus"]):::mq
    end
    subgraph PG["One PostgreSQL instance"]
        S1[("orders schema")]:::db
        S2[("restaurants schema")]:::db
        S3[("delivery schema")]:::db
    end

    HTTP --> OA & RA & DAPI
    OA -.->|"call"| RA
    OI -->|"publish"| BUS
    BUS -->|"deliver"| RI
    BUS -->|"deliver"| DI
    OI --> S1
    RI --> S2
    DI --> S3

    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mq fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
```

The rules that make it "modular":

| Rule | Why |
|---|---|
| A module is used only through its public API or its events | Internals can change without breaking others |
| A module owns its tables; no cross-module joins or foreign keys into another module's tables | Keeps the data split-ready |
| Modules communicate with the same style they would use as services (call an API, publish an event) | Extracting one later is a deployment change, not a rewrite |

In NestJS this maps directly to a module that only `exports` its public service (see [NestJS Modules](../Nest/nest-module.md)) and to the event pattern in [RabbitMQ: Configuration and Usage](../Nest/rabbitmq-config-use.md).

---

## Microservices

Each service is its own deployable, **owns its own database**, and is reached only over the network. Services are released, scaled and failed independently.

```mermaid
flowchart TB
    C["Clients"]:::client --> GW["API Gateway"]:::gw
    GW --> O & R & D
    subgraph SVC["Independently deployed services"]
        O["Orders service"]:::svc
        R["Restaurants service"]:::svc
        D["Delivery service"]:::svc
    end
    O --- ODB[("Orders DB")]:::db
    R --- RDB[("Restaurants DB")]:::db
    D --- DDB[("Delivery DB")]:::db
    BR{{"Message broker"}}:::mq
    O <-->|"events"| BR
    R <-->|"events"| BR
    D <-->|"events"| BR

    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mq fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
```

| Strengths | Weaknesses |
|---|---|
| Deploy and scale each service on its own | Every call is a network call: latency, timeouts, partial failure |
| A failure can be contained to one service | No cross-service transactions: you accept eventual consistency |
| Teams own services end to end and choose their stack | Much more to run: gateway, discovery, config, tracing, many pipelines |
| Clear, enforced boundaries | Debugging and testing span several processes |

---

## Side-by-side comparison

| Concern | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deployment | One unit | One unit | Per service |
| Scaling | Whole app | Whole app | Per service |
| Data | One schema, shared | Schema per module, one server | Database per service |
| Transactions | One ACID transaction | One ACID transaction (within the same database) | Sagas, eventual consistency |
| Failure isolation | None | Little | Strong, if designed for it |
| Latency between parts | In-process | In-process | Network |
| Team independence | Low | Medium | High |
| Operational cost | Low | Low | High |
| Boundary enforcement | Convention only | Code structure and review | The network itself |
| Technology choice | One stack | One stack | Per service |

---

## How one request differs

The same use case, "customer places an order", in both styles.

**Monolith / modular monolith.** One process, plain method calls, one database transaction. Either everything commits or nothing does.

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer
    participant API as Orders controller
    participant OS as OrdersService
    participant RS as RestaurantsService
    participant DB as Database

    C->>API: POST /orders
    API->>OS: placeOrder(dto)
    rect rgba(34,197,94,0.15)
        Note over OS,DB: One database transaction
        OS->>RS: checkAvailability(items)
        RS->>DB: read menu and stock
        DB-->>RS: items
        RS-->>OS: available
        OS->>DB: insert order and order items
        OS->>DB: reserve stock
    end
    OS-->>API: order
    API-->>C: 201 Created with the confirmed order
```

**Microservices.** The Orders service only owns orders. It cannot read the Restaurants database, so it stores the order as `PENDING`, publishes an event, and finishes the flow asynchronously. The client sees the result later.

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer
    participant GW as API Gateway
    participant OS as Orders service
    participant ODB as Orders DB
    participant BR as Message broker
    participant RS as Restaurants service
    participant DS as Delivery service

    C->>GW: POST /orders
    GW->>OS: POST /orders
    OS->>ODB: insert order (status PENDING)
    OS->>BR: publish order.placed
    OS-->>GW: 202 Accepted (order id, PENDING)
    GW-->>C: 202 Accepted

    BR->>RS: order.placed
    alt restaurant accepts
        RS->>BR: publish order.confirmed
        BR->>OS: order.confirmed
        OS->>ODB: update status CONFIRMED
        Note over BR,DS: later: order.ready, then delivery
        BR->>DS: order.ready
    else restaurant rejects
        RS->>BR: publish order.rejected
        BR->>OS: order.rejected
        OS->>ODB: update status CANCELLED
    end
```

What changed:

| | Monolith | Microservices |
|---|---|---|
| Consistency | Immediate | Eventual: the order is `PENDING` before it is confirmed |
| Failure handling | Roll back the transaction | Compensating actions (cancel the order) |
| Response | `201` with the final result | `202` with a pending result; status via polling or notification |
| Coupling | Compile-time | Event contract (names and payload shape) |

---

## Scaling

A monolith scales by running more copies of everything. Microservices scale only the part under load.

```mermaid
flowchart LR
    subgraph M["Monolith: scale the whole app"]
        direction TB
        LB1{{"Load balancer"}}:::gw
        A1["App copy 1<br/>orders, restaurants, delivery"]:::svc
        A2["App copy 2<br/>orders, restaurants, delivery"]:::svc
        A3["App copy 3<br/>orders, restaurants, delivery"]:::svc
        LB1 --> A1 & A2 & A3
    end
    subgraph S["Microservices: scale the hot service"]
        direction TB
        LB2{{"Load balancer"}}:::gw
        O1["Orders 1"]:::hot
        O2["Orders 2"]:::hot
        O3["Orders 3"]:::hot
        R1["Restaurants 1"]:::svc
        D1["Delivery 1"]:::svc
        LB2 --> O1 & O2 & O3 & R1 & D1
    end
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef hot fill:#fdba74,stroke:#c2410c,color:#0f172a
```

---

## Choosing an architecture

```mermaid
flowchart TD
    S(["New or existing system?"]):::entry
    S --> Q1{"Is the domain well understood<br/>and the boundaries stable?"}
    Q1 -->|"No"| MONO["Start with a modular monolith.<br/>Boundaries are cheap to move inside one codebase."]:::good
    Q1 -->|"Yes"| Q2{"Do several teams need to release<br/>independently?"}
    Q2 -->|"No"| MONO
    Q2 -->|"Yes"| Q3{"Do parts have very different<br/>scaling, availability or tech needs?"}
    Q3 -->|"No"| MONO2["Modular monolith is probably enough.<br/>Extract a service only when a real pain appears."]:::good
    Q3 -->|"Yes"| Q4{"Can you run gateway, discovery,<br/>config, tracing and CI per service?"}
    Q4 -->|"No"| WAIT["Invest in that platform first,<br/>or extract only one service."]:::warn
    Q4 -->|"Yes"| MS["Microservices fit.<br/>Cut along business boundaries."]:::done
    classDef entry fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef warn fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef done fill:#c4b5fd,stroke:#6d28d9,color:#0f172a
```

A useful default: **start modular, extract when it hurts.** Moving a boundary inside a monolith is a refactor; moving it between services is a migration.

---

## Migrating step by step

The safe path is incremental (the "strangler fig" approach): put a router in front, move one capability at a time, and leave the rest where it is.

```mermaid
flowchart LR
    subgraph P1["1. Monolith"]
        M1["Everything in one app"]:::bad
    end
    subgraph P2["2. Modular monolith"]
        M2["Modules with public APIs,<br/>schema per module"]:::mid
    end
    subgraph P3["3. Extract one module"]
        direction TB
        RT{{"Gateway or proxy"}}:::gw
        NEW["Orders service<br/>(new)"]:::good
        OLD["Remaining monolith"]:::mid
        RT -->|"/orders"| NEW
        RT -->|"everything else"| OLD
    end
    subgraph P4["4. Repeat"]
        M4["Extract the next module,<br/>shrink the monolith"]:::good
    end
    P1 --> P2 --> P3 --> P4
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mid fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef gw fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
```

Per extraction:

1. Pick a module that already has a clean public API and its own tables.
2. Move its tables to its own database and replace cross-module reads with API calls or events.
3. Deploy it as a service and route its paths to it through the gateway.
4. Delete the old code from the monolith once traffic is stable.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Splitting into services before the domain is understood | Boundaries are guessed wrong and every change crosses services | Start as a modular monolith and extract once boundaries are stable |
| Services that share one database | Schema changes break other services; this is a distributed monolith in disguise | One database (or at least one schema with exclusive ownership) per service |
| Long chains of synchronous calls (A calls B calls C calls D) | Latency adds up and one slow service stalls the chain | Prefer events for follow-up work; keep sync calls to one hop where possible |
| Services that must always be deployed together | You pay the cost of microservices without the independence | Re-draw the boundary, or merge them |
| Too many tiny services ("nano-services") | More network calls and pipelines than the team can run | Size a service by business capability and team ownership |
| Treating the move as a rewrite | Long freeze, high risk | Strangle incrementally, one capability at a time |

```mermaid
flowchart LR
    C["Client"]:::client --> A["Service A"]:::bad
    A -->|"sync"| B["Service B"]:::bad
    B -->|"sync"| X["Service C"]:::bad
    X -->|"sync"| Y["Service D"]:::bad
    A & B & X & Y --> DB[("One shared database")]:::db
    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef db fill:#fca5a5,stroke:#7f1d1d,color:#0f172a
```

The picture above is the **distributed monolith**: separate deployables that share a database and call each other in a chain, so they behave like one system that is harder to run.
