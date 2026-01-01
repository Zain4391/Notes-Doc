# NestJS Modules - Complete Guide

## @Module Decorator

The `@Module()` decorator is the core building block of NestJS applications. It provides metadata that Nest uses to organize the application structure.

### Basic Syntax

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

---

## Module Properties

### 1. **imports**

**Purpose:** Imports other modules whose providers are required in the current module.

**Key Points:**
- Brings in functionality from other modules
- Makes the exported providers of imported modules available in the current module
- Required when you need to use services/providers from another module
- Creates module dependency graph
- Prevents circular dependencies (Nest detects and warns about them)

**Example:**

```ts
@Module({
  imports: [
    TypeOrmModule,           // Database module
    ConfigModule,            // Configuration module
    UsersModule,             // Custom user module
    AuthModule,              // Authentication module
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

**How it works:**
- When you import `UsersModule`, any providers that `UsersModule` exports become available
- You can inject those exported providers into the current module's providers/controllers
- Only exported providers are accessible; internal providers remain private to their module

**Common Use Cases:**
- Import database modules (TypeOrmModule, MongooseModule)
- Import configuration modules (ConfigModule)
- Import shared utility modules
- Import feature modules that provide services you need
- Import third-party NestJS modules

---

### 2. **exports**

**Purpose:** Exposes providers to other modules that import this module.

**Key Points:**
- Makes providers from this module available to other modules
- Exported providers can be injected into other modules that import this one
- Can export both providers defined in this module AND imported modules
- Creates a public API for your module
- Only export what's necessary (principle of least privilege)
- Unexported providers are private to the module

**Example:**

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UsersRepository],  // Both available in this module
  exports: [UsersService],                      // Only UsersService available to others
})
export class UsersModule {}

// In another module
@Module({
  imports: [UsersModule],  // Now can inject UsersService
  controllers: [OrdersController],
  providers: [OrdersService],
})
export class OrdersModule {}

// OrdersService can inject UsersService
@Injectable()
export class OrdersService {
  constructor(private usersService: UsersService) {}  // ✅ Works
  // constructor(private usersRepo: UsersRepository) {}  // ❌ Not exported
}
```

**Exporting Imported Modules:**

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule],  // Re-export TypeOrmModule
})
export class UsersModule {}

// Now other modules that import UsersModule get TypeOrmModule too
```

**Common Patterns:**
- Export services that other modules need
- Export repository modules for database entities
- Create "barrel" modules that re-export multiple modules
- Export shared utilities and helpers

---

### 3. **controllers**

**Purpose:** Defines the controllers that handle HTTP requests for this module.

**Key Points:**
- Controllers handle incoming HTTP requests and return responses
- Each module should contain controllers related to its domain
- Controllers are automatically instantiated by Nest
- Dependency injection works in controllers
- Controllers should be thin (delegate logic to services)
- Decorated with `@Controller()` decorator
- Use route decorators: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, etc.

**Example:**

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
  controllers: [UsersController, UsersAdminController],  // Multiple controllers
  providers: [UsersService],
})
export class UsersModule {}
```

**Best Practices:**
- Group related endpoints in the same controller
- Keep controllers focused on a single resource/domain
- Use DTOs (Data Transfer Objects) for request validation
- Apply guards and interceptors at controller level when needed
- Don't put business logic in controllers - use services

**Controller Responsibilities:**
- Define routes (endpoints)
- Validate request data using DTOs
- Call service methods to handle business logic
- Format and return responses
- Handle request/response transformation
- Apply guards, interceptors, and pipes

---

### 4. **providers**

**Purpose:** Defines the services, repositories, factories, and helpers that can be injected.

**Key Points:**
- Providers are the core of dependency injection in Nest
- Most commonly services, but can be any class that can be injected
- Must be decorated with `@Injectable()` decorator
- Automatically managed by Nest's IoC (Inversion of Control) container
- Default scope is singleton (one instance shared across the app)
- Can be injected into controllers, other providers, guards, interceptors, etc.

**Basic Provider (Service):**

```ts
@Injectable()
export class UsersService {
  findAll() {
    return ['user1', 'user2'];
  }
}

@Module({
  providers: [UsersService],  // Shorthand syntax
  // Equivalent to: providers: [{ provide: UsersService, useClass: UsersService }]
})
export class UsersModule {}
```

**Provider Types:**

**a) Class Providers (Standard):**

