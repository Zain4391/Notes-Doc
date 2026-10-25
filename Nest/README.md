# NestJS

Practical reference notes for building NestJS back ends with TypeORM. Each file is a self-contained reference meant to be kept open next to the code while writing it: what a thing is, how to set it up, and the mistakes to avoid.

Examples use **TypeORM** with **PostgreSQL** (hosted on Supabase), `class-validator` for DTOs, Passport for JWT authentication, and RabbitMQ for events.

---

## Topics

| Note | Covers |
|---|---|
| [NestJS Core Concepts](nest-core.md) | Short overview of modules, controllers, providers, exports, the root module and global modules |
| [NestJS Modules](nest-module.md) | `@Module()` properties, custom provider types (`useClass`, `useValue`, `useFactory`, `useExisting`), `@Inject()` tokens, `@Global()`, dynamic modules, module patterns |
| [NestJS Controllers](controller.md) | Routing decorators, `@Param`/`@Query`/`@Body`, status codes, headers, guards, pipes, `ValidationPipe`, `class-validator` decorators |
| [NestJS Service Layer](service.md) | Services and DI, TypeORM CRUD, DTOs, pagination (manual and `nestjs-typeorm-paginate`), sorting, filtering, QueryBuilder, transactions, caching, logging, testing |
| [TypeORM + Supabase Configuration](typeorm-config.md) | Packages, app config vs CLI `DataSource`, entities, migrations scripts |
| [Custom Exceptions and Global Exception Filter](custom-exception.md) | Base and domain exceptions, the global filter (`@RestControllerAdvice` equivalent), usage in services, strategies, guards and controllers, example error responses |
| [JWT Authentication and Role-Based Access Control](jwt-authentication.md) | Passport JWT strategies for two user types, bcrypt, signing tokens, response DTOs, then RBAC with `@Roles()`, `RolesGuard` and permissions |
| [RabbitMQ: Configuration and Usage](rabbitmq-config-use.md) | Topic exchange, per-service queues, publisher service, event DTOs, `@EventPattern` listeners |

## Suggested reading order

Setting up a new project from scratch:

1. [NestJS Core Concepts](nest-core.md): the vocabulary used everywhere else.
2. [NestJS Modules](nest-module.md): how code is organized and shared.
3. [TypeORM + Supabase Configuration](typeorm-config.md): connect the app to the database.
4. [NestJS Controllers](controller.md): routes, input and validation.
5. [NestJS Service Layer](service.md): business logic, queries and pagination.
6. [Custom Exceptions and Global Exception Filter](custom-exception.md): consistent error responses.
7. [JWT Authentication and Role-Based Access Control](jwt-authentication.md): protect the API.
8. [RabbitMQ: Configuration and Usage](rabbitmq-config-use.md): events between modules.

## Quick lookup

| I need to... | Go to |
|---|---|
| Share a provider with another module | [Exports](nest-core.md#exports) |
| Fix "Nest can't resolve dependencies" | [Common mistakes](nest-core.md#common-mistakes) |
| Register a provider built from config or async code | [useFactory](nest-module.md#usefactory) |
| Inject a value by string or symbol token | [Injecting with `@Inject()`](nest-module.md#injecting-with-inject) |
| Make a module available everywhere | [@Global](nest-module.md#global) |
| Read route, query or body values | [Route parameters](controller.md#route-parameters) |
| Convert `:id` to a number or validate a UUID | [Pipes](controller.md#pipes) |
| Validate a request body | [Validation with ValidationPipe](controller.md#validation-with-validationpipe) |
| Add search, sorting and pagination to a list endpoint | [Customers service](service.md#customers-service) |
| Use `nestjs-typeorm-paginate` | [Pagination with nestjs-typeorm-paginate](service.md#pagination-with-nestjs-typeorm-paginate) |
| Wrap several writes in a transaction | [Service best practices](service.md#service-best-practices) |
| Understand the alias in `createQueryBuilder('x')` | [QueryBuilder aliases vs table names](service.md#querybuilder-aliases-vs-table-names) |
| Set up the database connection | [Config files](typeorm-config.md#config-files) |
| Generate and run a migration | [Migrations](typeorm-config.md#migrations) |
| Return one error format from the whole API | [Global exception filter](custom-exception.md#global-exception-filter) |
| Throw a domain-specific error | [Creating exceptions](custom-exception.md#creating-exceptions) |
| Protect a route with a JWT | [JWT strategies and guards](jwt-authentication.md#jwt-strategies-and-guards) |
| Restrict a route to certain roles | [Roles guard](jwt-authentication.md#roles-guard) |
| Hide the password hash in responses | [Types and response DTOs](jwt-authentication.md#types-and-response-dtos) |
| Emit an event to another module | [Emitting events](rabbitmq-config-use.md#emitting-events) |
| Handle an event in a controller | [Listening to events](rabbitmq-config-use.md#listening-to-events) |
