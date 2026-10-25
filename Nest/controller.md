# NestJS Controllers

How to define routes and handle requests: HTTP method decorators, route, query and body parameters, status codes, headers, guards, pipes and validation.

**Contents**

1. [Basic controller setup](#basic-controller-setup)
2. [HTTP method decorators](#http-method-decorators)
3. [Route parameters](#route-parameters)
4. [Query parameters](#query-parameters)
5. [Request body](#request-body)
6. [Request object decorators](#request-object-decorators)
7. [HTTP status codes](#http-status-codes)
8. [Response headers](#response-headers)
9. [Guards](#guards)
10. [Pipes](#pipes)
11. [Validation with ValidationPipe](#validation-with-validationpipe)
12. [class-validator decorators](#class-validator-decorators)
13. [Complete controller example](#complete-controller-example)
14. [Best practices](#best-practices)
15. [Common mistakes](#common-mistakes)

Related: [Services](service.md) (the logic behind the handlers) · [Custom Exceptions](custom-exception.md) (error responses) · [JWT Authentication](jwt-authentication.md) (guards in practice)

---

## Basic controller setup

A controller handles incoming requests and returns responses. `@Controller()` defines the **route prefix** for every route in the class.

```ts
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

Route: `GET /cats`

| Element | Description |
|---------|-------------|
| `@Controller('cats')` | Marks the class as a controller; `'cats'` is the route prefix. |
| `@Get()` | Maps the method to `GET /cats`. |
| Return value | Serialized automatically (strings and objects/arrays as JSON). |

---

## HTTP method decorators

| Decorator | HTTP method | Typical use |
|-----------|-------------|-------------|
| `@Get()` | GET | Retrieve data. |
| `@Post()` | POST | Create a resource. |
| `@Put()` | PUT | Full update (replace). |
| `@Patch()` | PATCH | Partial update. |
| `@Delete()` | DELETE | Remove a resource. |

Each takes an optional path (`@Get(':id')`) appended to the controller prefix.

```ts
@Get()
findAll() {
  return 'This action returns all cats';
}

@Post()
create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}

@Put(':id')
update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
  return `This action updates a #${id} cat`;
}

@Patch(':id')
partialUpdate(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
  return `This action partially updates a #${id} cat`;
}

@Delete(':id')
remove(@Param('id') id: string) {
  return `This action removes a #${id} cat`;
}
```

**How it works**

- Every handler is a plain method; the decorator on top decides the HTTP method and the path (`:id` is a placeholder that becomes the route parameter).
- `@Put(':id')` and `@Patch(':id')` look the same in code. The difference is the contract: PUT **replaces** the whole resource, so the body must be complete; PATCH changes **only the fields sent**. Nest does not enforce this, your service logic does.
- `@Body()` hands the handler the request body as a typed DTO, and `@Param('id')` hands it the matching path segment.
- The handlers return strings only to keep the example short. In a real controller they return whatever the service returns, and Nest serializes it (objects and arrays become JSON).

---

## Route parameters

`@Param()` extracts dynamic path variables.

| Form | Result |
|------|--------|
| `@Param('id') id` | One parameter. |
| `@Param() params` | Object containing all parameters. |

### Single parameter

```ts
@Get(':id')
findOne(@Param('id') id: string) {
  return `This action returns a #${id} cat`;
}
```

`GET /cats/123` gives `id = "123"`

### Multiple parameters

```ts
@Get(':category/:id')
findByCategoryAndId(
  @Param('category') category: string,
  @Param('id') id: string,
) {
  return `Category: ${category}, ID: ${id}`;
}
```

`GET /cats/persian/123` gives `category = "persian"`, `id = "123"`

### All parameters

```ts
@Get(':id')
findOne(@Param() params: any) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

> Route parameters arrive as **strings**. Use [`ParseIntPipe`](#built-in-pipes) to convert.

---

## Query parameters

`@Query()` extracts query-string values.

| Form | Result |
|------|--------|
| `@Query('limit') limit` | One query parameter. |
| `@Query() query` | Object containing all query parameters. |

```ts
// Single
@Get()
findAll(@Query('limit') limit: string) {
  return `This action returns all cats (limit: ${limit} items)`;
}
// GET /cats?limit=10  ->  limit = "10"

// Multiple
@Get()
findAll(@Query('limit') limit: string, @Query('offset') offset: string) {
  return `Limit: ${limit}, Offset: ${offset}`;
}
// GET /cats?limit=10&offset=20

// All at once
@Get()
findAll(@Query() query: ListAllEntities) {
  return `This action returns all cats (limit: ${query.limit} items)`;
}
```

**How it works**

- `@Query('limit')` reads one named key from the query string. `@Query()` with no name returns every key as an object, which is the form you pair with a DTO.
- Query-string values always arrive as **strings**: `limit` is `"10"`, not `10`. Convert with `ParseIntPipe` or a DTO with `@Type(() => Number)`.
- A parameter that is not in the URL is `undefined`, so give it a default or mark it optional.

---

## Request body

`@Body()` extracts the request body. Used with POST, PUT and PATCH.

| Form | Result |
|------|--------|
| `@Body() dto` | The full body (use a DTO class for validation). |
| `@Body('name') name` | One property of the body. |

```ts
// Full body
@Post()
create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}

// Specific properties
@Post()
create(@Body('name') name: string, @Body('age') age: number) {
  return `Creating cat: ${name}, age: ${age}`;
}

// Inline type
@Post('user')
async signupUser(@Body() userData: { name?: string; email: string }) {
  return this.userService.createUser(userData);
}
```

**How it works**

- `@Body() createCatDto: CreateCatDto` gives you the whole parsed JSON body. Typing it with a **class** lets `ValidationPipe` check it against the DTO's decorators (see [Validation with ValidationPipe](#validation-with-validationpipe)).
- `@Body('name')` pulls out a single property. It is quick, but each property is then validated on its own and you lose the single DTO that documents the request shape.
- The inline type `{ name?: string; email: string }` only helps the compiler. Types vanish at runtime, so nothing is validated; use a DTO class for real input.

---

## Request object decorators

| Decorator | Description | Express equivalent |
|-----------|-------------|--------------------|
| `@Request()`, `@Req()` | Full request object | `req` |
| `@Response()`, `@Res()` | Full response object | `res` |
| `@Next()` | Next middleware function | `next` |
| `@Session()` | Session object | `req.session` |
| `@Param(key?)` | Route parameters | `req.params` / `req.params[key]` |
| `@Body(key?)` | Request body | `req.body` / `req.body[key]` |
| `@Query(key?)` | Query parameters | `req.query` / `req.query[key]` |
| `@Headers(name?)` | Request headers | `req.headers` / `req.headers[name]` |
| `@Ip()` | Client IP address | `req.ip` |
| `@HostParam()` | Host parameters | `req.hosts` |

```ts
@Get()
findAll(
  @Req() request: Request,
  @Query() query: any,
  @Headers('user-agent') userAgent: string,
  @Ip() ip: string,
) {
  console.log('User Agent:', userAgent);
  console.log('IP:', ip);
  return this.catsService.findAll(query);
}
```

**How it works**

- Each decorator pulls one thing out of the request and passes it as a normal argument, so the handler does not have to touch `req` directly.
- `@Headers('user-agent')` reads a single header (header names are matched case-insensitively), and `@Ip()` reads the client address.
- `@Req()` gives the raw request. It is handy for rarely needed details, but ties the handler to Express or Fastify, so prefer the specific decorators when one exists.

---

## HTTP status codes

Defaults: **200** for GET, PUT, PATCH and DELETE; **201** for POST.

### `@HttpCode()`: set a custom status

```ts
import { HttpCode } from '@nestjs/common';

@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

`@HttpCode(204)` overrides the default status (201 for POST). A `204 No Content` response carries no body, so a handler using it should return nothing; the string returned here would not be sent.

### `@Res()`: manual control

Using `@Res()` switches the handler to library-specific mode; you must send the response yourself. Avoid unless needed.

```ts
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
    res.status(HttpStatus.OK).json([]);
  }
}
```

**How it works**

- Injecting `@Res() res` switches that handler into **library-specific mode**: Nest stops sending the return value and you must end the response yourself (`send()`, `json()`).
- `res.status(HttpStatus.CREATED)` sets the status; `HttpStatus` gives readable names instead of bare numbers.
- Interceptors and `@HttpCode()`/`@Header()` no longer apply to this handler. If you only need the response object for something small, use `@Res({ passthrough: true })` and keep returning values normally.

### `HttpStatus` enum

| Category | Member | Code |
|----------|--------|------|
| Success | `HttpStatus.OK` | 200 |
| | `HttpStatus.CREATED` | 201 |
| | `HttpStatus.ACCEPTED` | 202 |
| | `HttpStatus.NO_CONTENT` | 204 |
| Client error | `HttpStatus.BAD_REQUEST` | 400 |
| | `HttpStatus.UNAUTHORIZED` | 401 |
| | `HttpStatus.FORBIDDEN` | 403 |
| | `HttpStatus.NOT_FOUND` | 404 |
| | `HttpStatus.CONFLICT` | 409 |
| Server error | `HttpStatus.INTERNAL_SERVER_ERROR` | 500 |
| | `HttpStatus.NOT_IMPLEMENTED` | 501 |
| | `HttpStatus.BAD_GATEWAY` | 502 |

---

## Response headers

`@Header(name, value)` sets a response header. Stack it for multiple headers.

```ts
import { Header } from '@nestjs/common';

@Post()
@Header('Cache-Control', 'no-store')
@Header('X-Custom-Header', 'custom-value')
create() {
  return 'This action adds a new cat';
}
```

`@Header(name, value)` sets a fixed response header. Stacked decorators each add one header, and they run for every response from this handler. For headers that depend on the data (an ETag, a `Location` pointing at a new id), set them from the handler with `@Res({ passthrough: true })` instead.

---

## Guards

Guards decide whether a request is handled. Typical use: authentication and authorization.

| Level | How |
|-------|-----|
| Method | `@UseGuards(AuthGuard)` on a handler. |
| Controller | `@UseGuards(AuthGuard)` on the class: applies to every route. |
| Multiple | `@UseGuards(AuthGuard, RolesGuard)`: executed in order. |
| Global | `app.useGlobalGuards()` or the `APP_GUARD` provider. |

### Method level

```ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { AuthGuard } from './auth.guard';

@Controller('cats')
export class CatsController {
  @Get()
  @UseGuards(AuthGuard)
  findAll() {
    return 'This route is protected';
  }
}
```

**How it works**

- `@UseGuards(AuthGuard)` attaches the guard to this one route. Nest runs the guard before the handler.
- The guard's `canActivate()` returns `true` to let the request through. Returning `false` or throwing (for example `UnauthorizedException`) stops it, and the handler never runs.
- Passing the class (not `new AuthGuard()`) lets Nest create it and inject its dependencies.

### Controller level

```ts
@Controller('cats')
@UseGuards(AuthGuard)
export class CatsController {
  @Get()
  findAll() {
    return 'Protected route';
  }

  @Post()
  create() {
    return 'Also protected';
  }
}
```

Putting `@UseGuards` on the class protects **every** route in the controller, here both `findAll` and `create`. This is the right place for a guard that all routes need; add method-level guards only for the extra checks.

### Multiple guards

```ts
@Get()
@UseGuards(AuthGuard, RolesGuard)
findAll() {
  return 'Protected by both guards';
}
```

With several guards, Nest runs them **in the order listed** and the request must pass all of them. `AuthGuard` runs first (it sets `request.user`), then `RolesGuard` can read that user. Reversing the order would make `RolesGuard` run without a user.

### Global guard

In `main.ts` (cannot use dependency injection):

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalGuards(new AuthGuard());
  await app.listen(3000);
}
bootstrap();
```

`useGlobalGuards` applies the guard to every route in the application. Because the guard is created with `new` outside any module, it **cannot use dependency injection**. If the guard needs a service (a `JwtService`, a repository), register it with the `APP_GUARD` provider shown next, which Nest builds with full DI.

In `app.module.ts` (supports dependency injection):

```ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { AuthGuard } from './auth.guard';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
  ],
})
export class AppModule {}
```

**How it works**

- `provide: APP_GUARD` is a special token. Nest sees it and applies the `useClass` guard to **every** route, the same effect as `useGlobalGuards`.
- Because Nest creates the guard, it can use **dependency injection** (for example a `JwtService` or `Reflector`), which `new AuthGuard()` in `main.ts` cannot.
- Prefer this form whenever the guard has dependencies.

---

## Pipes

Pipes transform or validate the arguments before they reach the route handler.

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return `This action returns cat with id: ${id}`;
  // id is automatically converted to a number
}
```

`ParseIntPipe` runs before the handler: it converts the string `"42"` to the number `42`, and rejects `"abc"` with a `400 Bad Request` before your code runs. The handler can therefore treat `id` as a real number.

### Built-in Pipes

| Pipe | Behavior |
|------|----------|
| `ParseIntPipe` | String to integer. |
| `ParseFloatPipe` | String to float. |
| `ParseBoolPipe` | String to boolean. |
| `ParseArrayPipe` | String to array. |
| `ParseUUIDPipe` | Validates a UUID string. |
| `ParseEnumPipe` | Validates the value is a member of an enum. |

All throw `BadRequestException` when the value cannot be converted.

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  console.log(typeof id === 'number'); // true
  return this.catsService.findOne(id);
}

@Get()
findAll(@Query('active', ParseBoolPipe) active: boolean) {
  return this.catsService.findAll(active);
}

@Get()
findByIds(@Query('ids', new ParseArrayPipe({ items: Number, separator: ',' })) ids: number[]) {
  return this.catsService.findByIds(ids);
}

@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  return this.catsService.findOne(id);
}

@Get()
findByPrice(@Query('price', ParseFloatPipe) price: number) {
  return this.catsService.findByPrice(price);
}
```

**How to read these handlers**

- Each handler asks for one value and names the pipe that converts it: `ParseIntPipe` for `id`, `ParseBoolPipe` for `active`, `ParseArrayPipe` for `ids`, `ParseUUIDPipe` for a UUID, `ParseFloatPipe` for `price`.
- The pipe runs **before** the handler. A failed conversion returns `400 Bad Request`, so the service only ever sees valid, correctly typed values.
- `ParseBoolPipe` accepts the text `true` and `false`. `ParseArrayPipe` splits a string on a separator (`,` by default), so `ids=1,2,3` becomes an array; it needs `items: Number` to convert each element, which is why that one handler creates it with `new ParseArrayPipe({ items: Number, separator: ',' })`.

`ParseEnumPipe` needs the enum, so it is instantiated with `new`:

```ts
enum Status {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
}

@Get()
findByStatus(@Query('status', new ParseEnumPipe(Status)) status: Status) {
  return this.catsService.findByStatus(status);
}
```

**How it works**

- The pipe checks that the `status` query value is one of the enum's values (`active` or `inactive`); anything else returns `400`.
- It is created with `new` because it needs the enum as an argument. Pipes without arguments (like `ParseIntPipe`) can be passed as a class.
- The parameter is typed `Status`, so inside the handler the value is already narrowed to the enum.

### Combining pipes on several parameters

```ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
  @Query('limit', ParseIntPipe) limit: number,
) {
  return this.catsService.findOne(id, sort, limit);
}
```

Each parameter has its own pipe: `id` becomes a number, `sort` a boolean, `limit` a number. They are applied independently, and the **first failure** returns `400` for the whole request. After the pipes, the `typeof` checks all hold.

---

## Validation with ValidationPipe

`ValidationPipe` validates request data automatically using `class-validator` and `class-transformer`.

### Install

```bash
npm i --save class-validator class-transformer
```

### Enable globally (`main.ts`)

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

Registering `ValidationPipe` globally validates **every** incoming `@Body()`, `@Query()` and `@Param()` that is typed with a DTO class. Without this line (or a `@UsePipes` on the controller), the decorators in your DTOs are never checked.

### Define a DTO

```ts
import { IsString, IsInt, IsNotEmpty, IsEmail, Min, Max } from 'class-validator';

export class CreateCatDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsInt()
  @Min(0)
  @Max(20)
  age: number;

  @IsString()
  breed: string;
}

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @IsNotEmpty()
  name: string;
}
```

**What each rule enforces**

| Property | Rules | Rejected examples |
|---|---|---|
| `name` | non-empty string | `""`, `123`, missing |
| `age` | integer between 0 and 20 | `-1`, `21`, `3.5`, `"3"` (a string) |
| `breed` | string | `5`, missing |
| `email` (`CreateUserDto`) | valid email address | `"not-an-email"` |

Decorators on one property are combined with AND: all must pass. A failed request returns `400` with one message per broken rule, for example `age must not be greater than 20`.

### Use it in a controller

```ts
@Post()
create(@Body() createCatDto: CreateCatDto) {
  // Validated automatically; failure throws BadRequestException
  return this.catsService.create(createCatDto);
}
```

By the time `create` runs, the body has already passed the DTO rules, so the method only deals with valid data. When it fails, `ValidationPipe` throws `BadRequestException` before the handler starts, which your [global exception filter](custom-exception.md) can format.

### Custom ValidationPipe

For reference, how a validation pipe works internally:

```ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

**How it works**

1. `metatype` is the type declared on the parameter (`CreateCatDto`). Nest supplies it through `ArgumentMetadata`.
2. `toValidate` skips plain types (`String`, `Number`, `Object`...), because there are no decorators to check on them.
3. `plainToInstance` turns the raw JSON object into a real `CreateCatDto` instance. Without that step `class-validator` would have no decorators to read.
4. `validate(object)` runs every decorator and returns a list of errors.
5. Any error throws `BadRequestException`; otherwise the original value is returned.

This is a simplified version of what the built-in `ValidationPipe` does. The built-in one also supports `whitelist`, `transform` and detailed error messages, so use it in real code.

---

## class-validator decorators

| Type | Decorators |
|------|-----------|
| String | `@IsString()` `@IsNotEmpty()` `@MinLength(n)` `@MaxLength(n)` `@IsEmail()` `@IsUrl()` `@IsAlpha()` (letters only) `@IsAlphanumeric()` (letters and numbers) |
| Number | `@IsNumber()` `@IsInt()` `@Min(n)` `@Max(n)` `@IsPositive()` `@IsNegative()` |
| Boolean | `@IsBoolean()` |
| Date | `@IsDate()` `@MinDate(date)` |
| Array | `@IsArray()` `@ArrayMinSize(n)` `@ArrayMaxSize(n)` |
| Nested object | `@ValidateNested()` + `@Type(() => Dto)` from `class-transformer` |
| Optional | `@IsOptional()` (skips the other validators when the value is missing) |

```ts
@IsString()
@IsNotEmpty()
@MinLength(3)
@MaxLength(50)
name: string;

@IsInt()
@Min(0)
@Max(100)
age: number;

@IsArray()
@ArrayMinSize(1)
@ArrayMaxSize(10)
tags: string[];

@ValidateNested()
@Type(() => AddressDto)
address: AddressDto;

@IsOptional()
@IsString()
description?: string;
```

**How it works**

- Decorators are stacked on one property and **all** must pass.
- Choose the family that matches the type: string rules (`@MinLength`, `@IsEmail`), number rules (`@Min`, `@IsPositive`), `@IsBoolean`, `@IsDate`, array rules (`@ArrayMinSize`).
- `@ValidateNested()` validates an object inside the DTO, and it needs `@Type(() => AddressDto)` so `class-transformer` can build that inner object.
- `@IsOptional()` skips the remaining checks when the value is `undefined` or `null`; without it a missing property fails every rule on it.

---

## Complete controller example

```ts
import {
  Controller, Get, Post, Put, Delete,
  Body, Param, Query,
  ParseIntPipe, HttpCode, HttpStatus,
} from '@nestjs/common';
import { CatsService } from './cats.service';
import { CreateCatDto, UpdateCatDto } from './dto';

@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createCatDto: CreateCatDto) {
    return this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(
    @Query('limit', ParseIntPipe) limit: number,
    @Query('offset', ParseIntPipe) offset: number,
  ) {
    return this.catsService.findAll(limit, offset);
  }

  @Get(':id')
  async findOne(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.findOne(id);
  }

  @Put(':id')
  async update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateCatDto: UpdateCatDto,
  ) {
    return this.catsService.update(id, updateCatDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id', ParseIntPipe) id: number) {
    return this.catsService.remove(id);
  }
}
```

| Route | Handler | Status |
|-------|---------|--------|
| `POST /cats` | `create` | 201 |
| `GET /cats?limit=&offset=` | `findAll` | 200 |
| `GET /cats/:id` | `findOne` | 200 |
| `PUT /cats/:id` | `update` | 200 |
| `DELETE /cats/:id` | `remove` | 204 |

---

## Best practices

1. **Always use DTOs** for request body validation.
2. **Enable `ValidationPipe` globally** in production.
3. **Use specific pipes** (`ParseIntPipe`, ...) for route parameters.
4. **Avoid `@Res()`** unless low-level response control is needed.
5. **Use meaningful HTTP status codes** for each scenario.
6. **Keep controllers thin**: delegate business logic to services.
7. **Use proper TypeScript types** for type safety.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Reading `@Param('id')` as a number without a pipe | Route params are strings, so `id === 1` is false | Use `ParseIntPipe` (or type the DTO with `@Type(() => Number)`) |
| Using `@Res()` and also returning a value | In library-specific mode Nest does not send the return value, so the request hangs | Send the response yourself, use `@Res({ passthrough: true })`, or drop `@Res()` |
| Typing the body as an `interface` | Interfaces do not exist at runtime, so no validation runs | Use a DTO **class** with `class-validator` decorators |
| `ValidationPipe` not registered | The validation decorators are ignored | `app.useGlobalPipes(new ValidationPipe())` |
| Expecting DTO defaults (`page = 1`) or number conversion without `transform: true` | Values stay as sent (strings) and defaults are not applied | `new ValidationPipe({ transform: true })` |
| Guard written but never applied | The route is open | Apply it at method, controller or global level |