```ts
providers: [
  UsersService,  // Shorthand
  // Or explicit:
  {
    provide: UsersService,
    useClass: UsersService,
  },
]
```

**b) Value Providers:**

```ts
providers: [
  {
    provide: 'CONFIG',
    useValue: {
      apiKey: '12345',
      apiUrl: 'https://api.example.com',
    },
  },
]

// Inject using @Inject() with proper typing
interface AppConfig {
  apiKey: string;
  apiUrl: string;
}

@Injectable()
export class SomeService {
  constructor(@Inject('CONFIG') private config: AppConfig) {
    // config is now properly typed
    console.log(this.config.apiKey);
  }
}
```

**More @Inject() Examples with Types:**

```ts
// 1. String token with interface type
interface DatabaseConfig {
  host: string;
  port: number;
  database: string;
}

@Module({
  providers: [
    {
      provide: 'DATABASE_CONFIG',
      useValue: {
        host: 'localhost',
        port: 5432,
        database: 'mydb',
      },
    },
  ],
})
export class ConfigModule {}

@Injectable()
export class DatabaseService {
  constructor(
    @Inject('DATABASE_CONFIG') private dbConfig: DatabaseConfig
  ) {
    // dbConfig is typed as DatabaseConfig
    console.log(`Connecting to ${this.dbConfig.host}:${this.dbConfig.port}`);
  }
}

// 2. Factory provider with typed injection
@Module({
  providers: [
    {
      provide: 'API_CLIENT',
      useFactory: (config: AppConfig) => {
        return new ApiClient(config.apiUrl);
      },
      inject: ['CONFIG'],
    },
  ],
})
export class ApiModule {}

@Injectable()
export class ProductService {
  constructor(
    @Inject('API_CLIENT') private apiClient: ApiClient
  ) {
    // apiClient is typed as ApiClient class
  }
}

// 3. Symbol token (recommended for type safety)
export const DATABASE_CONNECTION = Symbol('DATABASE_CONNECTION');

interface DatabaseConnection {
  query: (sql: string) => Promise<any>;
  close: () => Promise<void>;
}

@Module({
  providers: [
    {
      provide: DATABASE_CONNECTION,
      useFactory: async () => {
        return await createConnection();
      },
    },
  ],
})
export class DatabaseModule {}

@Injectable()
export class UserRepository {
  constructor(
    @Inject(DATABASE_CONNECTION) private connection: DatabaseConnection
  ) {
    // connection is typed as DatabaseConnection interface
  }

  async findAll() {
    return this.connection.query('SELECT * FROM users');
  }
}

// 4. Class token with @Inject (explicit injection)
@Injectable()
export class LoggerService {
  log(message: string) {
    console.log(message);
  }
}

@Injectable()
export class AppService {
  constructor(
    @Inject(LoggerService) private logger: LoggerService
    // Usually you'd just do: private logger: LoggerService
    // @Inject is optional for class tokens but can be explicit
  ) {}
}

// 5. Conditional injection with typing
type PaymentProvider = StripePayment | PaypalPayment;

@Module({
  providers: [
    {
      provide: 'PAYMENT_PROVIDER',
      useFactory: (configService: ConfigService): PaymentProvider => {
        const provider = configService.get('PAYMENT_PROVIDER');
        return provider === 'stripe' 
          ? new StripePayment() 
          : new PaypalPayment();
      },
      inject: [ConfigService],
    },
  ],
})
export class PaymentModule {}

@Injectable()
export class CheckoutService {
  constructor(
    @Inject('PAYMENT_PROVIDER') private paymentProvider: PaymentProvider
  ) {
    // paymentProvider is typed as union type
  }
}
```

**c) Factory Providers (useFactory):**

```ts
providers: [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: async (configService: ConfigService) => {
      const config = configService.get('database');
      return await createConnection(config);
    },
    inject: [ConfigService],  // Dependencies for factory
  },
]
```

**d) Existing Providers (Alias):**

```ts
providers: [
  UsersService,
  {
    provide: 'USERS_SERVICE_ALIAS',
    useExisting: UsersService,  // Points to existing provider
  },
]
```

**Common Provider Examples:**
- Services (business logic)
- Repositories (data access)
- Factories (dynamic object creation)
- Helpers/Utilities
- Custom providers with tokens

