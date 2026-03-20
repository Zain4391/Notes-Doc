### Custom Exceptions in NestJS

Custom exceptions provide more control over error responses and allow you to create domain-specific error handling throughout your application.

## Architecture Overview (Similar to Spring's @RestControllerAdvice)

This pattern follows Spring Boot's exception handling approach:

**Spring Boot Pattern:**

- Services throw exceptions
- Controllers let exceptions bubble up (no try-catch)
- `@RestControllerAdvice` catches and formats all exceptions globally

**NestJS Equivalent:**

- Services throw custom exceptions
- Controllers let exceptions bubble up (no try-catch needed)
- Exception Filters (like `@Catch()`) act as `@RestControllerAdvice` to catch and format exceptions globally

**Flow:**

```
Controller → Service → throws CustomException
     ↓
Exception Filter catches exception
     ↓
Formatted JSON response sent to client
```

## Why Use Custom Exceptions

1. Consistent error responses across your API
2. Better error messages for specific business logic failures
3. Easier to test and maintain
4. Type-safe error handling
5. Centralized error logic

## Creating Custom Exceptions

### Basic Custom Exception

**src/common/exceptions/custom-base.exception.ts**

```ts
import { HttpException, HttpStatus } from "@nestjs/common";

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

### Domain-Specific Custom Exceptions

**src/common/exceptions/auth.exceptions.ts**

```ts
import { HttpStatus } from "@nestjs/common";
import { CustomBaseException } from "./custom-base.exception";

export class InvalidCredentialsException extends CustomBaseException {
  constructor() {
    super("Invalid email or password", HttpStatus.UNAUTHORIZED);
  }
}

export class EmailAlreadyExistsException extends CustomBaseException {
  constructor(email: string) {
    super(`Email ${email} is already registered`, HttpStatus.CONFLICT);
  }
}

export class TokenExpiredException extends CustomBaseException {
  constructor() {
    super(
      "Your session has expired. Please login again",
      HttpStatus.UNAUTHORIZED,
    );
  }
}

export class InvalidTokenException extends CustomBaseException {
  constructor() {
    super("Invalid authentication token", HttpStatus.UNAUTHORIZED);
  }
}

export class UnauthorizedAccessException extends CustomBaseException {
  constructor(resource?: string) {
    super(
      resource
        ? `You are not authorized to access ${resource}`
        : "You are not authorized to perform this action",
      HttpStatus.FORBIDDEN,
    );
  }
}
```

**src/common/exceptions/customer.exceptions.ts**

```ts
import { HttpStatus } from "@nestjs/common";
import { CustomBaseException } from "./custom-base.exception";

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

**src/common/exceptions/driver.exceptions.ts**

```ts
import { HttpStatus } from "@nestjs/common";
import { CustomBaseException } from "./custom-base.exception";

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

## Exception Handler Setup (Like @RestControllerAdvice)

### Create the Global Exception Handler

**src/common/filters/http-exception.filter.ts**

```ts
import {
  Catch,
  ExceptionFilter,
  ArgumentsHost,
  HttpException,
  Logger,
} from "@nestjs/common";
import { Response, Request } from "express";

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    let message: string | string[] = "Internal Server Error";
    let errorResponse: Record<string, unknown> | undefined;

    // custom exception handling
    if (typeof exceptionResponse === "object" && exceptionResponse !== null) {
      errorResponse = exceptionResponse as Record<string, unknown>;
      const respMessage = (errorResponse as { message?: string | string[] })
        .message;
      message = respMessage || message;
    } else if (typeof exceptionResponse === "string") {
      message = exceptionResponse;
    } else if (exception instanceof Error) {
      message = exception.message;
      this.logger.error(
        `Unhandled error: ${exception.message}`,
        exception.stack,
      );
    }

    const messageStr = Array.isArray(message) ? message.join(", ") : message;

    this.logger.error(
      `${request.method} ${request.url} - Status: ${status} - Message: ${messageStr}`,
    );

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

### Register Globally in Main.ts

**src/main.ts**

