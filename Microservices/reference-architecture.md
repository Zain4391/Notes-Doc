# Reference Architecture

One end-to-end picture of a microservice system built from the pieces in the other notes: the full topology, the synchronous and asynchronous paths, a complete order lifecycle, failure behavior, deployment and observability, plus a suggested order for building it.

The example is the food-delivery domain used throughout the NestJS notes (customers, restaurants, orders, drivers), including the event names from [RabbitMQ: Configuration and Usage](../Nest/rabbitmq-config-use.md).

**Contents**

1. [The whole system](#the-whole-system)
2. [Two paths: request and event](#two-paths-request-and-event)
3. [An order from start to finish](#an-order-from-start-to-finish)
4. [Order state machine](#order-state-machine)
5. [When a service is down](#when-a-service-is-down)
6. [Deployment view](#deployment-view)
7. [Observability](#observability)
8. [Where each concern is handled](#where-each-concern-is-handled)
9. [Build order](#build-order)
10. [Common mistakes](#common-mistakes)

Related: [Monolith vs Modular Monolith vs Microservices](monolith-vs-microservices.md) · [Service Boundaries](service-boundaries.md) · [API Gateway](api-gateway.md) · [Service Discovery](service-discovery.md) · [Configuration Management](configuration-management.md)

---

## The whole system

```mermaid
flowchart TB
    subgraph CL["Clients"]
        WEB["Web app"]:::client
        MOB["Mobile app"]:::client
        ADM["Admin portal"]:::client
    end

    LB{{"Load balancer<br/>TLS termination"}}:::lb
    GW["API Gateway / BFF<br/>authentication, rate limiting,<br/>routing, aggregation"]:::gw

    subgraph SV["Services: each owns its logic and its data"]
        IDS["Identity<br/>accounts, JWT, roles"]:::svc
        ORD["Orders<br/>cart, order lifecycle"]:::svc
        RES["Restaurants<br/>menus, preparation"]:::svc
        DEL["Delivery<br/>drivers, dispatch"]:::svc
        NOT["Notifications<br/>email, SMS, push"]:::svc
    end

    subgraph DATA["Databases: one per service"]
        IDB[("Identity DB")]:::db
        ODB[("Orders DB")]:::db
        RDB[("Restaurants DB")]:::db
        DDB[("Delivery DB")]:::db
    end

    MQ{{"RabbitMQ<br/>topic exchange: food-delivery-exchange<br/>one queue per service"}}:::mq

    subgraph PLAT["Platform"]
        REG[("Service discovery<br/>DNS or registry")]:::infra
        CFG[("Configuration<br/>and secrets")]:::infra
    end

    subgraph OBS["Observability"]
        LOG["Logs"]:::obs
        MET["Metrics"]:::obs
        TRC["Traces"]:::obs
        DASH["Dashboards and alerts"]:::obs
        LOG & MET & TRC --> DASH
    end

    WEB & MOB & ADM --> LB --> GW
    GW --> IDS & ORD & RES & DEL

    IDS --- IDB
    ORD --- ODB
    RES --- RDB
    DEL --- DDB

    ORD & RES & DEL & NOT <-->|"events"| MQ

    SV -.->|"resolve names, load config"| PLAT
    SV -.->|"logs, metrics, traces"| OBS

    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef lb fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mq fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
    classDef infra fill:#e5e7eb,stroke:#4b5563,color:#0f172a
    classDef obs fill:#fed7aa,stroke:#c2410c,color:#0f172a
```

| Piece | Note |
|---|---|
| Gateway | [API Gateway](api-gateway.md) |
| Service split and data ownership | [Service Boundaries](service-boundaries.md) |
| Discovery and health checks | [Service Discovery](service-discovery.md) |
| Config and secrets | [Configuration Management](configuration-management.md) |
| Broker, queues and events | [RabbitMQ: Configuration and Usage](../Nest/rabbitmq-config-use.md) |
| Authentication and roles | [JWT Authentication](../Nest/jwt-authentication.md) |
| Error format | [Custom Exceptions](../Nest/custom-exception.md) |

---

## Two paths: request and event

Traffic takes one of two routes, and choosing the right one for each interaction is most of the design work.

```mermaid
flowchart LR
    subgraph SYNC["Synchronous path: the caller waits"]
        direction LR
        C1["Client"]:::client -->|"HTTP"| G1["Gateway"]:::gw -->|"HTTP"| S1["Service"]:::svc
        S1 -->|"response"| G1
    end
    subgraph ASYNC["Asynchronous path: fire and continue"]
        direction LR
        P["Producer service"]:::svc -->|"publish event"| B{{"Broker"}}:::mq
        B -->|"deliver to queue"| Q1["Consumer A"]:::svc
        B -->|"deliver to queue"| Q2["Consumer B"]:::svc
    end
    classDef client fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef mq fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
```

| Use the synchronous path when | Use the asynchronous path when |
|---|---|
| The caller needs the answer now (read a profile, validate a token) | The work can finish later (notify, assign a driver, update a read model) |
| It is a query | It announces something that happened (a fact) |
| One service is asked and no one else needs to know | Several services may react independently |
| A short timeout is acceptable | You want the producer to keep working if a consumer is down |

---

## An order from start to finish

The full lifecycle with the event names and queue bindings from the RabbitMQ note (`ORDERS` queue: `order.confirmed`, `driver.assigned`, `order.delivered`; `RESTAURANTS`: `order.placed`; `DELIVERY`: `order.ready`, `order.picked.up`).

```mermaid
sequenceDiagram
    autonumber
    actor C as Customer
    participant GW as API Gateway
    participant OS as Orders service
    participant MQ as RabbitMQ exchange
    participant RS as Restaurants service
    participant DS as Delivery service
    participant NS as Notifications service

    C->>GW: POST /orders (Bearer token)
    GW->>GW: verify token, rate limit
    GW->>OS: POST /orders (X-User-Id)
    OS->>OS: save order as PENDING
    OS->>MQ: emit order.placed
    OS-->>GW: 202 Accepted (orderId)
    GW-->>C: 202 Accepted

    MQ->>RS: order.placed (restaurants queue)
    RS->>RS: accept and start preparing
    RS->>MQ: emit order.confirmed
    MQ->>OS: order.confirmed (orders queue)
    OS->>OS: status CONFIRMED

    Note over RS: food is prepared
    RS->>MQ: emit order.ready
    MQ->>DS: order.ready (delivery queue)
    DS->>DS: choose an available driver
    DS->>MQ: emit driver.assigned
    MQ->>OS: driver.assigned (orders queue)
    OS->>OS: attach driver, status ASSIGNED

    DS->>MQ: emit order.picked.up
    DS->>MQ: emit order.delivered
    MQ->>OS: order.delivered (orders queue)
    OS->>OS: status DELIVERED
    OS->>NS: customer notified
```

Points to notice:

| Point | Detail |
|---|---|
| The client gets `202`, not the final result | The order is `PENDING`; the client polls or receives a notification |
| Each service reacts only to the events it is bound to | Bindings live in `QUEUE_CONFIGS` |
| Nothing calls another service's database | Orders learns about a driver through `driver.assigned`, not by querying Delivery |
| `order.picked.up` | In the config of the RabbitMQ note only the `DELIVERY` queue is bound to it. If Orders must track pickup, add that routing key to the `ORDERS` entry |
| Notification step | Drawn as a direct call for brevity; in practice Notifications would be bound to the same events |

---

## Order state machine

The order's status changes only in response to events or validated commands.

```mermaid
stateDiagram-v2
    [*] --> Pending: order placed
    Pending --> Confirmed: order.confirmed
    Pending --> Cancelled: restaurant rejected or timeout
    Confirmed --> Ready: order.ready
    Confirmed --> Cancelled: customer cancels or restaurant fails
    Ready --> Assigned: driver.assigned
    Assigned --> PickedUp: order.picked.up
    PickedUp --> Delivered: order.delivered
    Delivered --> [*]
    Cancelled --> [*]
```

A state machine makes **duplicate or out-of-order events** safe to handle: ignore any event that is not valid for the current state.

---

## When a service is down

Because the broker stores messages in durable queues, a consumer outage delays work instead of losing it.

```mermaid
sequenceDiagram
    autonumber
    participant OS as Orders service
    participant MQ as RabbitMQ
    participant Q as restaurants-service-queue
    participant RS as Restaurants service

    OS->>MQ: emit order.placed
    MQ->>Q: route and store (durable queue)
    Note over RS: Restaurants service is down
    Q-->>Q: message waits

    Note over OS: Orders stays PENDING

    RS->>RS: restarts and reconnects
    Q->>RS: deliver order.placed
    RS->>RS: process
    alt message handled
        RS-->>Q: ack
    else waited longer than the message TTL (24 h in the RabbitMQ note)
        Q-->>Q: message expires and is dropped
        Note over OS: order stuck as PENDING, needs a timeout job
    end
```

| Behavior | Consequence for design |
|---|---|
| Messages wait in durable queues | Short outages are invisible to producers |
| Message TTL expires old messages | Add a **timeout** that moves stale `PENDING` orders to `CANCELLED` and notifies the customer |
| `noAck: false` (manual ack) | A message is redelivered if the consumer dies mid-processing, so handlers must be **idempotent** (safe to run twice) |
| The synchronous path has no buffer | Use timeouts and a circuit breaker so callers fail fast |

---

## Deployment view

```mermaid
flowchart TB
    INT(("Internet")):::infra --> LB{{"Cloud load balancer"}}:::lb

    subgraph CLUSTER["Kubernetes cluster (or equivalent)"]
        subgraph EDGE["Edge"]
            GW1["Gateway pod 1"]:::gw
            GW2["Gateway pod 2"]:::gw
        end
        subgraph APPS["Application namespace"]
            O1["orders pod 1"]:::svc
            O2["orders pod 2"]:::svc
            R1["restaurants pod 1"]:::svc
            D1["delivery pod 1"]:::svc
            D2["delivery pod 2"]:::svc
            I1["identity pod 1"]:::svc
        end
        SVCS{{"Kubernetes Services<br/>stable names, readiness-based routing"}}:::lb
        CFGM["ConfigMaps and Secrets"]:::infra
    end

    subgraph MANAGED["Managed or stateful services"]
        PG[("PostgreSQL<br/>one database per service")]:::db
        BRK{{"RabbitMQ cluster"}}:::mq
        SM[("Secret manager")]:::infra
    end

    LB --> GW1 & GW2
    GW1 & GW2 --> SVCS
    SVCS --> O1 & O2 & R1 & D1 & D2 & I1
    APPS --> PG
    APPS <--> BRK
    SM -.-> CFGM -.-> APPS

    classDef infra fill:#e5e7eb,stroke:#4b5563,color:#0f172a
    classDef lb fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
    classDef gw fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef db fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef mq fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
```

| Layer | Runs | Scaling |
|---|---|---|
| Edge | Gateway pods | Several instances behind the load balancer |
| Applications | One deployment per service | Independently, by load |
| Data and broker | Databases, RabbitMQ | Managed service or a dedicated stateful setup, not scaled like stateless pods |
| Local development | The same containers in [Docker Compose](../Docker/docker-compose.md) | One instance of each |

---

## Observability

With many services, one user action touches many processes. You need to follow **one request across all of them**.

```mermaid
sequenceDiagram
    autonumber
    participant GW as Gateway
    participant OS as Orders service
    participant MQ as RabbitMQ
    participant RS as Restaurants service
    participant TS as Tracing backend

    Note over GW: creates trace id T1 (header: traceparent)
    GW->>OS: HTTP request with traceparent T1
    OS->>MQ: publish order.placed, trace id T1 in message headers
    MQ->>RS: deliver message with T1
    GW-->>TS: span: gateway
    OS-->>TS: span: orders handle request
    RS-->>TS: span: restaurants handle event
    Note over TS: all three spans share T1, so one timeline can be shown
```

| Pillar | Answers | Rule of thumb |
|---|---|---|
| **Logs** | What happened in this service? | Structured (JSON), always include the trace or request ID |
| **Metrics** | Is it healthy and how loaded is it? | Request rate, error rate, latency, queue depth |
| **Traces** | Where did this request spend its time? | Propagate trace context through HTTP **and** message headers |

Queue depth deserves a dashboard of its own: a growing queue means a consumer is slow or down.

---

## Where each concern is handled

| Concern | Handled by | Note |
|---|---|---|
| TLS, public entry point | Load balancer and gateway | [API Gateway](api-gateway.md) |
| Authentication | Gateway verifies; services authorize | [JWT Authentication](../Nest/jwt-authentication.md) |
| Authorization (roles) | Each service | [JWT Authentication](../Nest/jwt-authentication.md) |
| Routing and aggregation | Gateway / BFF | [API Gateway](api-gateway.md) |
| Finding other services | DNS or registry | [Service Discovery](service-discovery.md) |
| Config and secrets | Environment and secret manager | [Configuration Management](configuration-management.md) |
| Data ownership | The owning service | [Service Boundaries](service-boundaries.md) |
| Cross-service workflows | Events and sagas | [RabbitMQ: Configuration and Usage](../Nest/rabbitmq-config-use.md) |
| Error format | Global exception filter | [Custom Exceptions](../Nest/custom-exception.md) |
| Schema changes | Migrations per service | [TypeORM + Supabase](../Nest/typeorm-config.md) |
| Containers | Docker, Compose, Kubernetes | [Docker notes](../Docker/README.md) |

---

## Build order

Do not build everything at once. Add each piece when the pain it removes is real.

```mermaid
flowchart LR
    S1["1. Modular monolith<br/>modules, schema per module,<br/>CI pipeline"]:::s1
    S2["2. Platform basics<br/>containers, per-service config,<br/>health endpoints"]:::s2
    S3["3. Gateway<br/>single entry, auth at the edge"]:::s3
    S4["4. Broker and first extracted service<br/>events, own database"]:::s4
    S5["5. Observability<br/>structured logs, metrics, traces"]:::s5
    S6["6. Resilience<br/>timeouts, retries, circuit breaker,<br/>idempotency, sagas"]:::s6
    S1 --> S2 --> S3 --> S4 --> S5 --> S6
    classDef s1 fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef s2 fill:#e5e7eb,stroke:#4b5563,color:#0f172a
    classDef s3 fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef s4 fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef s5 fill:#fed7aa,stroke:#c2410c,color:#0f172a
    classDef s6 fill:#c4b5fd,stroke:#6d28d9,color:#0f172a
```

Steps 5 and 6 should not be postponed far: add basic observability **before** the second service is extracted, since debugging across services without it is slow.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Making every interaction synchronous | Failures and latency cascade through the chain | Use events for follow-up work; keep sync calls for queries that need an answer |
| Making every interaction asynchronous | Simple reads become hard to reason about | Use plain request/response for queries |
| No timeout for work stuck waiting on an event | Orders stay `PENDING` forever after a lost message | A scheduled job that expires stale states |
| Non-idempotent event handlers | Redelivery causes double effects (two drivers, two charges) | Track processed event IDs or make updates state-based |
| Services bypass the broker and call each other directly for everything | Hidden coupling and call chains | Decide per interaction which path fits |
| Adding observability last | Cross-service bugs become guesswork | Request ID and structured logs from the first service |
| Building the full platform before the product needs it | Months of infrastructure, no features | Follow the [build order](#build-order); extract only when it hurts |
