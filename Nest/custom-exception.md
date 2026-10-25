# Custom Exceptions and Global Exception Filter

Domain-specific exceptions, a global exception filter that formats every error, and how services, strategies, guards and controllers use them. Mirrors Spring's `@RestControllerAdvice`.

**Contents**

1. [Architecture](#architecture)
2. [Why custom exceptions](#why-custom-exceptions)
3. [Folder layout](#folder-layout)
4. [Creating exceptions](#creating-exceptions)
5. [Global exception filter](#global-exception-filter)
6. [Registering the filter](#registering-the-filter)
7. [Usage: services throw](#usage-services-throw)
8. [Usage: strategies and guards](#usage-strategies-and-guards)
9. [Usage: controllers let it bubble up](#usage-controllers-let-it-bubble-up)
10. [Example error responses](#example-error-responses)
11. [Benefits](#benefits)
12. [Comparison with Spring Boot](#comparison-with-spring-boot)
13. [Key takeaways](#key-takeaways)
14. [Common mistakes](#common-mistakes)

Related: [Controllers](controller.md) · [Services](service.md) · [JWT Authentication](jwt-authentication.md)

---

## Architecture

The pattern mirrors Spring Boot's `@RestControllerAdvice`.

| Spring Boot | NestJS equivalent |
|-------------|-------------------|
| `@RestControllerAdvice` | Exception filter decorated with `@Catch()` |
| `@ExceptionHandler` methods | The `catch()` method of the filter |
| Services throw exceptions | Services throw custom exceptions |
| Controllers have no try-catch | Controllers have no try-catch |
| Custom exception classes | Custom classes extending `HttpException` |
| Global exception handling | `app.useGlobalFilters()` |

**Flow**

```text
Controller -> Service -> throws CustomException
                 |
                 v
     Exception Filter catches it
                 |
                 v
   Formatted JSON response sent to client
```

---

## Why custom exceptions

1. Consistent error responses across the API.
2. Clear messages for specific business-logic failures.
3. Easier to test and maintain.
4. Type-safe error handling.
5. Centralized error logic.

---

## Folder layout

```text
src/
├── common/
│   ├── exceptions/
│   │   ├── custom-base.exception.ts
│   │   ├── auth.exceptions.ts
│   │   ├── customer.exceptions.ts
│   │   └── driver.exceptions.ts
│   └── filters/
│       └── http-exception.filter.ts
└── main.ts
```

---

## Creating exceptions

### Base exception

`src/common/exceptions/custom-base.exception.ts`

All domain exceptions extend this, so every error has the same response shape.

```ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class CustomBaseException extends HttpException {
  constructor(message: string, statusCode: HttpStatus) {
    super(
      {
        success: false,
        error: {
          message,
          statusCode,
          timestamp: new Date().toISOString(),
        },
      },
      statusCode,
    );
  }
}
```

| Parameter | Description |
|-----------|-------------|
| `message` | Human-readable error message. |
| `statusCode` | HTTP status for the response (`HttpStatus` enum). |

### Auth exceptions

`src/common/exceptions/auth.exceptions.ts`

```ts
import { HttpStatus } from '@nestjs/common';
import { CustomBaseException } from './custom-base.exception';

export class InvalidCredentialsException extends CustomBaseException {
  constructor() {
    super('Invalid email or password', HttpStatus.UNAUTHORIZED);
  }
}

export class EmailAlreadyExistsException extends CustomBaseException {
  constructor(email: string) {
    super(`Email ${email} is already registered`, HttpStatus.CONFLICT);
  }
}

export class TokenExpiredException extends CustomBaseException {
  constructor() {
    super('Your session has expired. Please login again', HttpStatus.UNAUTHORIZED);
  }
}

export class InvalidTokenException extends CustomBaseException {
  constructor() {
    super('Invalid authentication token', HttpStatus.UNAUTHORIZED);
  }
}

export class UnauthorizedAccessException extends CustomBaseException {
  constructor(resource?: string) {
    super(
      resource
        ? `You are not authorized to access ${resource}`
        : 'You are not authorized to perform this action',
      HttpStatus.FORBIDDEN,
    );
  }
}
```

### Customer exceptions

`src/common/exceptions/customer.exceptions.ts`

```ts
import { HttpStatus } from '@nestjs/common';
import { CustomBaseException } from './custom-base.exception';

export class CustomerNotFoundException extends CustomBaseException {
  constructor(id: string) {
    super(`Customer with ID ${id} not found`, HttpStatus.NOT_FOUND);
  }
}

export class InvalidCustomerDataException extends CustomBaseException {
  constructor(message: string) {
    super(`Invalid customer data: ${message}`, HttpStatus.BAD_REQUEST);
  }
}

export class CustomerAlreadyExistsException extends CustomBaseException {
  constructor(email: string) {
    super(`Customer with email ${email} already exists`, HttpStatus.CONFLICT);
  }
}
```

### Driver exceptions

`src/common/exceptions/driver.exceptions.ts`

```ts
import { HttpStatus } from '@nestjs/common';
import { CustomBaseException } from './custom-base.exception';

export class DriverNotFoundException extends CustomBaseException {
  constructor(id: string) {
    super(`Driver with ID ${id} not found`, HttpStatus.NOT_FOUND);
  }
}

export class DriverNotAvailableException extends CustomBaseException {
  constructor(id: string) {
    super(`Driver with ID ${id} is not available`, HttpStatus.CONFLICT);
  }
}

export class InvalidVehicleTypeException extends CustomBaseException {
  constructor(vehicleType: string) {
    super(`Invalid vehicle type: ${vehicleType}`, HttpStatus.BAD_REQUEST);
  }
}

export class DriverAlreadyExistsException extends CustomBaseException {
  constructor(email: string) {
    super(`Driver with email ${email} already exists`, HttpStatus.CONFLICT);
  }
}
```

### Exception catalog

| Exception | Status | Message |
|-----------|--------|---------|
| `InvalidCredentialsException` | 401 | Invalid email or password |
| `EmailAlreadyExistsException` | 409 | Email `{email}` is already registered |
| `TokenExpiredException` | 401 | Your session has expired. Please login again |
| `InvalidTokenException` | 401 | Invalid authentication token |
| `UnauthorizedAccessException` | 403 | You are not authorized to access `{resource}` |
| `CustomerNotFoundException` | 404 | Customer with ID `{id}` not found |
| `InvalidCustomerDataException` | 400 | Invalid customer data: `{message}` |
| `CustomerAlreadyExistsException` | 409 | Customer with email `{email}` already exists |
| `DriverNotFoundException` | 404 | Driver with ID `{id}` not found |
| `DriverNotAvailableException` | 409 | Driver with ID `{id}` is not available |
| `InvalidVehicleTypeException` | 400 | Invalid vehicle type: `{vehicleType}` |
| `DriverAlreadyExistsException` | 409 | Driver with email `{email}` already exists |

---

## Global exception filter

`src/common/filters/http-exception.filter.ts`

The NestJS equivalent of `@RestControllerAdvice`: catches every exception thrown anywhere in the application and formats it consistently.

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch() // no argument: catches all exceptions
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message: string | string[] = 'Internal server error';
    let errorResponse: any;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      // Handle custom exception format
      if (typeof exceptionResponse === 'object' && exceptionResponse !== null) {
        errorResponse = exceptionResponse;
        message =
          (exceptionResponse as any).error?.message ||
          (exceptionResponse as any).message ||
          message;
      } else {
        message = exceptionResponse as string;
      }
    } else if (exception instanceof Error) {
      message = exception.message;
      this.logger.error(`Unhandled error: ${exception.message}`, exception.stack);
    }

    // Log the error
    this.logger.error(
      `${request.method} ${request.url} - Status: ${status} - Message: ${message}`,
    );

    // Send formatted response
    response.status(status).json(
      errorResponse || {
        success: false,
        error: {
          statusCode: status,
          message,
          timestamp: new Date().toISOString(),
          path: request.url,
        },
      },
    );
  }
}
```

### How it decides what to send

| Exception type | Status | Response body |
|----------------|--------|---------------|
| `HttpException` with object response (all custom exceptions) | From the exception | The exception's own response (`{ success, error }`). |
| `HttpException` with string response | From the exception | Generic `{ success: false, error: {...} }` wrapper. |
| Plain `Error` / `TypeError` | 500 | Generic wrapper; the message and stack are logged. |
| Anything else | 500 | Generic wrapper with "Internal server error". |

### Catch everything safely

`@Catch()` with no argument catches **everything**, including plain `Error` and `TypeError`, not only `HttpException`. Always check `instanceof` before calling `HttpException` methods.

```ts
// Wrong: crashes when a plain TypeError is thrown
catch(exception: HttpException, host: ArgumentsHost) {
  const status = exception.getStatus(); // TypeError: getStatus is not a function
}
```

```ts
// Correct
catch(exception: unknown, host: ArgumentsHost) {
  let status = HttpStatus.INTERNAL_SERVER_ERROR;
  let message = 'Internal Server Error';

  if (exception instanceof HttpException) {
    status = exception.getStatus();
    // ...
  } else if (exception instanceof Error) {
    message = exception.message;
    this.logger.error(exception.message, exception.stack);
  }
}
```

---

## Registering the filter

`src/main.ts`

```ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './common/filters/http-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Register global exception filter (like @RestControllerAdvice)
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Optional: validation pipe for DTO validation
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

| `ValidationPipe` option | Description |
|-------------------------|-------------|
| `whitelist` | Strips properties that have no validation decorator. |
| `forbidNonWhitelisted` | Rejects the request if unknown properties are sent. |
| `transform` | Converts plain payloads into DTO class instances. |

---

## Usage: services throw

Services hold the business rules and throw a domain exception when one is violated.

### Auth service

`src/auth/auth.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as bcrypt from 'bcrypt';
import { Customer } from '../customers/entities/customer.entity';
import { DeliveryDriver } from '../delivery-drivers/entities/delivery-driver.entity';
import { LoginDto } from './dto/login.dto';
import { RegisterCustomerDto, RegisterDriverDto } from './dto/register.dto';
import { InvalidCredentialsException } from '../common/exceptions/auth.exceptions';
import { CustomerAlreadyExistsException } from '../common/exceptions/customer.exceptions';
import { DriverAlreadyExistsException } from '../common/exceptions/driver.exceptions';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
    private jwtService: JwtService,
  ) {}

  async registerCustomer(registerDto: RegisterCustomerDto) {
    const existingCustomer = await this.customerRepository.findOne({
      where: { email: registerDto.email },
    });

    if (existingCustomer) {
      throw new CustomerAlreadyExistsException(registerDto.email);
    }

    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(registerDto.password, saltRounds);

    const customer = this.customerRepository.create({
      ...registerDto,
      password: hashedPassword,
    });

    return await this.customerRepository.save(customer);
  }

  async loginCustomer(loginDto: LoginDto) {
    const customer = await this.customerRepository.findOne({
      where: { email: loginDto.email },
    });

    if (!customer) {
      throw new InvalidCredentialsException();
    }

    const isPasswordValid = await bcrypt.compare(loginDto.password, customer.password);

    if (!isPasswordValid) {
      throw new InvalidCredentialsException();
    }

    // Generate token...
  }

  async registerDriver(registerDto: RegisterDriverDto) {
    const existingDriver = await this.driverRepository.findOne({
      where: { email: registerDto.email },
    });

    if (existingDriver) {
      throw new DriverAlreadyExistsException(registerDto.email);
    }

    // Rest of registration logic...
  }

  async loginDriver(loginDto: LoginDto) {
    const driver = await this.driverRepository.findOne({
      where: { email: loginDto.email },
    });

    if (!driver) {
      throw new InvalidCredentialsException();
    }

    const isPasswordValid = await bcrypt.compare(loginDto.password, driver.password);

    if (!isPasswordValid) {
      throw new InvalidCredentialsException();
    }

    // Generate token...
  }
}
```

> The same `InvalidCredentialsException` is thrown for "user not found" and "wrong password", so the response never reveals which emails are registered.

### Customers service

`src/customers/customers.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Customer } from './entities/customer.entity';
import {
  CustomerNotFoundException,
  InvalidCustomerDataException,
} from '../common/exceptions/customer.exceptions';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {}

  async findOne(id: string) {
    const customer = await this.customerRepository.findOne({ where: { id } });

    if (!customer) {
      throw new CustomerNotFoundException(id);
    }

    return customer;
  }

  async update(id: string, updateData: any) {
    const customer = await this.findOne(id); // reuses the not-found exception

    if (updateData.email && updateData.email !== customer.email) {
      const emailExists = await this.customerRepository.findOne({
        where: { email: updateData.email },
      });

      if (emailExists) {
        throw new InvalidCustomerDataException('Email already in use');
      }
    }

    Object.assign(customer, updateData);
    return await this.customerRepository.save(customer);
  }

  async remove(id: string) {
    const customer = await this.findOne(id);
    return await this.customerRepository.remove(customer);
  }
}
```

### Delivery drivers service

`src/delivery-drivers/delivery-drivers.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { DeliveryDriver } from './entities/delivery-driver.entity';
import {
  DriverNotFoundException,
  DriverNotAvailableException,
  InvalidVehicleTypeException,
} from '../common/exceptions/driver.exceptions';

@Injectable()
export class DeliveryDriversService {
  constructor(
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
  ) {}

  async findOne(id: string) {
    const driver = await this.driverRepository.findOne({ where: { id } });

    if (!driver) {
      throw new DriverNotFoundException(id);
    }

    return driver;
  }

  async findAvailableDriver(vehicleType?: string) {
    const query = this.driverRepository
      .createQueryBuilder('driver')
      .where('driver.is_available = :isAvailable', { isAvailable: true });

    if (vehicleType) {
      const validVehicleTypes = ['bike', 'car', 'van', 'truck'];
      if (!validVehicleTypes.includes(vehicleType.toLowerCase())) {
        throw new InvalidVehicleTypeException(vehicleType);
      }
      query.andWhere('driver.vehicle_type = :vehicleType', { vehicleType });
    }

    const driver = await query.getOne();

    if (!driver) {
      throw new DriverNotAvailableException('any');
    }

    return driver;
  }

  async updateAvailability(id: string, isAvailable: boolean) {
    const driver = await this.findOne(id);
    driver.is_available = isAvailable;
    return await this.driverRepository.save(driver);
  }

  async assignOrder(driverId: string, orderId: string) {
    const driver = await this.findOne(driverId);

    if (!driver.is_available) {
      throw new DriverNotAvailableException(driverId);
    }

    // Assign order logic...
    driver.is_available = false;
    return await this.driverRepository.save(driver);
  }
}
```

---

## Usage: strategies and guards

Exceptions also work outside services: in Passport strategies (`validate()`) and in guards (`handleRequest()`).

### JWT strategies

`src/auth/strategies/jwt-customer.strategy.ts`

```ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Customer } from '../../customers/entities/customer.entity';
import { InvalidTokenException } from '../../common/exceptions/auth.exceptions';
import { CustomerNotFoundException } from '../../common/exceptions/customer.exceptions';

@Injectable()
export class JwtCustomerStrategy extends PassportStrategy(Strategy, 'jwt-customer') {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.customerSecret,
    });
  }

  async validate(payload: any) {
    if (!payload || !payload.sub) {
      throw new InvalidTokenException();
    }

    const customer = await this.customerRepository.findOne({
      where: { id: payload.sub },
    });

    if (!customer) {
      throw new CustomerNotFoundException(payload.sub);
    }

    return {
      id: customer.id,
      email: customer.email,
      name: customer.name,
      userType: 'customer',
    };
  }
}
```

`src/auth/strategies/jwt-driver.strategy.ts` has the same structure with `DeliveryDriver`, `jwtConstants.driverSecret`, `DriverNotFoundException`, strategy name `'jwt-driver'` and `userType: 'driver'`:

```ts
@Injectable()
export class JwtDriverStrategy extends PassportStrategy(Strategy, 'jwt-driver') {
  constructor(
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.driverSecret,
    });
  }

  async validate(payload: any) {
    if (!payload || !payload.sub) {
      throw new InvalidTokenException();
    }

    const driver = await this.driverRepository.findOne({ where: { id: payload.sub } });

    if (!driver) {
      throw new DriverNotFoundException(payload.sub);
    }

    return { id: driver.id, email: driver.email, name: driver.name, userType: 'driver' };
  }
}
```

### Guards

`src/auth/guards/jwt-customer.guard.ts`

```ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { UnauthorizedAccessException } from '../../common/exceptions/auth.exceptions';

@Injectable()
export class JwtCustomerGuard extends AuthGuard('jwt-customer') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    if (err || !user) {
      throw new UnauthorizedAccessException('customer resources');
    }
    return user;
  }
}
```

`src/auth/guards/jwt-driver.guard.ts` is identical, with `AuthGuard('jwt-driver')`, class name `JwtDriverGuard` and the message `'driver resources'`.

| Hook | Description |
|------|-------------|
| `validate(payload)` (strategy) | Runs after the token is verified; throw to reject, return the object that becomes `req.user`. |
| `handleRequest(err, user, info)` (guard) | Runs after the strategy; replace Passport's default 401 with a custom exception. |

---

## Usage: controllers let it bubble up

Controllers only call services. Exceptions bubble up to `GlobalExceptionFilter`. No try-catch is needed.

### Auth controller

`src/auth/auth.controller.ts`

```ts
import {
  Controller, Post, Body, HttpCode, HttpStatus,
  UseInterceptors, ClassSerializerInterceptor,
} from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';
import { RegisterCustomerDto, RegisterDriverDto } from './dto/register.dto';

@Controller('auth')
@UseInterceptors(ClassSerializerInterceptor)
export class AuthController {
  constructor(private authService: AuthService) {}

  // No try-catch: exceptions are caught by GlobalExceptionFilter
  @Post('customer/register')
  async registerCustomer(@Body() registerDto: RegisterCustomerDto) {
    return this.authService.registerCustomer(registerDto);
    // CustomerAlreadyExistsException -> formatted by the filter
  }

  @HttpCode(HttpStatus.OK)
  @Post('customer/login')
  async loginCustomer(@Body() loginDto: LoginDto) {
    return this.authService.loginCustomer(loginDto);
    // InvalidCredentialsException -> formatted by the filter
  }

  @Post('driver/register')
  async registerDriver(@Body() registerDto: RegisterDriverDto) {
    return this.authService.registerDriver(registerDto);
  }

  @HttpCode(HttpStatus.OK)
  @Post('driver/login')
  async loginDriver(@Body() loginDto: LoginDto) {
    return this.authService.loginDriver(loginDto);
  }
}
```

### Customers controller

`src/customers/customers.controller.ts`

```ts
import { Controller, Get, Body, Patch, Param, Delete, UseGuards } from '@nestjs/common';
import { CustomersService } from './customers.service';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';

@Controller('customers')
@UseGuards(JwtCustomerGuard)
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Get(':id')
  async findOne(@Param('id') id: string) {
    return this.customersService.findOne(id);
    // Not found -> CustomerNotFoundException -> 404 in the standard format
  }

  @Patch(':id')
  async update(@Param('id') id: string, @Body() updateData: any) {
    return this.customersService.update(id, updateData);
    // May throw CustomerNotFoundException or InvalidCustomerDataException
  }

  @Delete(':id')
  async remove(@Param('id') id: string) {
    return this.customersService.remove(id);
  }
}
```

### Delivery drivers controller

`src/delivery-drivers/delivery-drivers.controller.ts`

```ts
import { Controller, Get, Post, Body, Patch, Param, Query, UseGuards } from '@nestjs/common';
import { DeliveryDriversService } from './delivery-drivers.service';
import { JwtDriverGuard } from '../auth/guards/jwt-driver.guard';

@Controller('drivers')
export class DeliveryDriversController {
  constructor(private readonly driversService: DeliveryDriversService) {}

  @Get('available')
  async findAvailable(@Query('vehicleType') vehicleType?: string) {
    return this.driversService.findAvailableDriver(vehicleType);
    // May throw InvalidVehicleTypeException or DriverNotAvailableException
  }

  @UseGuards(JwtDriverGuard)
  @Patch(':id/availability')
  async updateAvailability(
    @Param('id') id: string,
    @Body('isAvailable') isAvailable: boolean,
  ) {
    return this.driversService.updateAvailability(id, isAvailable);
    // May throw DriverNotFoundException
  }

  @Post(':id/assign-order')
  async assignOrder(@Param('id') driverId: string, @Body('orderId') orderId: string) {
    return this.driversService.assignOrder(driverId, orderId);
    // May throw DriverNotFoundException or DriverNotAvailableException
  }
}
```

---

## Example error responses

Every exception goes through the same format.

### Customer not found (404)

```json
{
  "success": false,
  "error": {
    "message": "Customer with ID abc-123 not found",
    "statusCode": 404,
    "timestamp": "2025-12-31T10:30:00.000Z",
    "path": "/customers/abc-123"
  }
}
```

### Invalid credentials (401)

```json
{
  "success": false,
  "error": {
    "message": "Invalid email or password",
    "statusCode": 401,
    "timestamp": "2025-12-31T10:30:00.000Z",
    "path": "/auth/customer/login"
  }
}
```

### Email already exists (409)

```json
{
  "success": false,
  "error": {
    "message": "Customer with email test@example.com already exists",
    "statusCode": 409,
    "timestamp": "2025-12-31T10:30:00.000Z",
    "path": "/auth/customer/register"
  }
}
```

### Driver not available (409)

```json
{
  "success": false,
  "error": {
    "message": "Driver with ID xyz-456 is not available",
    "statusCode": 409,
    "timestamp": "2025-12-31T10:30:00.000Z",
    "path": "/drivers/xyz-456/assign-order"
  }
}
```

### Unauthorized access (403)

```json
{
  "success": false,
  "error": {
    "message": "You are not authorized to access driver resources",
    "statusCode": 403,
    "timestamp": "2025-12-31T10:30:00.000Z",
    "path": "/drivers/profile"
  }
}
```

> **Note:** the `path` field is only present when the filter builds the generic wrapper itself. Custom exceptions send their own body (from `CustomBaseException`), which carries `message`, `statusCode` and `timestamp`. Add `path` to the base exception if you want it in every response.

---

## Benefits

1. **Separation of concerns:** services handle business logic and throw; controllers carry no error handling.
2. **Consistent responses:** the filter guarantees one structure for all errors.
3. **Type safety:** custom exceptions are strongly typed.
4. **Clean controllers:** no try-catch blocks.
5. **Better debugging:** domain-specific messages.
6. **Maintainable:** exception handling lives in one filter.
7. **Domain-specific:** names reflect business rules (`CustomerNotFoundException`).
8. **Testable:** error scenarios can be tested in services directly.
9. **Familiar** to developers coming from Spring Boot.

---

## Comparison with Spring Boot

| Spring Boot | NestJS |
|-------------|--------|
| `@RestControllerAdvice` | `@Catch()` exception filter |
| Service throws exceptions | Service throws custom exceptions |
| `@ExceptionHandler` methods | `catch()` method in the filter |
| Controller has no try-catch | Controller has no try-catch |
| Global exception handling | Global exception filter |
| Custom exception classes | Custom exception classes extending `HttpException` |

---

## Key takeaways

1. Create domain-specific exceptions in `src/common/exceptions/`.
2. Services throw exceptions when business rules are violated.
3. Controllers don't catch: let exceptions bubble up.
4. `GlobalExceptionFilter` catches all exceptions.
5. Type the filter's parameter as `unknown` and check `instanceof HttpException` first.
6. The response format stays consistent across the API.
7. All exception handling stays in one place.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| Typing `catch(exception: HttpException)` on a `@Catch()` filter | A plain `Error` has no `getStatus()` and the filter itself crashes | Type it `unknown` and check `instanceof HttpException` |
| Forgetting `app.useGlobalFilters(...)` | Custom exceptions still work (they extend `HttpException`) but unhandled errors use Nest's default format | Register the filter in `main.ts` |
| Throwing a plain `Error` from a service | Becomes a generic 500 | Throw a domain exception |
| `try/catch` in controllers that swallows the error | The filter never sees it | Let exceptions bubble up |
| Different response shapes per exception type | `ValidationPipe` errors come back as `{ statusCode, message[], error }`, not `{ success, error }`, because the filter passes an object response through unchanged | Normalize inside the filter if clients need a single shape |
| Different messages for "no such user" and "wrong password" | Reveals which emails are registered | Throw the same `InvalidCredentialsException` for both |