```ts
import { NestFactory } from "@nestjs/core";
import { ValidationPipe } from "@nestjs/common";
import { AppModule } from "./app.module";
import { GlobalExceptionFilter } from "./common/filters/http-exception.filter";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Register global exception filter (like @RestControllerAdvice)
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Optional: Add validation pipe for DTO validation
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

## Usage Pattern: Services Throw, Controllers Let Bubble Up

### In Auth Service (Services throw exceptions)

**src/auth/auth.service.ts**

```ts
import { Injectable } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import * as bcrypt from "bcrypt";
import { Customer } from "../customers/entities/customer.entity";
import { DeliveryDriver } from "../delivery-drivers/entities/delivery-driver.entity";
import { LoginDto } from "./dto/login.dto";
import { RegisterCustomerDto, RegisterDriverDto } from "./dto/register.dto";
import {
  InvalidCredentialsException,
  EmailAlreadyExistsException,
} from "../common/exceptions/auth.exceptions";
import { CustomerAlreadyExistsException } from "../common/exceptions/customer.exceptions";
import { DriverAlreadyExistsException } from "../common/exceptions/driver.exceptions";

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

    const isPasswordValid = await bcrypt.compare(
      loginDto.password,
      customer.password,
    );

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

    const isPasswordValid = await bcrypt.compare(
      loginDto.password,
      driver.password,
    );

    if (!isPasswordValid) {
      throw new InvalidCredentialsException();
    }

    // Generate token...
  }
}
```

### In JWT Strategies

**src/auth/strategies/jwt-customer.strategy.ts**

```ts
import { Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { jwtConstants } from "../constants";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Customer } from "../../customers/entities/customer.entity";
import { InvalidTokenException } from "../../common/exceptions/auth.exceptions";
import { CustomerNotFoundException } from "../../common/exceptions/customer.exceptions";

