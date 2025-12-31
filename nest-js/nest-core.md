### NestJS core concepts

Modules, controllers, providers, and exports are the building blocks of a NestJS application. This note summarizes their roles and shows minimal examples for clarity.

## Modules

A module is an organizational boundary and a unit of composition. A module's metadata declares which controllers, providers, and exports it contains, and which other modules it imports.

Example:

```ts
@Module({
  controllers: [BooksController],
  providers: [BooksService],
  exports: [BooksService],
})
export class BooksModule {}
```

Key points:
- Modules group related classes and make them discoverable by Nest's DI container.
- Use `imports` to consume exported providers from other modules.

## Controllers

Controllers handle incoming requests, define routes, and delegate work to providers (services).

Minimal controller example:

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

What controllers do:
- Handle HTTP requests
- Define route handlers (`@Get()`, `@Post()`, etc.)
- Call providers to perform business logic

## Providers

Providers are classes that encapsulate business logic. They are managed by Nest's Dependency Injection container.

Minimal provider (service) example:

```ts
@Injectable()
export class BooksService {
  findAll() {
    return [];
  }
}
```

Notes on providers:
- Providers are typically singletons (one instance per application) by default.
- Nest supports other scopes: `REQUEST`-scoped and `TRANSIENT` providers when needed.
- Providers include services, repositories, factories, guards, interceptors, pipes, etc.

## Exports

Exports make providers from one module available to other modules that import it.

Example:

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

- If `BooksService` is exported, `OrdersModule` can inject it. If not exported, the provider stays private to `BooksModule` and a DI error occurs when other modules attempt to inject it.

## Registration & bootstrap (AppModule)

Modules must be registered so Nest can include them in the application context. The root module (commonly `AppModule`) composes the application.

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

When the application boots, Nest creates a DI context from the registered modules; those modules (and their exported providers) become reachable within that context.

## Global modules

Marking a module with `@Global()` makes its exported providers available throughout the application without re-importing the module in each feature module. The module still needs to be imported once (typically in the root module) so Nest registers it.

```ts
@Global()
@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule {}
```

After a global module is imported once, its exported providers are available everywhere and do not need to be re-imported in other modules.

---