**Scope Options:**
- `DEFAULT` (Singleton) - One instance for entire app
- `REQUEST` - New instance per request
- `TRANSIENT` - New instance every time it's injected

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}
```

---

### 5. **useFactory**

**Purpose:** Creates providers dynamically using a factory function.

**Key Points:**
- Allows dynamic provider creation based on runtime conditions
- Factory function can be synchronous or asynchronous
- Can inject dependencies into the factory using `inject` array
- Useful for complex initialization logic
- Can return different implementations based on environment
- Perfect for database connections, API clients, configuration-based providers
- Factory runs once at application startup (unless using REQUEST scope)

**Basic Factory:**

```ts
@Module({
  providers: [
    {
      provide: 'API_CLIENT',
      useFactory: () => {
        return new ApiClient('https://api.example.com');
      },
    },
  ],
})
export class AppModule {}
```

**Async Factory with Dependencies:**

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
      inject: [ConfigService],  // Inject dependencies here
    },
  ],
})
export class DatabaseModule {}
```

**Conditional Provider Based on Environment:**

```ts
providers: [
  {
    provide: 'PAYMENT_SERVICE',
    useFactory: (configService: ConfigService) => {
      const env = configService.get('NODE_ENV');
      
      if (env === 'production') {
        return new StripePaymentService(configService.get('STRIPE_KEY'));
      } else {
        return new MockPaymentService();  // Testing/Development
      }
    },
    inject: [ConfigService],
  },
]
```

**Multiple Dependencies:**

```ts
providers: [
  {
    provide: 'EMAIL_SERVICE',
    useFactory: (
      configService: ConfigService,
      logger: Logger,
      httpService: HttpService,
    ) => {
      const apiKey = configService.get('SENDGRID_API_KEY');
      logger.log('Initializing email service...');
      
      return new EmailService(apiKey, httpService);
    },
    inject: [ConfigService, Logger, HttpService],
  },
]
```

**Factory Returning Different Implementations:**

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

**Common Use Cases:**
- Creating database connections
- Initializing third-party SDK clients
- Environment-based service selection
- Complex object construction requiring async operations
- Loading configuration files
- Creating connection pools
- Initializing message queue connections

**Important Notes:**
- The `inject` array order must match factory function parameter order
- Factory functions can be async (return Promise)
- Use tokens (strings) for provider names to avoid circular dependencies
- Factory is called once during module initialization (singleton by default)

---

### 6. **@Global**

**Purpose:** Makes a module globally available across the entire application.

**Key Points:**
- Decorated modules have their exports available everywhere without importing
- Should be used sparingly (only for truly global utilities)
- Typically registered once in root module or core module
- Reduces boilerplate imports in every module
- Common for: database connections, configuration, logging, caching
- Over-use can make dependencies unclear and harder to test
- Global modules still need to export providers to make them available

**Basic Global Module:**

```ts
import { Module, Global } from '@nestjs/common';

@Global()
@Module({
  providers: [DatabaseService, Logger, CacheService],
  exports: [DatabaseService, Logger, CacheService],  // Must export!
})
export class CoreModule {}

// In AppModule
@Module({
  imports: [CoreModule],  // Only import once in root
  // ...
})
export class AppModule {}

// In any other module - no need to import CoreModule
@Injectable()
export class UsersService {
  constructor(
    private databaseService: DatabaseService,  // ✅ Available globally
    private logger: Logger,                     // ✅ Available globally
  ) {}
}
```

**Global Configuration Module:**

```ts
@Global()
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,  // ConfigModule's built-in global flag
      envFilePath: '.env',
    }),
  ],
  providers: [ConfigService],
  exports: [ConfigService],
})
export class GlobalConfigModule {}
```

**Global Database Module:**

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

**When to Use @Global:**
- Configuration services (used everywhere)
- Logging services (needed in all modules)
- Database connections (shared across app)
- Cache managers (global cache)
- Utility services (helpers, validators, formatters)
- Exception filters
- Guards and interceptors

**When NOT to Use @Global:**
- Feature-specific services (UsersService, OrdersService)
- Business logic services
- Domain-specific repositories
- Services with clear module boundaries
- Anything that should have explicit dependencies

**Best Practices:**
- Limit to one or two global modules (e.g., CoreModule, SharedModule)
- Keep global modules focused on infrastructure concerns
- Document why something is global
- Prefer explicit imports for better code organization
- Test global modules thoroughly

**Example CoreModule Pattern:**