@Injectable()
export class JwtCustomerStrategy extends PassportStrategy(
  Strategy,
  "jwt-customer",
) {
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
      userType: "customer",
    };
  }
}
```

**src/auth/strategies/jwt-driver.strategy.ts**

```ts
import { Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { jwtConstants } from "../constants";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { DeliveryDriver } from "../../delivery-drivers/entities/delivery-driver.entity";
import { InvalidTokenException } from "../../common/exceptions/auth.exceptions";
import { DriverNotFoundException } from "../../common/exceptions/driver.exceptions";

@Injectable()
export class JwtDriverStrategy extends PassportStrategy(
  Strategy,
  "jwt-driver",
) {
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

    const driver = await this.driverRepository.findOne({
      where: { id: payload.sub },
    });

    if (!driver) {
      throw new DriverNotFoundException(payload.sub);
    }

    return {
      id: driver.id,
      email: driver.email,
      name: driver.name,
      userType: "driver",
    };
  }
}
```

### In Customers Service

**src/customers/customers.service.ts**

```ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Customer } from "./entities/customer.entity";
import {
  CustomerNotFoundException,
  InvalidCustomerDataException,
} from "../common/exceptions/customer.exceptions";

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
    const customer = await this.findOne(id); // Uses the exception from findOne

    if (updateData.email && updateData.email !== customer.email) {
      const emailExists = await this.customerRepository.findOne({
        where: { email: updateData.email },
      });

      if (emailExists) {
        throw new InvalidCustomerDataException("Email already in use");
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

### In Delivery Drivers Service

**src/delivery-drivers/delivery-drivers.service.ts**

```ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { DeliveryDriver } from "./entities/delivery-driver.entity";
import {
  DriverNotFoundException,
  DriverNotAvailableException,
  InvalidVehicleTypeException,
} from "../common/exceptions/driver.exceptions";

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
      .createQueryBuilder("driver")
      .where("driver.is_available = :isAvailable", { isAvailable: true });

    if (vehicleType) {
      const validVehicleTypes = ["bike", "car", "van", "truck"];
      if (!validVehicleTypes.includes(vehicleType.toLowerCase())) {
        throw new InvalidVehicleTypeException(vehicleType);
      }
      query.andWhere("driver.vehicle_type = :vehicleType", { vehicleType });
    }

    const driver = await query.getOne();

    if (!driver) {
      throw new DriverNotAvailableException("any");
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

### In Controllers (No Try-Catch Needed)

Controllers simply call service methods. Exceptions bubble up to the global exception filter automatically.

**src/auth/auth.controller.ts**

```ts
import {
  Controller,
  Post,
  Body,
  HttpCode,
  HttpStatus,
  UseInterceptors,
  ClassSerializerInterceptor,
} from "@nestjs/common";
import { AuthService } from "./auth.service";
import { LoginDto } from "./dto/login.dto";
import { RegisterCustomerDto, RegisterDriverDto } from "./dto/register.dto";

@Controller("auth")
@UseInterceptors(ClassSerializerInterceptor)
export class AuthController {
  constructor(private authService: AuthService) {}

  // No try-catch needed! Exceptions automatically caught by GlobalExceptionFilter
  @Post("customer/register")
  async registerCustomer(@Body() registerDto: RegisterCustomerDto) {
    return this.authService.registerCustomer(registerDto);
    // If service throws CustomerAlreadyExistsException,
    // GlobalExceptionFilter catches and formats it
  }

  @HttpCode(HttpStatus.OK)
  @Post("customer/login")
  async loginCustomer(@Body() loginDto: LoginDto) {
    return this.authService.loginCustomer(loginDto);
    // If service throws InvalidCredentialsException,
    // GlobalExceptionFilter catches and formats it
  }

  @Post("driver/register")
  async registerDriver(@Body() registerDto: RegisterDriverDto) {
    return this.authService.registerDriver(registerDto);
  }

  @HttpCode(HttpStatus.OK)
  @Post("driver/login")
  async loginDriver(@Body() loginDto: LoginDto) {
    return this.authService.loginDriver(loginDto);
  }
}
```

**src/customers/customers.controller.ts**

```ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  UseGuards,
} from "@nestjs/common";
import { CustomersService } from "./customers.service";
import { JwtCustomerGuard } from "../auth/guards/jwt-customer.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Controller("customers")
@UseGuards(JwtCustomerGuard)
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Get(":id")
  async findOne(@Param("id") id: string) {
    return this.customersService.findOne(id);
    // If customer not found, service throws CustomerNotFoundException
    // GlobalExceptionFilter catches it and returns 404 with proper format
  }

  @Patch(":id")
  async update(@Param("id") id: string, @Body() updateData: any) {
    return this.customersService.update(id, updateData);
    // Service may throw CustomerNotFoundException or InvalidCustomerDataException
    // No try-catch needed - GlobalExceptionFilter handles it
  }

  @Delete(":id")
  async remove(@Param("id") id: string) {
    return this.customersService.remove(id);
  }
}
```

**src/delivery-drivers/delivery-drivers.controller.ts**

```ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  UseGuards,
} from "@nestjs/common";
import { DeliveryDriversService } from "./delivery-drivers.service";
import { JwtDriverGuard } from "../auth/guards/jwt-driver.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Controller("drivers")
export class DeliveryDriversController {
  constructor(private readonly driversService: DeliveryDriversService) {}

  @Get("available")
  async findAvailable(@Query("vehicleType") vehicleType?: string) {
    return this.driversService.findAvailableDriver(vehicleType);
    // May throw InvalidVehicleTypeException or DriverNotAvailableException
    // GlobalExceptionFilter handles formatting
  }

  @UseGuards(JwtDriverGuard)
  @Patch(":id/availability")
  async updateAvailability(
    @Param("id") id: string,
    @Body("isAvailable") isAvailable: boolean,
  ) {
    return this.driversService.updateAvailability(id, isAvailable);
    // May throw DriverNotFoundException
  }

  @Post(":id/assign-order")
  async assignOrder(
    @Param("id") driverId: string,
    @Body("orderId") orderId: string,
  ) {
    return this.driversService.assignOrder(driverId, orderId);
    // May throw DriverNotFoundException or DriverNotAvailableException
  }
}
```

### In Guards

**src/auth/guards/jwt-customer.guard.ts**

```ts
import { Injectable, ExecutionContext } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
import { UnauthorizedAccessException } from "../../common/exceptions/auth.exceptions";

@Injectable()
export class JwtCustomerGuard extends AuthGuard("jwt-customer") {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    if (err || !user) {
      throw new UnauthorizedAccessException("customer resources");
    }
    return user;
  }
}
```

**src/auth/guards/jwt-driver.guard.ts**

```ts
import { Injectable, ExecutionContext } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
import { UnauthorizedAccessException } from "../../common/exceptions/auth.exceptions";

