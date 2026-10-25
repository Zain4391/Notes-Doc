# Configuration Management

How configuration and secrets are handled across many services and environments: keeping config out of the code, layering and precedence, where each kind of value lives, centralized config, injecting secrets, and validating everything at startup in NestJS.

**Contents**

1. [The problem](#the-problem)
2. [Principles](#principles)
3. [Layers and precedence](#layers-and-precedence)
4. [Configuration vs secrets](#configuration-vs-secrets)
5. [Where values live](#where-values-live)
6. [Injecting secrets](#injecting-secrets)
7. [Centralized configuration](#centralized-configuration)
8. [Per-service configuration](#per-service-configuration)
9. [NestJS: validated configuration](#nestjs-validated-configuration)
10. [Common mistakes](#common-mistakes)

Related: [Service Discovery](service-discovery.md) (addresses as configuration) · [Reference Architecture](reference-architecture.md) · [TypeORM + Supabase](../Nest/typeorm-config.md) (database config) · [Configuration & Environments](../Dotnet/configuration-env.md) and [Options Pattern](../Dotnet/options-pattern.md) (the same ideas in ASP.NET Core)

---

## The problem

Every service needs its own settings (database URL, broker URL, keys, URLs of other services), and every one of them runs in several environments.

```mermaid
flowchart LR
    subgraph SVCS["Services"]
        O["Orders"]:::svc
        R["Restaurants"]:::svc
        D["Delivery"]:::svc
        I["Identity"]:::svc
    end
    subgraph ENVS["Environments"]
        L["Local"]:::env
        T["Test / CI"]:::env
        S["Staging"]:::env
        P["Production"]:::env
    end
    SVCS ---|"4 services x 4 environments<br/>= 16 sets of values"| ENVS
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef env fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
```

Hand-edited files and values copied between servers do not scale. The aim is one **built artifact** per service, configured from the outside.

---

## Principles

| Principle | Meaning |
|---|---|
| **Config lives outside the code** | Nothing environment-specific is compiled into the image or committed to git |
| **Build once, configure many** | The same container image is promoted from staging to production; only the injected values differ |
| **Secrets are different** | Passwords and keys have stricter storage, access and rotation than ordinary settings |
| **Fail fast** | A service with missing or malformed config refuses to start |
| **Each service owns its config** | No single `.env` shared by every service |

```mermaid
flowchart LR
    SRC["Source code"]:::src --> CI["CI build"]:::ci
    CI --> IMG[("One container image<br/>orders:1.4.2")]:::img
    IMG --> D1["Staging<br/>staging config + secrets"]:::env
    IMG --> D2["Production<br/>production config + secrets"]:::env
    CFG1["Staging values"]:::cfg -.-> D1
    CFG2["Production values"]:::cfg -.-> D2
    classDef src fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef ci fill:#fde68a,stroke:#b45309,color:#0f172a
    classDef img fill:#e9d5ff,stroke:#7e22ce,color:#0f172a
    classDef env fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef cfg fill:#fecaca,stroke:#b91c1c,color:#0f172a
```

---

## Layers and precedence

Values come from several layers; a later layer overrides an earlier one. Keep the layers few and the order consistent across services.

```mermaid
flowchart TB
    L1["1. Defaults in code<br/>safe values for local development"]:::l1
    L2["2. Config file per environment<br/>.env.local, .env.staging (no secrets in git)"]:::l2
    L3["3. Environment variables<br/>set by Compose, Kubernetes or the host"]:::l3
    L4["4. Secret store<br/>injected at runtime"]:::l4
    FINAL(["Effective configuration"]):::final
    L1 --> L2 --> L3 --> L4 --> FINAL
    classDef l1 fill:#e5e7eb,stroke:#4b5563,color:#0f172a
    classDef l2 fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef l3 fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef l4 fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef final fill:#c4b5fd,stroke:#6d28d9,color:#0f172a
```

| Layer | Typical content | Committed to git? |
|---|---|---|
| Defaults | Ports, timeouts, log level for dev | Yes |
| Environment file | Non-secret settings per environment | Only non-secret templates (`.env.example`) |
| Environment variables | Everything that changes per deployment | No, set by the platform |
| Secret store | Passwords, API keys, signing secrets | Never |

In NestJS, `ConfigModule` reads `process.env` (plus an optional env file) and `process.env` wins over the file. The same layering in ASP.NET Core is described in [Configuration & Environments](../Dotnet/configuration-env.md).

---

## Configuration vs secrets

| | Configuration | Secrets |
|---|---|---|
| Examples | `PORT`, `LOG_LEVEL`, `RABBITMQ_EXCHANGE`, `ORDERS_URL` | `DATABASE_URL` with password, `JWT_CUSTOMER_SECRET`, API keys |
| If leaked | Harmless or mildly embarrassing | Breach |
| Storage | Env vars, ConfigMaps, files | Secret manager, Kubernetes Secrets, encrypted store |
| Access | Anyone on the team | Least privilege, audited |
| Rotation | Rare | Planned and automated |
| In logs | Fine | Never |

A connection string with a password is a **secret as a whole**, even though the host part is not.

---

## Where values live

| Platform | Configuration | Secrets |
|---|---|---|
| **Local** | `.env` files (git-ignored) | The same `.env`, with development-only values |
| **Docker Compose** | `environment:` or `env_file:` | `secrets:` or an `env_file` kept out of git |
| **Kubernetes** | ConfigMap | Secret (consider encryption at rest and an external secret manager) |
| **Cloud** | Parameter store or app settings | Cloud secret manager (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) |
| **Self-hosted** | Consul KV or files | HashiCorp Vault |

```mermaid
flowchart LR
    subgraph SOURCES["Sources of truth"]
        CM["ConfigMap or env file<br/>non-secret settings"]:::cfg
        SM[("Secret manager<br/>Vault or cloud")]:::sec
    end
    subgraph RUN["At runtime"]
        POD["Orders container<br/>process.env.*"]:::svc
    end
    CM -->|"environment variables"| POD
    SM -->|"environment variables or mounted file"| POD
    classDef cfg fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef sec fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef svc fill:#bbf7d0,stroke:#15803d,color:#0f172a
```

---

## Injecting secrets

Secrets reach the container **at deploy or start time**, never at build time.

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Git as Git repository
    participant CI as CI/CD pipeline
    participant SM as Secret manager
    participant PL as Platform
    participant App as Orders container

    Dev->>Git: push code (no secrets)
    Git->>CI: trigger build
    CI->>CI: build and publish image (no secrets inside)
    CI->>PL: deploy image tag
    PL->>SM: authenticate as the workload and request orders secrets
    SM-->>PL: secret values
    PL->>App: start container with env vars or mounted files
    App->>App: validate config, fail fast if invalid
    App-->>PL: ready
```

| Rule | Reason |
|---|---|
| No secret in the image, `Dockerfile` (`ENV`, `ARG`) or git history | Anyone with the image or repo can read it |
| The workload authenticates to the secret manager with its own identity | No shared long-lived admin credential |
| Different secrets per environment and per service | A leak in staging cannot reach production; one service cannot read another's keys |
| Rotate on a schedule and after any suspected exposure | Limits the lifetime of a leaked value |

---

## Centralized configuration

Some systems add a **configuration service** so values can change without redeploying every instance.

```mermaid
sequenceDiagram
    autonumber
    participant Admin as Operator
    participant CS as Config store or server
    participant O as Orders instance 1
    participant O2 as Orders instance 2

    O->>CS: fetch config for "orders" in "production" (at startup)
    CS-->>O: values (version 7)
    O2->>CS: fetch config
    CS-->>O2: values (version 7)

    Admin->>CS: change a value (version 8)
    CS-->>O: change notification, or O polls and sees version 8
    O->>O: reload the changed setting
    CS-->>O2: change notification
    O2->>O2: reload the changed setting
```

| Option | Good for | Cost |
|---|---|---|
| **Env vars and restart** | Most settings | A rolling restart per change; simple and predictable |
| **Central config store** (Consul, cloud parameter stores) | Many services sharing values, frequent changes | Another dependency that must be highly available |
| **Feature flags** | Turning behavior on or off at runtime | A flag service and discipline to remove old flags |

Start with environment variables and a rolling restart. Add a central store only when the cost of restarts or duplicated values is real. If the store is unreachable, a service should start from its last known values or defaults instead of failing completely.

---

## Per-service configuration

Each service defines and validates **its own** settings. A shared `.env` for everything makes every service depend on every other service's variables.

```mermaid
flowchart TB
    subgraph SHARED["Avoid: one shared file"]
        direction LR
        ALL[".env<br/>DATABASE_URL, ORDERS_DB, DRIVERS_DB,<br/>RABBITMQ_URL, JWT_CUSTOMER_SECRET,<br/>JWT_DRIVER_SECRET, STRIPE_KEY, ..."]:::bad
        SA["Orders"]:::svc
        SB["Delivery"]:::svc
        ALL --> SA
        ALL --> SB
    end
    subgraph OWN["Prefer: each service owns its config"]
        direction LR
        OC["orders config<br/>ORDERS_DATABASE_URL<br/>RABBITMQ_URL<br/>JWT_CUSTOMER_SECRET"]:::good --> OS["Orders"]:::svc
        DC["delivery config<br/>DELIVERY_DATABASE_URL<br/>RABBITMQ_URL<br/>JWT_DRIVER_SECRET"]:::good --> DS["Delivery"]:::svc
    end
    classDef bad fill:#fecaca,stroke:#b91c1c,color:#0f172a
    classDef good fill:#bbf7d0,stroke:#15803d,color:#0f172a
    classDef svc fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
```

| Variable | Orders | Restaurants | Delivery | Gateway |
|---|:---:|:---:|:---:|:---:|
| Own database URL | yes | yes | yes | no |
| `RABBITMQ_URL` and exchange | yes | yes | yes | no |
| Queue name for this service | yes | yes | yes | no |
| URLs of services it calls | restaurants | none | none | all |
| Token verification key or secret | yes | yes | yes | yes |
| Signing secret for issuing tokens | no | no | no | no (Identity service only) |

The principle behind the table: **a service only receives the values it needs.**

---

## NestJS: validated configuration

`@nestjs/config` loads the environment; a validation schema makes startup fail when something is missing or wrong.

```bash
npm install @nestjs/config joi
```

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ['.env.local', '.env'],
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'test', 'production').default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().uri().required(),
        RABBITMQ_URL: Joi.string().uri().required(),
        RABBITMQ_EXCHANGE: Joi.string().required(),
        RESTAURANTS_URL: Joi.string().uri().required(),
        JWT_CUSTOMER_SECRET: Joi.string().min(32).required(),
      }),
      validationOptions: { abortEarly: false },
    }),
  ],
})
export class AppModule {}
```

| Option | Effect |
|---|---|
| `isGlobal: true` | `ConfigService` is injectable everywhere without importing `ConfigModule` again |
| `envFilePath` | Env files to load; real environment variables still take precedence |
| `validationSchema` | The app **fails at startup** if a required value is missing or invalid |
| `abortEarly: false` | Reports every problem at once, not just the first |

Read values through `ConfigService`, using `getOrThrow` so a missing value is an error rather than `undefined`:

```ts
constructor(private readonly config: ConfigService) {
  const url = this.config.getOrThrow<string>('RESTAURANTS_URL');
}
```

For grouped, typed settings use `registerAs` (as in [TypeORM + Supabase](../Nest/typeorm-config.md)). This is the Nest counterpart of ASP.NET Core's options pattern with `ValidateOnStart()` described in [Options Pattern](../Dotnet/options-pattern.md).

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Committing `.env` or secrets | They stay in git history even after deletion | Git-ignore `.env`; commit only `.env.example` and rotate anything that leaked |
| Secrets in the image (`ENV`, `ARG`, copied files) | Anyone who can pull the image can read them | Inject at runtime from a secret store |
| Fallback secrets in code that reach production | Anyone who reads the repo can forge tokens | No default for secrets; make them required |
| One shared `.env` for every service | Hidden coupling, over-broad access | Per-service config with only the values needed |
| Different variable names for the same thing across services | Confusion and bugs | Agree on names (`RABBITMQ_URL`) and document them |
| No validation at startup | Misconfiguration shows up as a runtime error under load | Validation schema with required values |
| Same secrets in staging and production | A staging leak is a production leak | Separate secrets per environment |
| Logging the full config object | Secrets end up in logs | Log only non-sensitive keys, or redact |
| Changing a value without restarting or reloading | The running instances keep the old value | Rolling restart, or an explicit reload mechanism |