```ts
@Global()
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    LoggerModule,
  ],
  providers: [
    DatabaseService,
    CacheService,
    {
      provide: 'APP_VERSION',
      useValue: '1.0.0',
    },
  ],
  exports: [
    DatabaseService,
    CacheService,
    'APP_VERSION',
  ],
})
export class CoreModule {}

// Import only in AppModule
@Module({
  imports: [
    CoreModule,  // Once here
    UsersModule,
    OrdersModule,
    ProductsModule,
  ],
})
export class AppModule {}

// All feature modules can use CoreModule providers without importing
```

**Comparison: Regular vs Global Module:**

```ts
// Regular Module - Must import everywhere
@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class LoggerModule {}

@Module({
  imports: [LoggerModule],  // Need to import
  providers: [UsersService],
})
export class UsersModule {}

@Module({
  imports: [LoggerModule],  // Need to import again
  providers: [OrdersService],
})
export class OrdersModule {}

// Global Module - Import once
@Global()
@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class LoggerModule {}

@Module({
  imports: [LoggerModule],  // Only in AppModule
})
export class AppModule {}

@Module({
  // No import needed
  providers: [UsersService],
})
export class UsersModule {}

@Module({
  // No import needed
  providers: [OrdersService],
})
export class OrdersModule {}
```

---

## Complete Module Example

```ts
import { Module, Global } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';
import { User } from './entities/user.entity';
import { EmailService } from '../email/email.service';
import { EmailModule } from '../email/email.module';

@Module({
  // 1. IMPORTS - Bring in other modules
  imports: [
    TypeOrmModule.forFeature([User]),  // Database repository for User
    EmailModule,                        // Email functionality
    ConfigModule,                       // Configuration
  ],

  // 2. CONTROLLERS - Handle HTTP requests
  controllers: [UsersController],

  // 3. PROVIDERS - Services, repositories, factories
  providers: [
    // Standard provider
    UsersService,
    UsersRepository,

    // Factory provider - dynamic creation
    {
      provide: 'USER_CONFIG',
      useFactory: (configService: ConfigService) => {
        return {
          maxUsers: configService.get('MAX_USERS'),
          enableNotifications: configService.get('ENABLE_NOTIFICATIONS'),
        };
      },
      inject: [ConfigService],
    },

    // Value provider - static configuration
    {
      provide: 'DEFAULT_USER_ROLE',
      useValue: 'customer',
    },
  ],

  // 4. EXPORTS - Make available to other modules
  exports: [
    UsersService,        // Other modules can inject this
    UsersRepository,     // Other modules can inject this
    'USER_CONFIG',       // Other modules can inject this
    TypeOrmModule,       // Re-export for other modules
  ],
})
export class UsersModule {}
```

---

## Module Organization Patterns

### Feature Module Pattern

```ts
// Each feature gets its own module
@Module({
  imports: [TypeOrmModule.forFeature([Order])],
  controllers: [OrdersController],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

### Shared Module Pattern

```ts
// Common utilities shared across features
@Global()
@Module({
  providers: [
    LoggerService,
    HelperService,
    ValidatorService,
  ],
  exports: [
    LoggerService,
    HelperService,
    ValidatorService,
  ],
})
export class SharedModule {}
```

### Core Module Pattern

```ts
// Core infrastructure (loaded once)
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

## Dynamic Modules

Modules can be configured dynamically with `.forRoot()` and `.forFeature()` patterns:

```ts
// Dynamic module example
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
        DatabaseService,
      ],
      exports: [DatabaseService],
      global: true,  // Can make it global
    };
  }
}

// Usage
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
    }),
  ],
})
export class AppModule {}
```

---

## Key Takeaways

- **@Module()** - Core decorator that organizes application structure
- **imports** - Consume other modules (get their exported providers)
- **exports** - Share providers with other modules (public API)
- **controllers** - Handle HTTP requests (routes)
- **providers** - Injectable services, repositories, factories (business logic)
- **useFactory** - Create providers dynamically with runtime logic
- **@Global** - Make module exports available everywhere (use sparingly)

**Module Rules:**
1. Each module should have a single, well-defined purpose
2. Export only what other modules need (encapsulation)
3. Use @Global sparingly (prefer explicit imports)
4. Controllers should be thin (delegate to services)
5. Keep related features together in the same module
6. Use dynamic modules for configurable functionality