@Injectable()
export class JwtDriverGuard extends AuthGuard("jwt-driver") {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    if (err || !user) {
      throw new UnauthorizedAccessException("driver resources");
    }
    return user;
  }
}
```

## Exception Handler Setup (Like @RestControllerAdvice)

### Create the Global Exception Handler

This is the NestJS equivalent of Spring's `@RestControllerAdvice`. It catches all exceptions thrown anywhere in the application and formats them consistently.

**src/common/filters/http-exception.filter.ts**

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from "@nestjs/common";
import { Request, Response } from "express";

@Catch() // Catches all exceptions (like @RestControllerAdvice)
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message: string | string[] = "Internal server error";
    let errorResponse: any;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      // Handle custom exception format
      if (typeof exceptionResponse === "object" && exceptionResponse !== null) {
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
      this.logger.error(
        `Unhandled error: ${exception.message}`,
        exception.stack,
      );
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

### Register Globally in Main.ts

**src/main.ts**

```ts
import { NestFactory } from "@nestjs/core";
import { ValidationPipe } from "@nestjs/common";
import { AppModule } from "./app.module";
import { GlobalExceptionFilter } from "./common/filters/http-exception.filter";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Register global exception filter (like @RestControllerAdvice)
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Optional: Add validation pipe for DTO validation
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

## Usage Pattern: Services Throw, Controllers Let Bubble Up

### In Controllers (No Try-Catch Needed)

Controllers simply call service methods. Exceptions bubble up to the global exception filter automatically.

**src/auth/auth.controller.ts**

```ts
import {
  Controller,
  Post,
  Body,
  HttpCode,
  HttpStatus,
  UseInterceptors,
  ClassSerializerInterceptor,
} from "@nestjs/common";
import { AuthService } from "./auth.service";
import { LoginDto } from "./dto/login.dto";
import { RegisterCustomerDto, RegisterDriverDto } from "./dto/register.dto";

@Controller("auth")
@UseInterceptors(ClassSerializerInterceptor)
export class AuthController {
  constructor(private authService: AuthService) {}

  // No try-catch needed! Exceptions automatically caught by GlobalExceptionFilter
  @Post("customer/register")
  async registerCustomer(@Body() registerDto: RegisterCustomerDto) {
    return this.authService.registerCustomer(registerDto);
    // If service throws CustomerAlreadyExistsException,
    // GlobalExceptionFilter catches and formats it
  }

  @HttpCode(HttpStatus.OK)
  @Post("customer/login")
  async loginCustomer(@Body() loginDto: LoginDto) {
    return this.authService.loginCustomer(loginDto);
    // If service throws InvalidCredentialsException,
    // GlobalExceptionFilter catches and formats it
  }

  @Post("driver/register")
  async registerDriver(@Body() registerDto: RegisterDriverDto) {
    return this.authService.registerDriver(registerDto);
  }

  @HttpCode(HttpStatus.OK)
  @Post("driver/login")
  async loginDriver(@Body() loginDto: LoginDto) {
    return this.authService.loginDriver(loginDto);
  }
}
```

**src/customers/customers.controller.ts**

```ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  UseGuards,
} from "@nestjs/common";
import { CustomersService } from "./customers.service";
import { JwtCustomerGuard } from "../auth/guards/jwt-customer.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Controller("customers")
@UseGuards(JwtCustomerGuard)
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Get(":id")
  async findOne(@Param("id") id: string) {
    return this.customersService.findOne(id);
    // If customer not found, service throws CustomerNotFoundException
    // GlobalExceptionFilter catches it and returns 404 with proper format
  }

  @Patch(":id")
  async update(@Param("id") id: string, @Body() updateData: any) {
    return this.customersService.update(id, updateData);
    // Service may throw CustomerNotFoundException or InvalidCustomerDataException
    // No try-catch needed - GlobalExceptionFilter handles it
  }

  @Delete(":id")
  async remove(@Param("id") id: string) {
    return this.customersService.remove(id);
  }
}
```

**src/delivery-drivers/delivery-drivers.controller.ts**

```ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Query,
  UseGuards,
} from "@nestjs/common";
import { DeliveryDriversService } from "./delivery-drivers.service";
import { JwtDriverGuard } from "../auth/guards/jwt-driver.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Controller("drivers")
export class DeliveryDriversController {
  constructor(private readonly driversService: DeliveryDriversService) {}

  @Get("available")
  async findAvailable(@Query("vehicleType") vehicleType?: string) {
    return this.driversService.findAvailableDriver(vehicleType);
    // May throw InvalidVehicleTypeException or DriverNotAvailableException
    // GlobalExceptionFilter handles formatting
  }

  @UseGuards(JwtDriverGuard)
  @Patch(":id/availability")
  async updateAvailability(
    @Param("id") id: string,
    @Body("isAvailable") isAvailable: boolean,
  ) {
    return this.driversService.updateAvailability(id, isAvailable);
    // May throw DriverNotFoundException
  }

  @Post(":id/assign-order")
  async assignOrder(
    @Param("id") driverId: string,
    @Body("orderId") orderId: string,
  ) {
    return this.driversService.assignOrder(driverId, orderId);
    // May throw DriverNotFoundException or DriverNotAvailableException
  }
}
```

### In Services (Services Throw Exceptions)

Services focus on business logic and throw domain-specific exceptions when things go wrong.

**src/auth/auth.service.ts**

(Note: Service examples are shown earlier in the file with complete implementations)

## Example Error Responses

When exceptions are thrown, the GlobalExceptionFilter formats them consistently:

### Customer Not Found (404)

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

### Invalid Credentials (401)

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

### Email Already Exists (409)

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

### Driver Not Available (409)

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

### Unauthorized Access (403)

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

## `GlobalExceptionFilter` — Catch All Errors Safely

NestJS `@Catch()` (no argument) catches everything — including plain `Error` and `TypeError`, not just `HttpException`. Always check `instanceof` before calling `HttpException` methods:

```ts
// ❌ crashes when a plain TypeError is thrown
catch(exception: HttpException, host: ArgumentsHost) {
  const status = exception.getStatus(); // TypeError: getStatus is not a function
```

```ts
// ✅ correct
catch(exception: unknown, host: ArgumentsHost) {
  let status = HttpStatus.INTERNAL_SERVER_ERROR;
  let message = "Internal Server Error";

  if (exception instanceof HttpException) {
    status = exception.getStatus();
    // ...
  } else if (exception instanceof Error) {
    message = exception.message;
    this.logger.error(exception.message, exception.stack);
  }
```

## Benefits of This Approach (@RestControllerAdvice Pattern)

1. **Separation of Concerns** - Services handle business logic and throw exceptions; controllers don't need error handling code
2. **Consistent error responses** - GlobalExceptionFilter (like @RestControllerAdvice) ensures all errors follow the same structure
3. **Type safety** - Custom exceptions are strongly typed
4. **Clean Controllers** - No try-catch blocks cluttering controller methods
5. **Better debugging** - Clear, domain-specific error messages make debugging easier
6. **Maintainable** - Centralized exception logic in one filter, easy to update
7. **Domain-specific** - Exceptions reflect your business logic (CustomerNotFoundException, etc.)
8. **Testable** - Easy to test specific error scenarios in services without mocking error handlers
9. **Spring-like familiarity** - Developers from Spring Boot will find this pattern familiar

## Comparison with Spring Boot

| Spring Boot                 | NestJS Equivalent                                |
| --------------------------- | ------------------------------------------------ |
| `@RestControllerAdvice`     | `@Catch()` Exception Filter                      |
| Service throws exceptions   | Service throws custom exceptions                 |
| `@ExceptionHandler` methods | `catch()` method in filter                       |
| Controller has no try-catch | Controller has no try-catch                      |
| Global exception handling   | Global exception filter                          |
| Custom exception classes    | Custom exception classes extending HttpException |

## Key Takeaways

1. **Create domain-specific exceptions** in `src/common/exceptions/`
2. **Services throw exceptions** when business rules are violated
3. **Controllers don't catch** - let exceptions bubble up
4. **GlobalExceptionFilter** (like @RestControllerAdvice) catches all exceptions
5. **Consistent response format** across the entire API
6. **Easy to maintain** - all exception handling in one place
