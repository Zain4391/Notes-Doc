# JWT Authentication and Role-Based Access Control

JWT authentication with Passport (separate customer and driver flows, bcrypt password hashing, typed payloads and responses), followed by role-based access control (RBAC) with a roles decorator and guard.

**Contents**

1. [How it works](#how-it-works)
2. [Packages](#packages)
3. [Project structure](#project-structure)
4. [Core authentication files](#core-authentication-files)
5. [JWT strategies and guards](#jwt-strategies-and-guards)
6. [Types and response DTOs](#types-and-response-dtos)
7. [Auth service](#auth-service)
8. [Auth controller](#auth-controller)
9. [Modules](#modules)
10. [RBAC overview](#rbac-overview)
11. [Roles enum](#roles-enum)
12. [Role column on the entities](#role-column-on-the-entities)
13. [Roles decorator](#roles-decorator)
14. [Roles guard](#roles-guard)
15. [Role in the JWT strategies](#role-in-the-jwt-strategies)
16. [Role in the types](#role-in-the-types)
17. [Role in the auth service](#role-in-the-auth-service)
18. [Using RBAC in controllers](#using-rbac-in-controllers)
19. [Global vs route-level guards](#global-vs-route-level-guards)
20. [Permission-based access control](#permission-based-access-control)
21. [Best practices](#best-practices)
22. [Common mistakes](#common-mistakes)
23. [Summary](#summary)

Related: [Custom Exceptions](custom-exception.md) (replace the built-in exceptions used here) · [Services](service.md) · [TypeORM + Supabase](typeorm-config.md) (entities and database)

---

## How it works

```text
Register  ->  password hashed with bcrypt  ->  saved
Login     ->  password compared  ->  JWT signed with a per-user-type secret  ->  returned
Request   ->  Authorization: Bearer <token>  ->  Passport strategy verifies  ->  validate() returns user  ->  req.user
```

| Piece | Responsibility |
|-------|----------------|
| **Strategy** (`PassportStrategy`) | Extracts and verifies the token, loads the user, returns what becomes `req.user`. |
| **Guard** (`AuthGuard('<name>')`) | Triggers the named strategy on a route. |
| **`JwtService`** | Signs tokens at login. |
| **DTOs** | Validate input and shape output. |

This project has **two user types**, each with its own secret, strategy and guard: `customer` and `driver`.

---

## Packages

```bash
npm install --save @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install --save-dev @types/passport-jwt @types/bcrypt
```

| Package | Purpose |
|---------|---------|
| `@nestjs/jwt` | Sign and verify JWTs (`JwtService`). |
| `@nestjs/passport` | Passport integration (`PassportStrategy`, `AuthGuard`). |
| `passport`, `passport-jwt` | Authentication middleware and the JWT strategy. |
| `bcrypt` | Password hashing. |

---

## Project structure

```text
src/
├── auth/
│   ├── constants.ts
│   ├── dto/
│   │   ├── login.dto.ts
│   │   ├── register.dto.ts
│   │   └── auth-response.dto.ts
│   ├── guards/
│   │   ├── jwt-customer.guard.ts
│   │   └── jwt-driver.guard.ts
│   ├── strategies/
│   │   ├── jwt-customer.strategy.ts
│   │   └── jwt-driver.strategy.ts
│   ├── decorators/
│   │   └── current-user.decorator.ts
│   ├── types/
│   │   └── auth.types.ts
│   ├── auth.module.ts
│   ├── auth.service.ts
│   └── auth.controller.ts
├── customers/
│   ├── customers.module.ts
│   ├── customers.service.ts
│   └── customers.controller.ts
└── delivery-drivers/
    ├── delivery-drivers.module.ts
    ├── delivery-drivers.service.ts
    └── delivery-drivers.controller.ts
```

---

## Core authentication files

### Constants

`src/auth/constants.ts`

```ts
export const jwtConstants = {
  customerSecret: process.env.JWT_CUSTOMER_SECRET || 'customer-secret-change-in-production',
  driverSecret: process.env.JWT_DRIVER_SECRET || 'driver-secret-change-in-production',
  expiresIn: '7d', // token valid for 7 days
};
```

| Key | Description |
|-----|-------------|
| `customerSecret` / `driverSecret` | Separate signing secrets, so a customer token can never validate as a driver token. Set real values through env variables. |
| `expiresIn` | Token lifetime. |

> The fallback strings are for local development only. Never ship them to production.

### DTOs

`src/auth/dto/login.dto.ts`

```ts
import { IsEmail, IsNotEmpty, IsString } from 'class-validator';

export class LoginDto {
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @IsNotEmpty()
  password: string;
}
```

`src/auth/dto/register.dto.ts`

```ts
import { IsEmail, IsNotEmpty, IsString, MinLength, IsEnum, IsOptional } from 'class-validator';
import { VEHICLE_TYPE } from '../../delivery-drivers/entities/delivery-driver.entity';

export class RegisterCustomerDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @IsNotEmpty()
  address: string;

  @IsString()
  @IsOptional()
  profile_image_url?: string;
}

export class RegisterDriverDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  @IsNotEmpty()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @IsNotEmpty()
  phone: string;

  @IsEnum(VEHICLE_TYPE)
  vehicle_type: VEHICLE_TYPE;

  @IsString()
  @IsOptional()
  profile_image_url?: string;
}
```

### Current user decorator

`src/auth/decorators/current-user.decorator.ts`

Extracts the authenticated user (set by the strategy) from the request.

```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Usage: `getProfile(@CurrentUser() user: AuthenticatedUser)`

---

## JWT strategies and guards

A strategy validates the token and the user. Each user type has its own named strategy.

### Customer strategy

`src/auth/strategies/jwt-customer.strategy.ts`

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Customer } from '../../customers/entities/customer.entity';

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
    const customer = await this.customerRepository.findOne({
      where: { id: payload.sub },
    });

    if (!customer) {
      throw new UnauthorizedException();
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

### Driver strategy

`src/auth/strategies/jwt-driver.strategy.ts`

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { DeliveryDriver } from '../../delivery-drivers/entities/delivery-driver.entity';

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
    const driver = await this.driverRepository.findOne({
      where: { id: payload.sub },
    });

    if (!driver) {
      throw new UnauthorizedException();
    }

    return {
      id: driver.id,
      email: driver.email,
      name: driver.name,
      userType: 'driver',
    };
  }
}
```

### Strategy options

| Option | Description |
|--------|-------------|
| `PassportStrategy(Strategy, 'jwt-customer')` | Registers the strategy under this name; guards refer to it. |
| `jwtFromRequest` | Where to read the token. `fromAuthHeaderAsBearerToken()` reads `Authorization: Bearer <token>`. |
| `ignoreExpiration: false` | Reject expired tokens. |
| `secretOrKey` | Secret used to verify the signature (must match the one used to sign). |
| `validate(payload)` | Called with the decoded payload after verification. The returned object becomes `req.user`. Throwing rejects the request. |

### Guards

`src/auth/guards/jwt-customer.guard.ts`

```ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtCustomerGuard extends AuthGuard('jwt-customer') {}
```

`src/auth/guards/jwt-driver.guard.ts`

```ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtDriverGuard extends AuthGuard('jwt-driver') {}
```

### Improvements

1. Replace `payload: any` with a proper payload type.
2. Define the shape of the returned user explicitly.

---

## Types and response DTOs

### Auth types

`src/auth/types/auth.types.ts`

```ts
export interface JwtPayload {
  sub: string;
  email: string;
  name: string;
  userType: 'customer' | 'driver';
}

export interface AuthenticatedUser {
  id: string;
  email: string;
  name: string;
  userType: 'customer' | 'driver';
}
```

### Response DTOs

`src/auth/dto/auth-response.dto.ts`

`UserResponseDto` and `AuthResponseDto` could also live in separate files.

```ts
import { Exclude, Expose } from 'class-transformer';

export class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  email: string;

  @Expose()
  name: string;

  constructor(partial: Partial<UserResponseDto>) {
    Object.assign(this, partial);
  }
}

export class AuthResponseDto {
  @Expose()
  access_token: string;

  @Expose()
  user: UserResponseDto;

  constructor(partial: Partial<AuthResponseDto>) {
    Object.assign(this, partial);
  }
}

export class CustomerResponseDto extends UserResponseDto {
  @Expose()
  address: string;

  @Expose()
  profile_image_url?: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<CustomerResponseDto>) {
    super(partial);
    Object.assign(this, partial);
  }
}

export class DriverResponseDto extends UserResponseDto {
  @Expose()
  phone: string;

  @Expose()
  vehicle_type: string;

  @Expose()
  profile_image_url?: string;

  @Expose()
  is_available: boolean;

  @Exclude()
  password: string;

  constructor(partial: Partial<DriverResponseDto>) {
    super(partial);
    Object.assign(this, partial);
  }
}
```

| Decorator | Effect (with `ClassSerializerInterceptor`) |
|-----------|--------------------------------------------|
| `@Expose()` | Property is included in the response. |
| `@Exclude()` | Property is removed from the response (e.g. `password`). |

### Generic API response wrappers

`src/common/dto/api-response.dto.ts`

These can be used app-wide.

```ts
import { Expose } from 'class-transformer';

export class ApiSuccessResponse<T> {
  @Expose()
  success: boolean;

  @Expose()
  data: T;

  @Expose()
  message?: string;

  constructor(data: T, message?: string) {
    this.success = true;
    this.data = data;
    this.message = message;
  }
}

export class ApiErrorResponse {
  @Expose()
  success: boolean;

  @Expose()
  error: {
    message: string;
    statusCode: number;
    timestamp: string;
  };

  constructor(message: string, statusCode: number) {
    this.success = false;
    this.error = {
      message,
      statusCode,
      timestamp: new Date().toISOString(),
    };
  }
}
```

---

## Auth service

`src/auth/auth.service.ts`

Handles registration and login for both user types.

```ts
import { Injectable, UnauthorizedException, ConflictException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as bcrypt from 'bcrypt';
import { Customer } from '../customers/entities/customer.entity';
import { DeliveryDriver } from '../delivery-drivers/entities/delivery-driver.entity';
import { LoginDto } from './dto/login.dto';
import { RegisterCustomerDto, RegisterDriverDto } from './dto/register.dto';
import { AuthResponseDto, CustomerResponseDto, DriverResponseDto } from './dto/auth-response.dto';
import { jwtConstants } from './constants';
import { JwtPayload } from './types/auth.types';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
    private jwtService: JwtService,
  ) {}

  // Customer registration
  async registerCustomer(registerDto: RegisterCustomerDto): Promise<CustomerResponseDto> {
    const existingCustomer = await this.customerRepository.findOne({
      where: { email: registerDto.email },
    });

    if (existingCustomer) {
      throw new ConflictException('Email already exists');
    }

    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(registerDto.password, saltRounds);

    const customer = this.customerRepository.create({
      ...registerDto,
      password: hashedPassword,
    });

    const savedCustomer = await this.customerRepository.save(customer);

    return new CustomerResponseDto(savedCustomer);
  }

  // Customer login
  async loginCustomer(loginDto: LoginDto): Promise<AuthResponseDto> {
    const customer = await this.customerRepository.findOne({
      where: { email: loginDto.email },
    });

    if (!customer) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await bcrypt.compare(loginDto.password, customer.password);

    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const payload: JwtPayload = {
      sub: customer.id,
      email: customer.email,
      name: customer.name,
      userType: 'customer',
    };

    const accessToken = await this.jwtService.signAsync(payload, {
      secret: jwtConstants.customerSecret,
      expiresIn: jwtConstants.expiresIn,
    });

    return new AuthResponseDto({
      access_token: accessToken,
      user: {
        id: customer.id,
        email: customer.email,
        name: customer.name,
      },
    });
  }

  // Driver registration
  async registerDriver(registerDto: RegisterDriverDto): Promise<DriverResponseDto> {
    const existingDriver = await this.driverRepository.findOne({
      where: { email: registerDto.email },
    });

    if (existingDriver) {
      throw new ConflictException('Email already exists');
    }

    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(registerDto.password, saltRounds);

    const driver = this.driverRepository.create({
      ...registerDto,
      password: hashedPassword,
    });

    const savedDriver = await this.driverRepository.save(driver);

    return new DriverResponseDto(savedDriver);
  }

  // Driver login
  async loginDriver(loginDto: LoginDto): Promise<AuthResponseDto> {
    const driver = await this.driverRepository.findOne({
      where: { email: loginDto.email },
    });

    if (!driver) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await bcrypt.compare(loginDto.password, driver.password);

    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const payload: JwtPayload = {
      sub: driver.id,
      email: driver.email,
      name: driver.name,
      userType: 'driver',
    };

    const accessToken = await this.jwtService.signAsync(payload, {
      secret: jwtConstants.driverSecret,
      expiresIn: jwtConstants.expiresIn,
    });

    return new AuthResponseDto({
      access_token: accessToken,
      user: {
        id: driver.id,
        email: driver.email,
        name: driver.name,
      },
    });
  }
}
```

| Call | Description |
|------|-------------|
| `bcrypt.hash(password, saltRounds)` | Hashes the password; `10` is a common cost factor. |
| `bcrypt.compare(plain, hash)` | Checks a plain password against a stored hash. |
| `jwtService.signAsync(payload, { secret, expiresIn })` | Signs the token. The secret must match the strategy's `secretOrKey`. |
| `payload.sub` | Subject: the user's ID, read back in `validate()`. |

> Both "user not found" and "wrong password" return the same `Invalid credentials` message, so the response never reveals which emails exist. To use custom exceptions instead of Nest's built-ins, see `custom-exception.md`.

---

## Auth controller

`src/auth/auth.controller.ts`

```ts
import {
  Controller, Post, Body, HttpCode, HttpStatus,
  UseInterceptors, ClassSerializerInterceptor,
} from '@nestjs/common';
import { AuthService } from './auth.service';
import { LoginDto } from './dto/login.dto';
import { RegisterCustomerDto, RegisterDriverDto } from './dto/register.dto';
import { AuthResponseDto, CustomerResponseDto, DriverResponseDto } from './dto/auth-response.dto';

@Controller('auth')
@UseInterceptors(ClassSerializerInterceptor)
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('customer/register')
  async registerCustomer(@Body() registerDto: RegisterCustomerDto): Promise<CustomerResponseDto> {
    return this.authService.registerCustomer(registerDto);
  }

  @HttpCode(HttpStatus.OK)
  @Post('customer/login')
  async loginCustomer(@Body() loginDto: LoginDto): Promise<AuthResponseDto> {
    return this.authService.loginCustomer(loginDto);
  }

  @Post('driver/register')
  async registerDriver(@Body() registerDto: RegisterDriverDto): Promise<DriverResponseDto> {
    return this.authService.registerDriver(registerDto);
  }

  @HttpCode(HttpStatus.OK)
  @Post('driver/login')
  async loginDriver(@Body() loginDto: LoginDto): Promise<AuthResponseDto> {
    return this.authService.loginDriver(loginDto);
  }
}
```

| Endpoint | Status | Purpose |
|----------|--------|---------|
| `POST /auth/customer/register` | 201 | Register a customer. |
| `POST /auth/customer/login` | 200 | Log in a customer, returns token. |
| `POST /auth/driver/register` | 201 | Register a driver. |
| `POST /auth/driver/login` | 200 | Log in a driver, returns token. |

Login uses `@HttpCode(HttpStatus.OK)` because it does not create a resource (POST would otherwise return 201).

### Why no `@UseGuards` on the auth controller

| Tool | Purpose |
|------|---------|
| `@UseInterceptors(ClassSerializerInterceptor)` | Serializes responses, e.g. drops fields marked `@Exclude()` such as passwords. |
| `@UseGuards(JwtCustomerGuard)` | Protects routes that require an authenticated user. |

Register and login are **public endpoints**: you cannot already be logged in to create an account or to log in. Guards belong on **protected** endpoints:

```ts
import { Controller, Get, Patch, Body, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';

@Controller('customers')
@UseGuards(JwtCustomerGuard) // every route in this controller requires authentication
export class CustomersController {
  @Get('profile')
  async getProfile(@CurrentUser() user) {
    return user; // only authenticated customers get here
  }

  @Patch('profile')
  async updateProfile(@CurrentUser() user, @Body() updateDto) {
    return this.customersService.update(user.id, updateDto);
  }
}
```

| Controller | `@UseInterceptors` | `@UseGuards` |
|------------|--------------------|--------------|
| Auth (register/login) | Yes | No (public) |
| Protected controllers | Optional | Yes |

---

## Modules

### Customers module

`src/customers/customers.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { CustomersService } from './customers.service';
import { CustomersController } from './customers.controller';
import { Customer } from './entities/customer.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Customer])],
  controllers: [CustomersController],
  providers: [CustomersService],
  exports: [CustomersService, TypeOrmModule],
})
export class CustomersModule {}
```

### Delivery drivers module

`src/delivery-drivers/delivery-drivers.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { DeliveryDriversService } from './delivery-drivers.service';
import { DeliveryDriversController } from './delivery-drivers.controller';
import { DeliveryDriver } from './entities/delivery-driver.entity';

@Module({
  imports: [TypeOrmModule.forFeature([DeliveryDriver])],
  controllers: [DeliveryDriversController],
  providers: [DeliveryDriversService],
  exports: [DeliveryDriversService, TypeOrmModule],
})
export class DeliveryDriversModule {}
```

> Exporting `TypeOrmModule` lets other modules (such as `AuthModule`) inject the `Customer` / `DeliveryDriver` repositories.

### Auth module

`src/auth/auth.module.ts`

Registers the strategies, the service and the controller. This file is not part of the original notes: it is a minimal wiring that the structure above expects.

```ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtCustomerStrategy } from './strategies/jwt-customer.strategy';
import { JwtDriverStrategy } from './strategies/jwt-driver.strategy';
import { Customer } from '../customers/entities/customer.entity';
import { DeliveryDriver } from '../delivery-drivers/entities/delivery-driver.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([Customer, DeliveryDriver]),
    PassportModule,
    JwtModule.register({}), // secret and expiry are passed per call in signAsync()
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtCustomerStrategy, JwtDriverStrategy],
})
export class AuthModule {}
```

> `JwtModule.register({})` is empty on purpose: each `signAsync()` call passes its own `secret`, which is how two user types use two secrets.

---

## RBAC overview

RBAC restricts access based on the role assigned to a user. It controls which users can reach specific endpoints or perform certain actions.

| Component | Purpose |
|-----------|---------|
| **Roles** | The user types (`admin`, `customer`, `driver`, `manager`). |
| **`@Roles()` decorator** | Marks an endpoint with the roles allowed to use it. |
| **`RolesGuard`** | Verifies `req.user` has one of the required roles. |
| **Role assignment** | Role stored in the database and exposed to the guard through the JWT strategy. |

**Request flow**

```text
Request + JWT
  -> JwtCustomerGuard   (verifies token, fills req.user, including role)
  -> RolesGuard         (reads @Roles() metadata, compares with req.user.role)
  -> Handler            (or 403 Forbidden)
```

---

## Roles enum

One central enum for all roles.

`src/auth/enums/role.enum.ts`

```ts
export enum Role {
  ADMIN = 'admin',
  CUSTOMER = 'customer',
  DRIVER = 'driver',
  MANAGER = 'manager',
}
```

| Benefit | Description |
|---------|-------------|
| Type safety | Prevents typos when assigning or checking roles. |
| Single source of truth | All roles in one place. |
| Autocomplete | IDEs suggest the available roles. |
| Easy refactoring | Rename a role in one place. |

---

## Role column on the entities

Add a `role` column to the user entities.

`src/customers/entities/customer.entity.ts`

```ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';
import { Role } from '../../auth/enums/role.enum';

@Entity('customers')
export class Customer {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column()
  address: string;

  @Column({ nullable: true })
  profile_image_url: string;

  @Column({
    type: 'enum',
    enum: Role,
    default: Role.CUSTOMER,
  })
  role: Role;

  @CreateDateColumn()
  created_at: Date;
}
```

`src/delivery-drivers/entities/delivery-driver.entity.ts`

```ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';
import { Role } from '../../auth/enums/role.enum';

export enum VEHICLE_TYPE {
  BIKE = 'bike',
  SCOOTER = 'scooter',
  CAR = 'car',
}

@Entity('delivery_drivers')
export class DeliveryDriver {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column()
  phone: string;

  @Column({ type: 'enum', enum: VEHICLE_TYPE })
  vehicle_type: VEHICLE_TYPE;

  @Column({ nullable: true })
  profile_image_url: string;

  @Column({ default: true })
  is_available: boolean;

  @Column({
    type: 'enum',
    enum: Role,
    default: Role.DRIVER,
  })
  role: Role;

  @CreateDateColumn()
  created_at: Date;
}
```

| Option | Description |
|--------|-------------|
| `type: 'enum'` | Stores the role as a database enum. |
| `enum: Role` | References the TypeScript enum. |
| `default: Role.CUSTOMER` | Assigned automatically on creation. |
| Result | The database only accepts valid roles. |

---

## Roles decorator

`src/auth/decorators/roles.decorator.ts`

```ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

| Part | Description |
|------|-------------|
| `SetMetadata()` | Attaches metadata to a route handler or class. |
| `ROLES_KEY` | Key used to store and retrieve the metadata. |
| `...roles: Role[]` | Accepts several roles. |

```ts
@Roles(Role.ADMIN)               // only admins
@Get('admin/dashboard')
getDashboard() {}

@Roles(Role.ADMIN, Role.MANAGER) // admins OR managers
@Get('reports')
getReports() {}
```

---

## Roles guard

`src/auth/guards/roles.guard.ts`

```ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from '../enums/role.enum';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Get required roles from decorator metadata
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(), // method-level decorator
      context.getClass(),   // class-level decorator
    ]);

    // No roles required: allow access
    if (!requiredRoles) {
      return true;
    }

    // Get user from request (set by the JWT strategy)
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      throw new ForbiddenException('User not authenticated');
    }

    // User needs at least one of the required roles
    const hasRole = requiredRoles.some((role) => user.role === role);

    if (!hasRole) {
      throw new ForbiddenException(
        `Access denied. Required roles: ${requiredRoles.join(', ')}. Your role: ${user.role}`,
      );
    }

    return true;
  }
}
```

### How the guard decides

| Step | Behavior |
|------|----------|
| 1. Read required roles | `reflector.getAllAndOverride()` reads `@Roles()` from the method and the class; the method-level value wins. |
| 2. No roles required | Returns `true`, so unannotated routes stay open. |
| 3. Read the user | `request.user` comes from the strategy's `validate()` and must contain `role`. |
| 4. Compare | `some()` gives **OR** logic: any matching role passes. |
| 5. Result | Match returns `true`; otherwise `ForbiddenException` with a descriptive message. |

Example 403 response:

```json
{
  "statusCode": 403,
  "message": "Access denied. Required roles: admin, manager. Your role: customer",
  "error": "Forbidden"
}
```

---

## Role in the JWT strategies

Return the role from `validate()` so it lands on `request.user`.

`src/auth/strategies/jwt-customer.strategy.ts`

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Customer } from '../../customers/entities/customer.entity';
import { JwtPayload } from '../types/auth.types';

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

  async validate(payload: JwtPayload) {
    const customer = await this.customerRepository.findOne({
      where: { id: payload.sub },
    });

    if (!customer) {
      throw new UnauthorizedException('User not found');
    }

    // Becomes request.user; RolesGuard reads role from here
    return {
      id: customer.id,
      email: customer.email,
      name: customer.name,
      role: customer.role, // included for RBAC
      userType: 'customer',
    };
  }
}
```

`src/auth/strategies/jwt-driver.strategy.ts`

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from '../constants';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { DeliveryDriver } from '../../delivery-drivers/entities/delivery-driver.entity';
import { JwtPayload } from '../types/auth.types';

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

  async validate(payload: JwtPayload) {
    const driver = await this.driverRepository.findOne({
      where: { id: payload.sub },
    });

    if (!driver) {
      throw new UnauthorizedException('User not found');
    }

    return {
      id: driver.id,
      email: driver.email,
      name: driver.name,
      role: driver.role, // included for RBAC
      userType: 'driver',
    };
  }
}
```

**What changed:** `role` is added to the returned object, so `request.user.role` exists for the guard.

**Flow**

1. Request arrives with a JWT.
2. The strategy verifies the token and decodes the payload.
3. The strategy loads the user (and role) from the database.
4. The returned object is attached to `request.user`.
5. `RolesGuard` compares `request.user.role` with `@Roles()`.

---

## Role in the types

`src/auth/types/auth.types.ts`

```ts
import { Role } from '../enums/role.enum';

export interface JwtPayload {
  sub: string;                       // user ID
  email: string;
  name: string;
  role: Role;                        // user's role
  userType: 'customer' | 'driver';
}

export interface AuthenticatedUser {
  id: string;
  email: string;
  name: string;
  role: Role;                        // user's role
  userType: 'customer' | 'driver';
}
```

### Role in the JWT payload

| Benefit | Description |
|---------|-------------|
| Fast authorization | Role is available without extra work. |
| Stateless | Everything needed is in the token. |
| Lower DB load | No lookup solely to read the role. |

> **Security:** a JWT payload is base64-encoded, **not encrypted**. Never store secrets in it. Roles are safe to include, but they can go stale if a role changes before the token expires, so verify against the database for critical operations.

---

## Role in the auth service

Add `role` to the payload at login. Only the changed parts of the login methods are shown.

`src/auth/auth.service.ts`

```ts
// Customer login
async loginCustomer(loginDto: LoginDto): Promise<AuthResponseDto> {
  const customer = await this.customerRepository.findOne({
    where: { email: loginDto.email },
  });

  if (!customer) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const isPasswordValid = await bcrypt.compare(loginDto.password, customer.password);

  if (!isPasswordValid) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const payload: JwtPayload = {
    sub: customer.id,
    email: customer.email,
    name: customer.name,
    role: customer.role,           // role added to the token
    userType: 'customer',
  };

  const accessToken = await this.jwtService.signAsync(payload, {
    secret: jwtConstants.customerSecret,
    expiresIn: jwtConstants.expiresIn,
  });

  return new AuthResponseDto({
    access_token: accessToken,
    user: { id: customer.id, email: customer.email, name: customer.name },
  });
}

// Driver login
async loginDriver(loginDto: LoginDto): Promise<AuthResponseDto> {
  const driver = await this.driverRepository.findOne({
    where: { email: loginDto.email },
  });

  if (!driver) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const isPasswordValid = await bcrypt.compare(loginDto.password, driver.password);

  if (!isPasswordValid) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const payload: JwtPayload = {
    sub: driver.id,
    email: driver.email,
    name: driver.name,
    role: driver.role,             // role added to the token
    userType: 'driver',
  };

  const accessToken = await this.jwtService.signAsync(payload, {
    secret: jwtConstants.driverSecret,
    expiresIn: jwtConstants.expiresIn,
  });

  return new AuthResponseDto({
    access_token: accessToken,
    user: { id: driver.id, email: driver.email, name: driver.name },
  });
}
```

**What happens at login**

1. User sends email and password.
2. Service validates the credentials.
3. Service builds the JWT payload **with the role**.
4. The token is signed and returned.
5. The client sends it on later requests.
6. The strategy decodes it and loads the user and role.
7. `RolesGuard` compares the role with the required roles.

---

## Using RBAC in controllers

### Admin controller (class-level)

`src/admin/admin.controller.ts`

```ts
import { Controller, Get, Delete, Param, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { AuthenticatedUser } from '../auth/types/auth.types';

@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard) // both guards
@Roles(Role.ADMIN)                       // every route requires ADMIN
export class AdminController {
  @Get('users')
  getAllUsers(@CurrentUser() user: AuthenticatedUser) {
    return { message: 'List of all users', admin: user.name };
  }

  @Get('statistics')
  getStatistics() {
    return { message: 'System statistics' };
  }

  @Delete('users/:id')
  deleteUser(@Param('id') id: string) {
    return { message: `User ${id} deleted` };
  }
}
```

**Guard order matters**

1. `JwtCustomerGuard` validates the JWT and fills `request.user`.
2. `RolesGuard` checks `request.user.role` against `@Roles()`.

### Mixed roles controller (method-level)

`src/orders/orders.controller.ts`

```ts
import { Controller, Get, Post, Patch, Delete, Param, Body, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { AuthenticatedUser } from '../auth/types/auth.types';

@Controller('orders')
@UseGuards(JwtCustomerGuard, RolesGuard) // applies to the whole controller
export class OrdersController {
  @Post()
  @Roles(Role.CUSTOMER)                  // only customers create orders
  createOrder(@CurrentUser() user: AuthenticatedUser, @Body() createOrderDto) {
    return { message: 'Order created', customerId: user.id };
  }

  @Get('my-orders')
  @Roles(Role.CUSTOMER)
  getMyOrders(@CurrentUser() user: AuthenticatedUser) {
    return { message: 'Your orders', customerId: user.id };
  }

  @Get('all')
  @Roles(Role.ADMIN, Role.MANAGER)       // admins OR managers
  getAllOrders() {
    return { message: 'All orders in the system' };
  }

  @Patch(':id/status')
  @Roles(Role.ADMIN, Role.MANAGER, Role.DRIVER) // multiple roles allowed
  updateOrderStatus(@Param('id') id: string, @Body() statusDto) {
    return { message: `Order ${id} status updated` };
  }

  @Delete(':id')
  @Roles(Role.ADMIN)
  deleteOrder(@Param('id') id: string) {
    return { message: `Order ${id} deleted` };
  }
}
```

**Access matrix**

| Endpoint | Admin | Manager | Driver | Customer |
|----------|:-----:|:-------:|:------:|:--------:|
| `POST /orders` | No | No | No | Yes |
| `GET /orders/my-orders` | No | No | No | Yes |
| `GET /orders/all` | Yes | Yes | No | No |
| `PATCH /orders/:id/status` | Yes | Yes | Yes | No |
| `DELETE /orders/:id` | Yes | No | No | No |

> **Heads up:** this controller authenticates with `JwtCustomerGuard`, which validates against the **customers** table. A user from the **drivers** table carries a driver token (signed with `driverSecret`) and would fail that guard, so the `Role.DRIVER` entry on `PATCH :id/status` never gets reached. To let drivers through, use a guard that accepts both strategies (for example `AuthGuard(['jwt-customer', 'jwt-driver'])`) or put driver routes behind `JwtDriverGuard`.

### Public and protected routes together

`src/products/products.controller.ts`

```ts
import { Controller, Get, Post, Patch, Delete, Param, Body, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';

@Controller('products')
export class ProductsController {
  // Public: no guards
  @Get()
  getAllProducts() {
    return { message: 'Public product list' };
  }

  // Public: no guards
  @Get(':id')
  getProduct(@Param('id') id: string) {
    return { message: `Product ${id} details` };
  }

  // Protected: admin or manager
  @Post()
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN, Role.MANAGER)
  createProduct(@Body() createProductDto) {
    return { message: 'Product created' };
  }

  // Protected: admin or manager
  @Patch(':id')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN, Role.MANAGER)
  updateProduct(@Param('id') id: string, @Body() updateProductDto) {
    return { message: `Product ${id} updated` };
  }

  // Protected: admin only
  @Delete(':id')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN)
  deleteProduct(@Param('id') id: string) {
    return { message: `Product ${id} deleted` };
  }
}
```

| Route type | How |
|------------|-----|
| Public | No `@UseGuards()`: anyone can access. |
| Protected | Authentication guard plus `RolesGuard` and `@Roles()`. |
| Mixed | Apply the guards at **method level** in the same controller. |

---

## Global vs route-level guards

### Option 1: Route-level (recommended)

```ts
@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard) // controller-level
@Roles(Role.ADMIN)
export class AdminController {
  // all routes protected
}

@Controller('products')
export class ProductsController {
  @Get() // public
  getAll() {}

  @Post()
  @UseGuards(JwtCustomerGuard, RolesGuard) // route-level
  @Roles(Role.ADMIN)
  create() {} // protected
}
```

Advantages: fine-grained control, easy to mix public and protected routes, clear which routes are protected, easy to maintain.

### Option 2: Global guard

`main.ts`

```ts
import { NestFactory, Reflector } from '@nestjs/core';
import { AppModule } from './app.module';
import { RolesGuard } from './auth/guards/roles.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global RolesGuard
  const reflector = app.get(Reflector);
  app.useGlobalGuards(new RolesGuard(reflector));

  await app.listen(3000);
}
bootstrap();
```

| Advantages | Disadvantages |
|------------|---------------|
| Applied to all routes automatically | Public routes must be marked explicitly |
| DRY, centralized configuration | Less flexible |
| | Harder to debug |

### Option 3: Hybrid

Global authentication, route-level roles.

```ts
// main.ts: global authentication guard
app.useGlobalGuards(new JwtAuthGuard());
```

```ts
@Controller('products')
export class ProductsController {
  @Get()
  @Public()            // custom decorator to skip auth
  getAll() {}

  @Post()
  @Roles(Role.ADMIN)   // add role requirement
  create() {}
}
```

`src/auth/decorators/public.decorator.ts`

```ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

> The global guard must read `IS_PUBLIC_KEY` through `Reflector` and skip authentication when it is set.

---

## Permission-based access control

For finer granularity than roles, map each role to a set of permissions and check permissions instead.

`src/auth/enums/permission.enum.ts`

```ts
export enum Permission {
  // User permissions
  CREATE_USER = 'create:user',
  READ_USER = 'read:user',
  UPDATE_USER = 'update:user',
  DELETE_USER = 'delete:user',

  // Order permissions
  CREATE_ORDER = 'create:order',
  READ_ORDER = 'read:order',
  UPDATE_ORDER = 'update:order',
  DELETE_ORDER = 'delete:order',

  // Product permissions
  CREATE_PRODUCT = 'create:product',
  READ_PRODUCT = 'read:product',
  UPDATE_PRODUCT = 'update:product',
  DELETE_PRODUCT = 'delete:product',
}
```

`src/auth/constants/role-permissions.ts`

```ts
import { Role } from '../enums/role.enum';
import { Permission } from '../enums/permission.enum';

export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  [Role.ADMIN]: [
    ...Object.values(Permission), // admins have all permissions
  ],

  [Role.MANAGER]: [
    Permission.READ_USER,
    Permission.UPDATE_USER,
    Permission.CREATE_ORDER,
    Permission.READ_ORDER,
    Permission.UPDATE_ORDER,
    Permission.CREATE_PRODUCT,
    Permission.READ_PRODUCT,
    Permission.UPDATE_PRODUCT,
  ],

  [Role.DRIVER]: [
    Permission.READ_ORDER,
    Permission.UPDATE_ORDER,
  ],

  [Role.CUSTOMER]: [
    Permission.READ_USER,
    Permission.UPDATE_USER,
    Permission.CREATE_ORDER,
    Permission.READ_ORDER,
    Permission.READ_PRODUCT,
  ],
};
```

`src/auth/decorators/permissions.decorator.ts`

```ts
import { SetMetadata } from '@nestjs/common';
import { Permission } from '../enums/permission.enum';

export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

`src/auth/guards/permissions.guard.ts`

```ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Permission } from '../enums/permission.enum';
import { PERMISSIONS_KEY } from '../decorators/permissions.decorator';
import { ROLE_PERMISSIONS } from '../constants/role-permissions';

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<Permission[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user || !user.role) {
      throw new ForbiddenException('User not authenticated');
    }

    // Permissions for the user's role
    const userPermissions = ROLE_PERMISSIONS[user.role] || [];

    // User must have ALL required permissions
    const hasAllPermissions = requiredPermissions.every((permission) =>
      userPermissions.includes(permission),
    );

    if (!hasAllPermissions) {
      throw new ForbiddenException(
        `Insufficient permissions. Required: ${requiredPermissions.join(', ')}`,
      );
    }

    return true;
  }
}
```

Usage:

```ts
@Controller('users')
@UseGuards(JwtCustomerGuard, PermissionsGuard)
export class UsersController {
  @Post()
  @RequirePermissions(Permission.CREATE_USER)
  create(@Body() createUserDto) {
    return { message: 'User created' };
  }

  @Delete(':id')
  @RequirePermissions(Permission.DELETE_USER)
  delete(@Param('id') id: string) {
    return { message: 'User deleted' };
  }

  @Patch(':id')
  @RequirePermissions(Permission.UPDATE_USER)
  update(@Param('id') id: string, @Body() updateUserDto) {
    return { message: 'User updated' };
  }
}
```

| | `RolesGuard` | `PermissionsGuard` |
|--|--------------|--------------------|
| Logic | **Any** required role matches (OR) | **All** required permissions held (AND) |
| Source | `user.role` | `ROLE_PERMISSIONS[user.role]` |

---

## Best practices

### 1. Apply guards in the correct order

```ts
// Correct
@UseGuards(JwtCustomerGuard, RolesGuard)

// Wrong: RolesGuard runs before authentication
@UseGuards(RolesGuard, JwtCustomerGuard)
```

`JwtCustomerGuard` populates `request.user`, which `RolesGuard` needs.

### 2. Use enums for type safety

```ts
@Roles(Role.ADMIN, Role.MANAGER)   // type-safe, autocomplete works
@Roles('admin', 'manager')         // typo-prone, no autocomplete
```

### 3. Prefer class-level guards when possible

```ts
// DRY: all routes protected
@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)
@Roles(Role.ADMIN)
export class AdminController {
  @Get('users') getUsers() {}
  @Get('stats') getStats() {}
}

// Repetitive: same guards on every route
@Controller('admin')
export class AdminController {
  @Get('users')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN)
  getUsers() {}

  @Get('stats')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN)
  getStats() {}
}
```

### 4. Re-verify roles in the database for critical operations

JWTs can go stale when a user's role changes.

```ts
@Delete('users/:id')
@Roles(Role.ADMIN)
async deleteUser(@Param('id') id: string, @CurrentUser() user: AuthenticatedUser) {
  const currentUser = await this.userRepository.findOne({ where: { id: user.id } });

  if (currentUser.role !== Role.ADMIN) {
    throw new ForbiddenException('Role verification failed');
  }

  return this.userService.delete(id);
}
```

### 5. Use descriptive error messages

```ts
// Good
throw new ForbiddenException(
  `Access denied. Required roles: ${requiredRoles.join(', ')}. Your role: ${user.role}`,
);

// Bad
throw new ForbiddenException('Access denied');
```

### 6. Document role requirements

```ts
/**
 * Delete a user from the system
 * @requires Role.ADMIN - only administrators can delete users
 * @param id - user ID to delete
 */
@Delete(':id')
@Roles(Role.ADMIN)
async deleteUser(@Param('id') id: string) {}
```

### 7. Test role-based access

```ts
describe('AdminController', () => {
  it('should allow admin to access dashboard', async () => {
    const adminToken = 'valid-admin-token';
    await request(app.getHttpServer())
      .get('/admin/dashboard')
      .set('Authorization', `Bearer ${adminToken}`)
      .expect(200);
  });

  it('should deny customer access to admin dashboard', async () => {
    const customerToken = 'valid-customer-token';
    await request(app.getHttpServer())
      .get('/admin/dashboard')
      .set('Authorization', `Bearer ${customerToken}`)
      .expect(403);
  });
});
```

---

## Common mistakes

| # | Pitfall | Fix |
|---|---------|-----|
| 1 | `@Roles()` used without `RolesGuard` (does nothing) | Add `@UseGuards(JwtCustomerGuard, RolesGuard)` |
| 2 | Wrong guard order (`RolesGuard` first) | Authentication guard first |
| 3 | Role missing from the strategy's returned user | Return `role` from `validate()` |
| 4 | Hard-coded role strings | Use the `Role` enum |
| 5 | Assuming `request.user` exists | Check `user` and `user.role` before comparing |
| 6 | Using one secret for customers and drivers | A token for one user type could validate as the other | Keep separate secrets (`customerSecret`, `driverSecret`) and strategies |
| 7 | Leaving the fallback secrets in `constants.ts` for production | Anyone who reads the repo can forge tokens | Set `JWT_CUSTOMER_SECRET` and `JWT_DRIVER_SECRET` in the environment |
| 8 | Returning the saved entity from register | The password hash is sent to the client | Return a response DTO (`CustomerResponseDto`) with `@Exclude()` and `ClassSerializerInterceptor` |

```ts
// 1. Missing RolesGuard
@Controller('admin')
@Roles(Role.ADMIN)                              // wrong: metadata is never checked
export class AdminController {}

@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)        // correct
@Roles(Role.ADMIN)
export class AdminController {}

// 3. Role missing in validate()
async validate(payload: JwtPayload) {
  return { id: user.id, email: user.email };                 // wrong
  return { id: user.id, email: user.email, role: user.role }; // correct
}

// 5. Unchecked user
const hasRole = requiredRoles.some((role) => user.role === role);   // wrong: user may be undefined

if (!user || !user.role) {
  throw new ForbiddenException('User not authenticated');           // correct
}
```

---

## Summary

| Step | What |
|------|------|
| 1 | Define roles in an enum. |
| 2 | Store the role on the user entities. |
| 3 | Create the `@Roles()` decorator. |
| 4 | Create `RolesGuard`. |
| 5 | Return `role` from the JWT strategies. |
| 6 | Add `role` to the types. |
| 7 | Include `role` in the JWT payload at login. |
| 8 | Apply `@UseGuards()` and `@Roles()` in controllers. |

| Concept | Role in the system |
|---------|--------------------|
| Roles | Which user types exist (admin, customer, driver, manager). |
| `@Roles()` | Marks endpoints with required roles. |
| `RolesGuard` | Enforces the requirement. |
| JWT strategy | Supplies the user's role to the guard. |
| Order | Authentication guard, then role guard. |

**Security tips**

- Always validate the JWT before checking roles.
- Use enums for type safety.
- Double-check roles from the database for critical operations.
- Provide clear error messages.
- Test role-based access thoroughly.
