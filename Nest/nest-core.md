# NestJS Core Concepts

The building blocks of a NestJS application: modules, controllers, providers, exports, the root module and global modules, with a minimal example of each.

**Contents**

1. [Overview](#overview)
2. [Modules](#modules)
3. [Controllers](#controllers)
4. [Providers](#providers)
5. [Exports](#exports)
6. [Registration and bootstrap](#registration-and-bootstrap)
7. [Global modules](#global-modules)
8. [Quick reference](#quick-reference)
9. [Common mistakes](#common-mistakes)

Related: [Modules](nest-module.md) (the `@Module()` decorator in depth) · [Controllers](controller.md) · [Services](service.md)

---

## Overview

| Building block | Role | Decorator |
|----------------|------|-----------|
| **Module** | Organizational boundary; groups related classes | `@Module()` |
| **Controller** | Receives requests, defines routes, delegates work | `@Controller()` |
| **Provider** | Encapsulates business logic; injectable | `@Injectable()` |
| **Export** | Makes a provider usable by other modules | `exports: []` |

How they fit together:

```text
AppModule (root)
├── imports ──► BooksModule
│                ├── controllers: BooksController ──► calls ──► BooksService
│                ├── providers:   BooksService
│                └── exports:     BooksService
└── imports ──► OrdersModule (can inject BooksService because it is exported)
```

Read the tree top-down: `AppModule` imports `BooksModule`, whose controller calls its own service. `BooksService` is exported, so `OrdersModule` (also imported by `AppModule`) can inject it. If the export were missing, the same injection would fail with a dependency-resolution error.

---

## Modules

A module is an organizational boundary and a unit of composition. Its metadata declares which controllers and providers it contains, which providers it shares, and which other modules it consumes.

```ts
@Module({
  controllers: [BooksController],
  providers: [BooksService],
  exports: [BooksService],
})
export class BooksModule {}
```

**How it works**

- `controllers` registers `BooksController`, so its routes become part of the application.
- `providers` tells Nest's DI container to create one `BooksService` and keep it available inside this module.
- `exports` makes that provider usable by any module that imports `BooksModule`. Without it, `BooksService` stays private to this module.

### `@Module()` metadata

| Property | Description |
|----------|-------------|
| `controllers` | Controllers that belong to this module. |
| `providers` | Providers instantiated by Nest's DI container and available inside this module. |
| `imports` | Other modules whose **exported** providers this module needs. |
| `exports` | Providers (or modules) this module makes available to modules that import it. |

**Key points**

- Modules group related classes and make them discoverable by Nest's DI container.
- Use `imports` to consume exported providers from other modules.

---

## Controllers

Controllers handle incoming requests, define routes, and delegate work to providers (services).

```ts
@Controller('books')
export class BooksController {
  constructor(private readonly booksService: BooksService) {}

  @Get()
  findAll() {
    return this.booksService.findAll();
  }
}
```

| Element | Description |
|---------|-------------|
| `@Controller('books')` | Marks the class as a controller; `'books'` is the route prefix. |
| `constructor(private readonly booksService: BooksService)` | Constructor injection of a provider. |
| `@Get()`, `@Post()`, ... | Route handler decorators, one per HTTP method. |

**What controllers do**

- Handle HTTP requests.
- Define route handlers.
- Call providers to perform the business logic (keep controllers thin).

---

## Providers

Providers are classes that encapsulate business logic. They are managed by Nest's Dependency Injection (DI) container.

```ts
@Injectable()
export class BooksService {
  findAll() {
    return [];
  }
}
```

**Notes**

- Providers are **singletons** by default (one instance per application).
- Other scopes exist when needed: `REQUEST`-scoped (one per request) and `TRANSIENT` (one per consumer).
- "Provider" is a broad term: services, repositories, factories, guards, interceptors, pipes and so on.

| Scope | Instance lifetime |
|-------|-------------------|
| `DEFAULT` | One shared instance for the whole application. |
| `REQUEST` | New instance for every incoming request. |
| `TRANSIENT` | New instance for every class that injects it. |

---

## Exports

Exports make providers from one module available to other modules that import it.

```ts
@Module({
  providers: [BooksService],
  exports: [BooksService],
})
export class BooksModule {}

@Module({
  imports: [BooksModule],
})
export class OrdersModule {}
```

| Case | Result |
|------|--------|
| `BooksService` is exported | `OrdersModule` can inject it. |
| `BooksService` is **not** exported | It stays private to `BooksModule`; injecting it elsewhere throws a DI error. |

---

## Registration and bootstrap

Modules must be registered so Nest includes them in the application context. The root module (commonly `AppModule`) composes the application.

```ts
@Module({
  imports: [BooksModule, UsersModule],
})
export class AppModule {}

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
```

When the application boots, Nest builds a DI context from the registered modules. Those modules, and their exported providers, become reachable inside that context.

---

## Global modules

`@Global()` makes a module's exported providers available throughout the application, without importing the module in every feature module.

```ts
@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

- The module must still be imported **once** (typically in the root module) so Nest registers it.
- After that, its exported providers can be injected anywhere.
- Use sparingly: explicit `imports` keep dependencies visible. Good fits are config, logging and database modules.

---

## Quick reference

| I want to... | Do this |
|--------------|---------|
| Group related code | Create a `@Module()` |
| Handle an HTTP route | Add a method with `@Get()` / `@Post()` ... in a `@Controller()` |
| Put business logic somewhere injectable | `@Injectable()` class listed in `providers` |
| Use a provider from another module | Export it there, add that module to `imports` here |
| Use a provider everywhere | Mark its module `@Global()` and import it once in `AppModule` |

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Injecting a provider that its module does not export | Nest throws "can't resolve dependencies" | Add it to `exports` in the owning module and `imports` the module where it is used |
| Provider missing from `providers` | Nothing registers it, so it cannot be injected | List it in the module's `providers` |
| Module never imported by any module reachable from `AppModule` | Its controllers and providers are never registered | Add it to `imports` of `AppModule` (or of a module that `AppModule` imports) |
| `@Global()` module not imported anywhere | The decorator makes exports global, but Nest still has to register the module | Import it once, in the root module |
| Business logic inside controllers | Hard to reuse and to test | Move it into a provider (service) and inject that |
