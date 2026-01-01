# NestJS Controllers

Controllers are responsible for handling incoming requests and returning responses to the client. They define routes and handle HTTP methods.

## Table of Contents
- [Basic Controller Setup](#basic-controller-setup)
- [HTTP Methods/Verbs](#http-methodsverbs)
- [Route Parameters](#route-parameters)
- [Query Parameters](#query-parameters)
- [Request Body](#request-body)
- [Complete CRUD Example](#complete-crud-example)
- [HTTP Status Codes](#http-status-codes)
- [Response Headers](#response-headers)
- [Guards](#guards)
- [Pipes](#pipes)
- [Validation with ValidationPipe](#validation-with-validationpipe)
- [Built-in Pipes](#built-in-pipes)
- [Request Object Decorators](#request-object-decorators)

---

## Basic Controller Setup

A controller is created using the `@Controller()` decorator which defines a route prefix for all routes in that controller.

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

**Route:** `GET /cats`

---

## HTTP Methods/Verbs

NestJS provides decorators for all standard HTTP methods:

### @Get()
Handles GET requests - used for retrieving data.

```typescript
@Get()
findAll() {
  return 'This action returns all cats';
}
```

### @Post()
Handles POST requests - used for creating new resources.

```typescript
@Post()
create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

### @Put()
Handles PUT requests - used for full updates of resources.

```typescript
@Put(':id')
update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
  return `This action updates a #${id} cat`;
}
```

### @Patch()
Handles PATCH requests - used for partial updates of resources.

```typescript
@Patch(':id')
partialUpdate(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
  return `This action partially updates a #${id} cat`;
}
```

### @Delete()
Handles DELETE requests - used for deleting resources.

```typescript
@Delete(':id')
remove(@Param('id') id: string) {
  return `This action removes a #${id} cat`;
}
```

---

## Route Parameters

Use `@Param()` decorator to extract dynamic route parameters (path variables).

### Single Parameter

```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  return `This action returns a #${id} cat`;
}
```

**Route:** `GET /cats/123` → id = "123"

### Multiple Parameters

```typescript
@Get(':category/:id')
findByCategoryAndId(
  @Param('category') category: string,
  @Param('id') id: string
) {
  return `Category: ${category}, ID: ${id}`;
}
```

**Route:** `GET /cats/persian/123` → category = "persian", id = "123"

### All Parameters at Once

```typescript
@Get(':id')
findOne(@Param() params: any) {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

---

## Query Parameters

Use `@Query()` decorator to extract query string parameters.

### Single Query Parameter

```typescript
@Get()
findAll(@Query('limit') limit: string) {
  return `This action returns all cats (limit: ${limit} items)`;
}
```

**Route:** `GET /cats?limit=10` → limit = "10"

### Multiple Query Parameters

```typescript
@Get()
findAll(
  @Query('limit') limit: string,
  @Query('offset') offset: string
) {
  return `Limit: ${limit}, Offset: ${offset}`;
}
```

**Route:** `GET /cats?limit=10&offset=20`

### All Query Parameters at Once

```typescript
@Get()
findAll(@Query() query: ListAllEntities) {
  return `This action returns all cats (limit: ${query.limit} items)`;
}
```

---

## Request Body

Use `@Body()` decorator to extract the request body, typically used with POST, PUT, and PATCH requests.

### Full Body

```typescript
@Post()
create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

### Specific Body Properties

```typescript
@Post()
create(
  @Body('name') name: string,
  @Body('age') age: number
) {
  return `Creating cat: ${name}, age: ${age}`;
}
```

### With Type Definition

```typescript
@Post('user')
async signupUser(
  @Body() userData: { name?: string; email: string }
) {
  return this.userService.createUser(userData);
}
```

---

## Complete CRUD Example

Here's a complete controller with all CRUD operations:

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Body,
  Param,
  Query
} from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  // CREATE
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  // READ ALL
  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  // READ ONE
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  // UPDATE (full replacement)
  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  // UPDATE (partial)
  @Patch(':id')
  partialUpdate(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action partially updates a #${id} cat`;
  }

  // DELETE
  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

---

## HTTP Status Codes

By default, NestJS returns:
- **200** for GET, PUT, PATCH, DELETE
- **201** for POST

### Custom Status Code with @HttpCode()

```typescript
import { HttpCode } from '@nestjs/common';

@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

### Using @Res() for Manual Control

```typescript
import { Controller, Post, Res, HttpStatus } from '@nestjs/common';
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

### Common HTTP Status Codes

```typescript
import { HttpStatus } from '@nestjs/common';

// Success
HttpStatus.OK                    // 200
HttpStatus.CREATED               // 201
HttpStatus.ACCEPTED              // 202
HttpStatus.NO_CONTENT            // 204

// Client Errors
HttpStatus.BAD_REQUEST           // 400
HttpStatus.UNAUTHORIZED          // 401
HttpStatus.FORBIDDEN             // 403
HttpStatus.NOT_FOUND             // 404
HttpStatus.CONFLICT              // 409

// Server Errors
HttpStatus.INTERNAL_SERVER_ERROR // 500
HttpStatus.NOT_IMPLEMENTED       // 501
HttpStatus.BAD_GATEWAY           // 502
```

---

## Response Headers

### Custom Header with @Header()

```typescript
import { Header } from '@nestjs/common';

@Post()
@Header('Cache-Control', 'no-store')
create() {
  return 'This action adds a new cat';
}
```

### Multiple Headers

```typescript
@Post()
@Header('Cache-Control', 'no-store')
@Header('X-Custom-Header', 'custom-value')
create() {
  return 'This action adds a new cat';
}
```

---

## Guards

Guards are used to protect routes by determining whether a request should be handled or not. They're commonly used for authentication and authorization.

### Method-Level Guard

Apply a guard to a specific route:

```typescript
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

### Controller-Level Guard

Apply a guard to all routes in a controller:

```typescript
import { Controller, Get, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from './auth.guard';

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

### Multiple Guards

Apply multiple guards (they execute in order):

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { RolesGuard } from './roles.guard';

@Controller('cats')
export class CatsController {
  @Get()
  @UseGuards(AuthGuard, RolesGuard)
  findAll() {
    return 'Protected by both guards';
  }
}
```

### Global Guard

Apply a guard to all routes in the application.

In `main.ts`:
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AuthGuard } from './auth.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalGuards(new AuthGuard());
  await app.listen(3000);
}
bootstrap();
```

Or with dependency injection in `app.module.ts`:
```typescript
import { Module } from '@nestjs/core';
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

---

## Pipes

Pipes are used to transform or validate data before it reaches the route handler. They operate on the arguments being processed by a controller route handler.

### Using Pipes on Parameters

```typescript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return `This action returns cat with id: ${id}`;
  // id is automatically converted to a number
}
```

---

## Validation with ValidationPipe

ValidationPipe provides automatic validation using the `class-validator` and `class-transformer` packages.

### Installation

```bash
npm i --save class-validator class-transformer
```

### Global ValidationPipe Setup

In your `main.ts`:

```typescript
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

### Creating a DTO with Validation

```typescript
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

### Using Validated DTOs in Controllers

```typescript
@Post()
create(@Body() createCatDto: CreateCatDto) {
  // createCatDto is automatically validated
  // If validation fails, BadRequestException is thrown
  return this.catsService.create(createCatDto);
}
```

### Custom ValidationPipe Implementation

```typescript
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException
} from '@nestjs/common';
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

---

## Built-in Pipes

NestJS provides several built-in pipes for common transformations:

### ParseIntPipe
Converts string to integer.

```typescript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  console.log(typeof id === 'number'); // true
  return this.catsService.findOne(id);
}
```

### ParseBoolPipe
Converts string to boolean.

```typescript
@Get()
findAll(@Query('active', ParseBoolPipe) active: boolean) {
  console.log(typeof active === 'boolean'); // true
  return this.catsService.findAll(active);
}
```

### ParseArrayPipe
Converts string to array.

```typescript
@Get()
findByIds(@Query('ids', ParseArrayPipe) ids: number[]) {
  return this.catsService.findByIds(ids);
}
```

### ParseUUIDPipe
Validates and parses UUID strings.

```typescript
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {
  return this.catsService.findOne(id);
}
```

### ParseEnumPipe
Validates that a value is part of an enum.

```typescript
enum Status {
  ACTIVE = 'active',
  INACTIVE = 'inactive'
}

@Get()
findByStatus(@Query('status', new ParseEnumPipe(Status)) status: Status) {
  return this.catsService.findByStatus(status);
}
```

### ParseFloatPipe
Converts string to float.

```typescript
@Get()
findByPrice(@Query('price', ParseFloatPipe) price: number) {
  return this.catsService.findByPrice(price);
}
```

### Combining Multiple Pipes

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
  @Query('limit', ParseIntPipe) limit: number
) {
  console.log(typeof id === 'number');     // true
  console.log(typeof sort === 'boolean');  // true
  console.log(typeof limit === 'number');  // true
  return this.catsService.findOne(id, sort, limit);
}
```

---

## Request Object Decorators

NestJS provides decorators to access different parts of the request:

| Decorator | Description | Express Equivalent |
|-----------|-------------|-------------------|
| `@Request()`, `@Req()` | Full request object | `req` |
| `@Response()`, `@Res()` | Full response object | `res` |
| `@Next()` | Next middleware function | `next` |
| `@Session()` | Session object | `req.session` |
| `@Param(key?: string)` | Route parameters | `req.params` / `req.params[key]` |
| `@Body(key?: string)` | Request body | `req.body` / `req.body[key]` |
| `@Query(key?: string)` | Query parameters | `req.query` / `req.query[key]` |
| `@Headers(name?: string)` | Request headers | `req.headers` / `req.headers[name]` |
| `@Ip()` | Client IP address | `req.ip` |
| `@HostParam()` | Host parameters | `req.hosts` |

### Examples

```typescript
@Get()
findAll(
  @Req() request: Request,
  @Query() query: any,
  @Headers('user-agent') userAgent: string,
  @Ip() ip: string
) {
  console.log('User Agent:', userAgent);
  console.log('IP:', ip);
  return this.catsService.findAll(query);
}
```

---

## Common Validation Decorators

Here are commonly used `class-validator` decorators:

### String Validators
```typescript
@IsString()
@IsNotEmpty()
@MinLength(3)
@MaxLength(50)
@IsEmail()
@IsUrl()
@IsAlpha()        // only letters
@IsAlphanumeric() // letters and numbers
name: string;
```

### Number Validators
```typescript
@IsNumber()
@IsInt()
@Min(0)
@Max(100)
@IsPositive()
@IsNegative()
age: number;
```

### Boolean Validators
```typescript
@IsBoolean()
isActive: boolean;
```

### Date Validators
```typescript
@IsDate()
@MinDate(new Date())
createdAt: Date;
```

### Array Validators
```typescript
@IsArray()
@ArrayMinSize(1)
@ArrayMaxSize(10)
tags: string[];
```

### Nested Object Validation
```typescript
@ValidateNested()
@Type(() => AddressDto)
address: AddressDto;
```

### Optional Fields
```typescript
@IsOptional()
@IsString()
description?: string;
```

---

## Best Practices

1. **Always use DTOs** for request body validation
2. **Enable ValidationPipe globally** in production
3. **Use specific pipes** (ParseIntPipe, etc.) for route parameters
4. **Avoid using @Res()** unless you need low-level response control
5. **Use meaningful HTTP status codes** for different scenarios
6. **Keep controllers thin** - delegate business logic to services
7. **Use proper TypeScript types** for better type safety

---

## Example: Complete Controller with Validation

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  ParseIntPipe,
  HttpCode,
  HttpStatus
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
    @Query('offset', ParseIntPipe) offset: number
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
    @Body() updateCatDto: UpdateCatDto
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

This documentation covers all major aspects of NestJS controllers, including routing, HTTP methods, parameter handling, validation, and pipes.
