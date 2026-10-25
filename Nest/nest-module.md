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

### Value provider

```ts
providers: [
  {
    provide: 'CONFIG',
    useValue: { apiKey: '12345', apiUrl: 'https://api.example.com' },
  },
]
```

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

### Shared module

```ts
@Global()
@Module({
  providers: [LoggerService, HelperService, ValidatorService],
  exports: [LoggerService, HelperService, ValidatorService],
})
export class SharedModule {}
```

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
