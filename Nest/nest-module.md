# NestJS Modules

The `@Module()` decorator and its properties (`imports`, `exports`, `controllers`, `providers`), custom provider types, factories, global modules, dynamic modules and common module patterns.

**Contents**

1. [The `@Module()` decorator](#the-module-decorator)
2. [imports](#imports)
3. [exports](#exports)
4. [controllers](#controllers)
5. [providers](#providers)
6. [Custom provider types](#custom-provider-types)
7. [Injecting with `@Inject()`](#injecting-with-inject)
8. [useFactory](#usefactory)
9. [@Global](#global)
10. [Dynamic modules](#dynamic-modules)
11. [Module organization patterns](#module-organization-patterns)
12. [Complete module example](#complete-module-example)
13. [Rules and takeaways](#rules-and-takeaways)
14. [Common mistakes](#common-mistakes)

Related: [NestJS Core Concepts](nest-core.md) (short overview) · [Services](service.md) · [TypeORM + Supabase](typeorm-config.md)

---

## The `@Module()` decorator

`@Module()` is the core building block of a NestJS application. Its metadata is what Nest uses to organize the application structure.

```ts
import { Module } from '@nestjs/common';

@Module({
  imports: [],
  controllers: [],
  providers: [],
  exports: [],
})
export class AppModule {}
```

| Property | Purpose |
|----------|---------|
| `imports` | Modules whose **exported** providers this module needs. |
| `controllers` | Controllers that handle HTTP requests for this module. |
| `providers` | Services, repositories, factories and helpers that can be injected. |
| `exports` | Providers (or modules) made available to modules that import this one. |

---

## imports

**Purpose:** import other modules whose providers are required in the current module.

| Point | Detail |
|-------|--------|
| Effect | The **exported** providers of the imported module become injectable here. |
| Visibility | Only exported providers are accessible; internal providers stay private to their module. |
| Graph | Builds the module dependency graph. Nest detects and warns about circular dependencies. |

```ts
@Module({
  imports: [
    TypeOrmModule,   // Database module
    ConfigModule,    // Configuration module
    UsersModule,     // Custom user module
    AuthModule,      // Authentication module
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`AppModule` pulls four modules into its scope. Everything those modules **export** can now be injected into `AppController` and `AppService`; anything they keep private cannot. The comments show the typical roles: persistence, configuration, and two feature modules. The order inside `imports` does not matter.

**Common use cases:** database modules (`TypeOrmModule`, `MongooseModule`), configuration (`ConfigModule`), shared utility modules, feature modules whose services you need, third-party NestJS modules.

---

## exports

**Purpose:** expose providers to other modules that import this module.

| Point | Detail |
|-------|--------|
| What can be exported | Providers defined in this module **and** modules imported by it (re-export). |
| Role | Forms the module's public API. |
| Principle | Export only what is necessary (least privilege). Unexported providers are private. |

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UsersRepository],  // both available inside this module
  exports: [UsersService],                      // only UsersService available to others
})
export class UsersModule {}

// In another module
@Module({
  imports: [UsersModule],                       // can now inject UsersService
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}

@Injectable()
export class OrdersService {
  constructor(private usersService: UsersService) {}     // works
  // constructor(private usersRepo: UsersRepository) {}  // fails: not exported
}
```

**How it works**

- `UsersModule` registers two providers, but exports only `UsersService`. `UsersRepository` is an internal detail.
- `OrdersModule` imports `UsersModule`, so it may inject `UsersService`. Injecting `UsersRepository` would fail with a dependency-resolution error, because it was not exported.
- This is the module's public API: other modules can depend only on what you export, so you can change the rest freely.

### Re-exporting an imported module

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule],  // modules importing UsersModule get TypeOrmModule too
})
export class UsersModule {}
```

**Common patterns:** export services other modules need, export repository modules for entities, create "barrel" modules that re-export several modules, export shared utilities.

---

## controllers

**Purpose:** declare the controllers that handle HTTP requests for this module.

| Point | Detail |
|-------|--------|
| Instantiation | Automatic, by Nest. Dependency injection works in controllers. |
| Decorators | `@Controller()` on the class; `@Get()`, `@Post()`, `@Put()`, `@Delete()` ... on methods. |
| Scope | One module holds the controllers related to its own domain. |
| Design | Keep controllers thin: delegate logic to services. |

```ts
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController, UsersAdminController],  // multiple controllers
  providers: [UsersService],
})
export class UsersModule {}
```

**Controller responsibilities**

- Define routes (endpoints).
- Validate request data with DTOs.
- Call service methods for business logic.
- Format and return responses; handle request/response transformation.
- Apply guards, interceptors and pipes.

**Best practices**

- Group related endpoints in one controller, focused on a single resource/domain.
- Use DTOs for request validation.
- Apply guards and interceptors at controller level when needed.
- Never put business logic in controllers.

---

## providers

**Purpose:** declare the services, repositories, factories and helpers that can be injected.

| Point | Detail |
|-------|--------|
| Decorator | Classes must be decorated with `@Injectable()`. |
| Managed by | Nest's IoC (Inversion of Control) container. |
| Default scope | Singleton: one instance shared across the app. |
| Injectable into | Controllers, other providers, guards, interceptors, etc. |

```ts
@Injectable()
export class UsersService {
  findAll() {
    return ['user1', 'user2'];
  }
}

@Module({
  providers: [UsersService],
  // shorthand for: providers: [{ provide: UsersService, useClass: UsersService }]
})
export class UsersModule {}
```

**How it works**

- `@Injectable()` marks the class as something Nest's container can create and inject.
- Listing `UsersService` in `providers` is what registers it. Without that line it exists as a class but nobody can inject it.
- The shorthand `providers: [UsersService]` is expanded by Nest into `{ provide: UsersService, useClass: UsersService }`: the class itself is both the **token** (what you ask for) and the **implementation** (what you get).

### Scopes

| Scope | Instance lifetime |
|-------|-------------------|
| `DEFAULT` (singleton) | One instance for the entire app. |
| `REQUEST` | New instance per request. |
| `TRANSIENT` | New instance every time it is injected. |

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}
```

Setting `scope` on `@Injectable()` changes the lifetime from the default singleton to **one new instance per incoming request**. The scope also propagates: any class that injects a request-scoped provider becomes request-scoped too, which costs performance, so use it only when the instance must hold per-request state.

---

## Custom provider types

| Type | Key | Use it for |
|------|-----|------------|
| Class | `useClass` | Standard classes, or swapping implementations. |
| Value | `useValue` | Static values and configuration objects. |
| Factory | `useFactory` | Dynamic or async creation, built from other providers. |
| Alias | `useExisting` | A second token pointing to an existing provider. |

### Class provider

```ts
providers: [
  UsersService,                                   // shorthand
  { provide: UsersService, useClass: UsersService }, // explicit
]
```

Both entries mean the same thing. The explicit form separates the **token** (`provide`) from the **implementation** (`useClass`), which lets you swap implementations, for example `{ provide: UsersService, useClass: MockUsersService }` in tests, without changing any code that injects `UsersService`.

### Value provider

```ts
providers: [
  {
    provide: 'CONFIG',
    useValue: { apiKey: '12345', apiUrl: 'https://api.example.com' },
  },
]
```

`useValue` registers a ready-made value instead of a class. The token is the string `'CONFIG'`, because a plain object has no class to use as a token. Anything that wants this value must ask for it by that exact string with `@Inject('CONFIG')` (see [Injecting with `@Inject()`](#injecting-with-inject)).

### Factory provider

```ts
providers: [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: async (configService: ConfigService) => {
      const config = configService.get('database');
      return await createConnection(config);
    },
    inject: [ConfigService],  // dependencies passed to the factory
  },
]
```

**How it works**

1. Nest first resolves everything in `inject` (here `ConfigService`) and passes it to the factory as an argument.
2. The factory reads the config and builds the connection. Because it is `async`, Nest **waits** for the promise before any provider that depends on `'DATABASE_CONNECTION'` is created.
3. The resolved connection becomes the provider's value, shared like any singleton.

### Existing (alias) provider

```ts
providers: [
  UsersService,
  {
    provide: 'USERS_SERVICE_ALIAS',
    useExisting: UsersService,  // points to the existing provider
  },
]
```

`useExisting` creates a **second token** that points at the same instance: `'USERS_SERVICE_ALIAS'` and `UsersService` resolve to one object, not two. It is useful for exposing one implementation under another name or interface token without creating a second instance.

---

## Injecting with `@Inject()`

Class providers are injected by type. Providers registered under a **string** or **symbol** token need `@Inject(token)`, with the type declared on the parameter.

### String token + interface

```ts
interface AppConfig {
  apiKey: string;
  apiUrl: string;
}

@Injectable()
export class SomeService {
  constructor(@Inject('CONFIG') private config: AppConfig) {
    console.log(this.config.apiKey);  // typed as AppConfig
  }
}
```

**How it works**

- Interfaces are erased at runtime, so Nest cannot use `AppConfig` to know what to inject. The string `'CONFIG'` is what Nest looks up, via `@Inject('CONFIG')`.
- The `: AppConfig` annotation only helps TypeScript: inside the class `this.config.apiKey` is typed and autocompleted.
- The string in `@Inject` must match `provide` exactly, and a typo only fails at startup with a "can't resolve dependencies" error.

### Value provider with a typed config

```ts
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
}

@Module({
  providers: [
    {
      provide: 'DATABASE_CONFIG',
      useValue: { host: 'localhost', port: 5432, database: 'mydb' },
    },
  ],
})
export class ConfigModule {}

@Injectable()
export class DatabaseService {
  constructor(@Inject('DATABASE_CONFIG') private dbConfig: DatabaseConfig) {
    console.log(`Connecting to ${this.dbConfig.host}:${this.dbConfig.port}`);
  }
}
```

The module registers a plain object under the token `'DATABASE_CONFIG'`, and `DatabaseService` receives it through `@Inject('DATABASE_CONFIG')`. The `DatabaseConfig` interface describes the object's shape for the compiler. Note that `ConfigModule` must **export** the token (or be imported) for another module's `DatabaseService` to resolve it.

### Factory provider injected by token

```ts
@Module({
  providers: [
    {
      provide: 'API_CLIENT',
      useFactory: (config: AppConfig) => new ApiClient(config.apiUrl),
      inject: ['CONFIG'],
    },
  ],
})
export class ApiModule {}

@Injectable()
export class ProductService {
  constructor(@Inject('API_CLIENT') private apiClient: ApiClient) {}
}
```

**How the chain resolves**

1. `ApiModule` defines `'API_CLIENT'` with a factory that needs a value registered as `'CONFIG'`.
2. `inject: ['CONFIG']` tells Nest which providers to pass into the factory's parameters, in order.
3. The factory builds an `ApiClient` from `config.apiUrl`.
4. `ProductService` asks for `'API_CLIENT'` and receives that client, typed as `ApiClient`.

`'CONFIG'` must be available in `ApiModule`'s scope (provided there or exported from an imported module).

### Symbol token (recommended for type safety)

Symbols cannot collide, unlike strings.

```ts
export const DATABASE_CONNECTION = Symbol('DATABASE_CONNECTION');

interface DatabaseConnection {
  query: (sql: string) => Promise<any>;
  close: () => Promise<void>;
}

@Module({
  providers: [
    {
      provide: DATABASE_CONNECTION,
      useFactory: async () => await createConnection(),
    },
  ],
})
export class DatabaseModule {}

@Injectable()
export class UserRepository {
  constructor(@Inject(DATABASE_CONNECTION) private connection: DatabaseConnection) {}

  async findAll() {
    return this.connection.query('SELECT * FROM users');
  }
}
```

**How it works**

- `Symbol(...)` creates a token that is **unique by construction**: no other file can accidentally reuse it, unlike a string such as `'DATABASE'`.
- The token is exported from a shared file, so the module that provides it and every class that injects it import the same value. A typo becomes a compile error instead of a runtime failure.
- The factory is `async`, so the repository receives an already-created connection.

### Class token with `@Inject` (explicit, optional)

```ts
@Injectable()
export class AppService {
  constructor(
    @Inject(LoggerService) private logger: LoggerService
    // Normally just: private logger: LoggerService
  ) {}
}
```

For a class provider the type annotation is already the token, so `@Inject(LoggerService)` is redundant and the comment in the code shows the usual short form. Use `@Inject()` explicitly only when the token is **not** the parameter's type: strings, symbols, or interfaces.

### Conditional implementation with a union type

```ts
type PaymentProvider = StripePayment | PaypalPayment;

@Module({
  providers: [
    {
      provide: 'PAYMENT_PROVIDER',
      useFactory: (configService: ConfigService): PaymentProvider => {
        const provider = configService.get('PAYMENT_PROVIDER');
        return provider === 'stripe' ? new StripePayment() : new PaypalPayment();
      },
      inject: [ConfigService],
    },
  ],
})
export class PaymentModule {}

@Injectable()
export class CheckoutService {
  constructor(@Inject('PAYMENT_PROVIDER') private paymentProvider: PaymentProvider) {}
}
```

**How it works**

- The factory reads `PAYMENT_PROVIDER` from configuration and returns **either** a `StripePayment` or a `PaypalPayment`. The consumer does not know or care which.
- The union type `PaymentProvider` is the contract: both classes must offer the members `CheckoutService` uses.
- Switching providers is a configuration change, not a code change. A shared interface or abstract class is cleaner than a union once there are more than two options.

---

## useFactory

**Purpose:** create providers dynamically with a factory function.

| Point | Detail |
|-------|--------|
| Sync / async | The factory may be synchronous or return a Promise. |
| Dependencies | Listed in `inject`; passed to the factory as parameters **in the same order**. |
| Runs | Once at module initialization (singleton by default). |
| Good for | Database connections, API clients, config-based or environment-based providers. |

### Basic factory

```ts
@Module({
  providers: [
    {
      provide: 'API_CLIENT',
      useFactory: () => new ApiClient('https://api.example.com'),
    },
  ],
})
export class AppModule {}
```

The simplest factory: no dependencies, so no `inject` array. The function runs once while the module initializes and its return value is the provider. The URL is hardcoded only for the example; in real code read it from configuration.

### Async factory with dependencies

```ts
@Module({
  imports: [ConfigModule],
  providers: [
    {
      provide: 'DATABASE_CONNECTION',
      useFactory: async (configService: ConfigService) => {
        const dbConfig = configService.get('database');

        // Complex async initialization
        const connection = await createConnection({
          host: dbConfig.host,
          port: dbConfig.port,
          username: dbConfig.username,
          password: dbConfig.password,
        });

        // Run migrations, seed data, etc.
        await connection.runMigrations();

        return connection;
      },
      inject: [ConfigService],
    },
  ],
})
export class DatabaseModule {}
```

**How it works**

- `imports: [ConfigModule]` makes `ConfigService` available, and `inject: [ConfigService]` passes it to the factory.
- The factory is `async`: it connects, runs migrations, and only then returns the connection. Nest waits for it, so the application does not finish starting (and no dependent provider is created) until the database is ready.
- If anything in the factory throws, startup fails immediately, which is what you want for a missing database.

### Conditional on environment

```ts
providers: [
  {
    provide: 'PAYMENT_SERVICE',
    useFactory: (configService: ConfigService) => {
      const env = configService.get('NODE_ENV');

      if (env === 'production') {
        return new StripePaymentService(configService.get('STRIPE_KEY'));
      }
      return new MockPaymentService();  // testing / development
    },
    inject: [ConfigService],
  },
]
```

The factory chooses an implementation from `NODE_ENV`: the real Stripe client in production, a mock everywhere else, so development and tests never charge a card. Consumers inject `'PAYMENT_SERVICE'` and behave identically in both cases.

### Multiple dependencies

```ts
providers: [
  {
    provide: 'EMAIL_SERVICE',
    useFactory: (configService: ConfigService, logger: Logger, httpService: HttpService) => {
      const apiKey = configService.get('SENDGRID_API_KEY');
      logger.log('Initializing email service...');
      return new EmailService(apiKey, httpService);
    },
    inject: [ConfigService, Logger, HttpService],  // order matches the parameters
  },
]
```

The factory has three dependencies, listed in `inject` in the **same order** as the parameters (`ConfigService`, `Logger`, `HttpService`). Nest matches them by position, not by name, so reordering one array without the other silently passes the wrong object.

### Returning different implementations

```ts
providers: [
  {
    provide: 'CACHE_MANAGER',
    useFactory: (configService: ConfigService) => {
      const cacheType = configService.get('CACHE_TYPE');

      switch (cacheType) {
        case 'redis':
          return new RedisCacheManager(configService.get('REDIS_URL'));
        case 'memory':
          return new InMemoryCacheManager();
        case 'memcached':
          return new MemcachedManager(configService.get('MEMCACHED_URL'));
        default:
          throw new Error(`Unsupported cache type: ${cacheType}`);
      }
    },
    inject: [ConfigService],
  },
]
```

**Common use cases:** database connections, third-party SDK clients, environment-based service selection, async construction, loading config files, connection pools, message queue connections.

**Important notes**

- The `inject` array order must match the factory's parameter order.
- Use string/symbol tokens for provider names to avoid circular dependencies.

---

## @Global

**Purpose:** make a module's exports available across the entire application without importing it everywhere.

| Point | Detail |
|-------|--------|
| Registration | Import once, typically in the root or core module. |
| Exports | A global module must still **export** its providers. |
| Trade-off | Less boilerplate, but over-use hides dependencies and makes testing harder. |

```ts
import { Module, Global } from '@nestjs/common';

@Global()
@Module({
  providers: [DatabaseService, Logger, CacheService],
  exports: [DatabaseService, Logger, CacheService],  // must export
})
export class CoreModule {}

// AppModule: import once
@Module({
  imports: [CoreModule],
})
export class AppModule {}

// Any other module: no import needed
@Injectable()
export class UsersService {
  constructor(
    private databaseService: DatabaseService,  // available globally
    private logger: Logger,                     // available globally
  ) {}
}
```

**How it works**

- `@Global()` makes the module's exports visible everywhere without each module listing it in `imports`.
- It does **not** export anything by itself: the providers still have to be listed in `exports`, hence the comment.
- `AppModule` imports `CoreModule` once so Nest registers it. `UsersService` then injects `DatabaseService` and `Logger` with no import of `CoreModule` in its own module.

### Regular vs global module

```ts
// Regular: import in every module that needs it
@Module({ providers: [LoggerService], exports: [LoggerService] })
export class LoggerModule {}

@Module({ imports: [LoggerModule], providers: [UsersService] })   // import here...
export class UsersModule {}

@Module({ imports: [LoggerModule], providers: [OrdersService] })  // ...and here
export class OrdersModule {}

// Global: import once
@Global()
@Module({ providers: [LoggerService], exports: [LoggerService] })
export class LoggerModule {}

@Module({ imports: [LoggerModule] })      // only in AppModule
export class AppModule {}

@Module({ providers: [UsersService] })    // no import needed
export class UsersModule {}
```

The two halves show the same `LoggerModule` in both styles.

| | Regular module | Global module |
|---|---|---|
| Who imports it | Every module that uses `LoggerService` | Only `AppModule`, once |
| Dependencies visible in each module's `imports` | Yes | No (hidden) |
| Risk | Repetitive imports | Hidden dependencies and harder tests |

Prefer the regular form by default and use `@Global()` for infrastructure that nearly everything needs.

### Global configuration module

```ts
@Global()
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,      // ConfigModule's built-in global flag
      envFilePath: '.env',
    }),
  ],
  providers: [ConfigService],
  exports: [ConfigService],
})
export class GlobalConfigModule {}
```

`ConfigModule.forRoot({ isGlobal: true })` already makes `ConfigService` global, so wrapping it in a `@Global()` module like this is redundant: the `isGlobal` flag is enough. The example shows both only to illustrate that two different mechanisms can produce a global provider. `envFilePath` points at the file to load.

### Global database module

```ts
@Global()
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('DB_HOST'),
        port: configService.get('DB_PORT'),
        username: configService.get('DB_USER'),
        password: configService.get('DB_PASSWORD'),
        database: configService.get('DB_NAME'),
        autoLoadEntities: true,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class GlobalDatabaseModule {}
```

`TypeOrmModule.forRootAsync` builds the database options at startup from `ConfigService` (`useFactory` + `inject`), instead of hardcoding them. `autoLoadEntities: true` registers every entity added with `forFeature()` automatically. Making the module global means feature modules only need `TypeOrmModule.forFeature([Entity])` to get repositories.

### CoreModule pattern

```ts
@Global()
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), LoggerModule],
  providers: [
    DatabaseService,
    CacheService,
    { provide: 'APP_VERSION', useValue: '1.0.0' },
  ],
  exports: [DatabaseService, CacheService, 'APP_VERSION'],
})
export class CoreModule {}

// Import only in AppModule
@Module({
  imports: [CoreModule, UsersModule, OrdersModule, ProductsModule],
})
export class AppModule {}
```

**What a CoreModule bundles**

- `imports` brings in configuration and logging for the whole app.
- `providers` and `exports` expose `DatabaseService`, `CacheService` and a constant `'APP_VERSION'` (a value provider, injected with `@Inject('APP_VERSION')`).
- `AppModule` imports `CoreModule` **once**; every feature module can then use those providers with no import.

Keep a CoreModule to infrastructure. Business features belong in their own modules.

### When to use it

| Use `@Global` for | Avoid `@Global` for |
|-------------------|---------------------|
| Configuration services | Feature services (`UsersService`, `OrdersService`) |
| Logging | Business logic services |
| Database connections | Domain-specific repositories |
| Cache managers | Anything with clear module boundaries |
| Utility services (helpers, validators, formatters) | Anything that should declare explicit dependencies |
| Exception filters, guards and interceptors | |

**Best practices:** limit to one or two global modules (e.g. `CoreModule`, `SharedModule`), keep them to infrastructure concerns, document why something is global, prefer explicit imports otherwise, and test global modules thoroughly.

---

## Dynamic modules

Modules can be configured at import time with `.forRoot()` / `.forFeature()` style static methods that return a `DynamicModule`.

```ts
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        { provide: 'DATABASE_OPTIONS', useValue: options },
        DatabaseService,
      ],
      exports: [DatabaseService],
      global: true,  // optionally make it global
    };
  }
}

// Usage
@Module({
  imports: [
    DatabaseModule.forRoot({ host: 'localhost', port: 5432 }),
  ],
})
export class AppModule {}
```

**How it works**

- A **dynamic module** is a module whose metadata is computed by a static method. `@Module({})` is empty on purpose.
- `forRoot(options)` returns a `DynamicModule`: the class (`module`), plus the providers, exports and `global` flag built from the options you pass.
- `AppModule` calls `DatabaseModule.forRoot({ host, port })`; the options are stored under `'DATABASE_OPTIONS'` and `DatabaseService` can inject them.
- The same pattern is behind `ConfigModule.forRoot()` and `TypeOrmModule.forFeature()`.

---

## Module organization patterns

| Pattern | Purpose |
|---------|---------|
| **Feature module** | One module per feature (orders, users, ...). |
| **Shared module** | Common utilities shared across features. |
| **Core module** | Infrastructure loaded once (config, database, cache). |

### Feature module

```ts
@Module({
  imports: [TypeOrmModule.forFeature([Order])],
  controllers: [OrdersController],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

A feature module is self-contained: it registers its entity with `forFeature`, owns its controller and service, and **exports** only the service other modules may use. Other modules import `OrdersModule` and never touch its repository.

### Shared module

```ts
@Global()
@Module({
  providers: [LoggerService, HelperService, ValidatorService],
  exports: [LoggerService, HelperService, ValidatorService],
})
export class SharedModule {}
```

A shared module collects small, stateless utilities. Marking it `@Global()` avoids importing it in every module, which is acceptable for helpers used everywhere (logging, validation, formatting) but not for business services.

### Core module

```ts
@Global()
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule,
    CacheModule,
  ],
  providers: [AppLogger, AppCache],
  exports: [AppLogger, AppCache],
})
export class CoreModule {}
```

The core module is infrastructure loaded once. It imports configuration (global), the database and the cache, and exports the logger and cache wrappers. Because it is global, the rest of the app uses `AppLogger` and `AppCache` without importing this module.

---

## Complete module example

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';
import { User } from './entities/user.entity';
import { EmailModule } from '../email/email.module';

@Module({
  // 1. IMPORTS: bring in other modules
  imports: [
    TypeOrmModule.forFeature([User]),  // database repository for User
    EmailModule,                        // email functionality
    ConfigModule,                       // configuration
  ],

  // 2. CONTROLLERS: handle HTTP requests
  controllers: [UsersController],

  // 3. PROVIDERS: services, repositories, factories
  providers: [
    // Standard providers
    UsersService,
    UsersRepository,

    // Factory provider: dynamic creation
    {
      provide: 'USER_CONFIG',
      useFactory: (configService: ConfigService) => ({
        maxUsers: configService.get('MAX_USERS'),
        enableNotifications: configService.get('ENABLE_NOTIFICATIONS'),
      }),
      inject: [ConfigService],
    },

    // Value provider: static configuration
    { provide: 'DEFAULT_USER_ROLE', useValue: 'customer' },
  ],

  // 4. EXPORTS: make available to other modules
  exports: [
    UsersService,       // other modules can inject this
    UsersRepository,    // other modules can inject this
    'USER_CONFIG',      // other modules can inject this
    TypeOrmModule,      // re-export for other modules
  ],
})
export class UsersModule {}
```

**How the four parts fit together**

1. **`imports`**: the module needs the `User` repository (`forFeature`), email functionality and configuration.
2. **`controllers`**: `UsersController` exposes the HTTP routes.
3. **`providers`**: two classes, a factory that builds a `'USER_CONFIG'` object from `ConfigService`, and a fixed value `'DEFAULT_USER_ROLE'`.
4. **`exports`**: the public API. Other modules can inject `UsersService`, `UsersRepository` and `'USER_CONFIG'`, and re-exporting `TypeOrmModule` hands them the repository providers too. `'DEFAULT_USER_ROLE'` is not exported, so it stays private.

Export only what other modules truly need; each export becomes something you must keep stable.

---

## Rules and takeaways

| Concept | Summary |
|---------|---------|
| `@Module()` | Core decorator that organizes the application structure. |
| `imports` | Consume other modules (get their exported providers). |
| `exports` | Share providers with other modules (public API). |
| `controllers` | Handle HTTP requests (routes). |
| `providers` | Injectable services, repositories, factories (business logic). |
| `useFactory` | Create providers dynamically with runtime logic. |
| `@Global` | Make module exports available everywhere (use sparingly). |

**Module rules**

1. Each module has a single, well-defined purpose.
2. Export only what other modules need (encapsulation).
3. Use `@Global` sparingly; prefer explicit imports.
4. Keep controllers thin and delegate to services.
5. Keep related features together in the same module.
6. Use dynamic modules for configurable functionality.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Forgetting `@Injectable()` on a provider class | Nest cannot resolve its constructor dependencies | Decorate every provider class |
| Exporting something the module neither provides nor imports | Nest throws at startup | Provide it (or import the module that provides it) first |
| `inject` array order differs from the factory parameter order | Parameters receive the wrong dependencies | Keep both in the same order |
| Overusing `@Global()` | Hides dependencies and makes modules harder to test | Reserve it for config, logging, database and cache |
| Two modules importing each other | Circular dependency error | Move shared code into a third module, or use `forwardRef()` as a last resort |
| String tokens that collide (`'CONFIG'` used twice) | One provider silently replaces the other | Use `Symbol()` tokens or unique names |
