# Complete JWT Authentication Setup for NestJS

## Step 1: Install packages

```bash
npm install --save @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install --save-dev @types/passport-jwt @types/bcrypt
```

## Step 2: Project Structure
```
src/
├── auth/
│   ├── constants.ts
│   ├── dto/
│   │   ├── login.dto.ts
│   │   └── register.dto.ts
│   ├── guards/
│   │   ├── jwt-customer.guard.ts
│   │   └── jwt-driver.guard.ts
│   ├── strategies/
│   │   ├── jwt-customer.strategy.ts
│   │   └── jwt-driver.strategy.ts
│   ├── decorators/
│   │   └── current-user.decorator.ts
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

## Step 3: Core Authentication Files

### Constants

**src/auth/constants.ts**

```ts
export const jwtConstants = {
  customerSecret: process.env.JWT_CUSTOMER_SECRET || 'customer-secret-change-in-production',
  driverSecret: process.env.JWT_DRIVER_SECRET || 'driver-secret-change-in-production',
  expiresIn: '7d', // Token valid for 7 days
};

```

### DTOs

**src/auth/dto/login-dto.ts**

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

**src/auth/dto/register.dto**

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

### Decorators

**src/auth/decorators/current-user.decorator.ts**

Extract Current user from the request

``` ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

```

## Step 4: JWT Strategies & Guards

Validates the user.

### Customer Strategy

**src/auth/strategies/jwt-customer.strategy.ts**

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

### Driver Strategy

**src/auth/strategies/jwt-driver.strategy.ts**

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

**src/auth/guards/customer.guard.ts**

```ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtCustomerGuard extends AuthGuard('jwt-customer') {}

```
**src/auth/guards/driver.guard.ts**

```ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtDriverGuard extends AuthGuard('jwt-driver') {}

```

### Key points

1. any type should be changed to a proper payload type.
2. return format can be specified.

### Key Improvements

Add types:

**src/auth/types/auth.types.ts**

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

**src/auth/dto/auth-response.dto.ts**

*NOTE*: The UserResponseDTO and AuthResponseDTO can be in separate files.

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

**src/common/dto/api-response.dto.ts**

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

Now these tyoes can be used app wide.

## Step 5: Auth Service

Handles registration and login

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
import { JwtPayload } from '../common/types/auth.types';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
    private jwtService: JwtService,
  ) {}

  // Customer Registration
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

  // Customer Login
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

  // Driver Registration
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

  // Driver Login
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

- ## Step 5: Controller

```ts
import { 
  Controller, 
  Post, 
  Body, 
  HttpCode, 
  HttpStatus,
  UseInterceptors,
  ClassSerializerInterceptor,
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

### Why No @UseGuards on Auth Controller?

**Important Distinction:**

- **`@UseInterceptors(ClassSerializerInterceptor)`** - Used for serializing response data (e.g., excluding password fields from responses)
- **`@UseGuards(JwtCustomerGuard)`** - Used for protecting routes that require authentication

**Authentication vs Authorization:**

The auth controller endpoints (register/login) are **public endpoints** - they don't require authentication because:
- `/auth/customer/register` - Anyone can register (you can't be logged in to create an account)
- `/auth/customer/login` - Anyone can login (you can't be logged in to login)
- `/auth/driver/register` - Anyone can register as a driver
- `/auth/driver/login` - Anyone can login as a driver

**Where to Use Guards:**

Guards are used on **protected endpoints** that require the user to be authenticated:

```ts
import { Controller, Get, Patch, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';

@Controller('customers')
@UseGuards(JwtCustomerGuard) // All routes in this controller require authentication
export class CustomersController {
  
  @Get('profile')
  async getProfile(@CurrentUser() user) {
    // Only authenticated customers can access this
    return user;
  }

  @Patch('profile')
  async updateProfile(@CurrentUser() user, @Body() updateDto) {
    // Only authenticated customers can update their profile
    return this.customersService.update(user.id, updateDto);
  }
}
```

**ClassSerializerInterceptor Purpose:**

It transforms the response and excludes fields marked with `@Exclude()` decorator (like passwords):

```ts
export class CustomerResponseDto {
  @Expose()
  id: string;

  @Expose()
  email: string;

  @Exclude() // This field won't be sent in the response
  password: string;
}
```

**Summary:**
- Auth controller: `@UseInterceptors` ✓, `@UseGuards` ✗ (public endpoints)
- Protected controllers: `@UseGuards` ✓ (requires authentication)
- Both can be used together when needed

## Step 6: Update the Modules

### Customers Module

**src/customers/customers.module.ts**

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

### Drivers Module

**src/driver/drivers.module.ts**

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

---

# Role-Based Access Control (RBAC)

Role-based Access Control (RBAC) is a security mechanism that restricts system access based on user roles. In NestJS, RBAC allows you to control which users can access specific endpoints or perform certain actions based on their assigned roles.

## Overview

RBAC consists of several key components:

1. **Roles** - Define different user types (e.g., 'admin', 'customer', 'driver', 'manager')
2. **Role Decorator** - Mark endpoints with required roles
3. **Roles Guard** - Verify user has the required role(s)
4. **Role Assignment** - Store and retrieve user roles from database/JWT

## Step 1: Define Roles Enum

Create a centralized enum for all roles in your application.

**src/auth/enums/role.enum.ts**

```ts
export enum Role {
  ADMIN = 'admin',
  CUSTOMER = 'customer',
  DRIVER = 'driver',
  MANAGER = 'manager',
}
```

**Why use an enum?**
- Type safety: Prevents typos when assigning or checking roles
- Centralized management: Single source of truth for all roles
- Autocomplete: IDEs can suggest available roles
- Easy refactoring: Change role names in one place

## Step 2: Update Database Entities

Add role field to your user entities to store user roles.

**src/customers/entities/customer.entity.ts**

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

**src/delivery-drivers/entities/delivery-driver.entity.ts**

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

  @Column({
    type: 'enum',
    enum: VEHICLE_TYPE,
  })
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

**Key Points:**
- `type: 'enum'` - Stores role as enum in database
- `enum: Role` - References the Role enum
- `default: Role.CUSTOMER` - Auto-assigns default role on creation
- Database ensures only valid roles can be stored

## Step 3: Create Roles Decorator

Create a custom decorator to specify which roles can access an endpoint.

**src/auth/decorators/roles.decorator.ts**

```ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

**How it works:**
- `SetMetadata()` - Attaches metadata to route handlers
- `ROLES_KEY` - Unique key to store/retrieve role metadata
- `...roles: Role[]` - Accepts multiple roles (e.g., `@Roles(Role.ADMIN, Role.MANAGER)`)
- Returns a decorator that can be applied to controller methods or classes

**Usage Example:**
```ts
@Roles(Role.ADMIN)  // Only admins can access
@Get('admin/dashboard')
getDashboard() { }

@Roles(Role.ADMIN, Role.MANAGER)  // Admins OR managers can access
@Get('reports')
getReports() { }
```

## Step 4: Create Roles Guard

Create a guard that checks if the user has the required role(s).

**src/auth/guards/roles.guard.ts**

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
      context.getHandler(), // Method-level decorator
      context.getClass(),   // Class-level decorator
    ]);

    // If no roles are required, allow access
    if (!requiredRoles) {
      return true;
    }

    // Get user from request (set by JWT strategy)
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    // Check if user exists
    if (!user) {
      throw new ForbiddenException('User not authenticated');
    }

    // Check if user has at least one of the required roles
    const hasRole = requiredRoles.some((role) => user.role === role);

    if (!hasRole) {
      throw new ForbiddenException(
        `Access denied. Required roles: ${requiredRoles.join(', ')}. Your role: ${user.role}`
      );
    }

    return true;
  }
}
```

**How the guard works:**

1. **Retrieve Required Roles:**
   - `reflector.getAllAndOverride()` gets roles from `@Roles()` decorator
   - Checks both method-level and class-level decorators
   - Method-level decorators override class-level ones

2. **No Roles Required:**
   - If `@Roles()` decorator isn't present, allow access
   - Makes the guard non-breaking for unprotected routes

3. **Get Current User:**
   - `request.user` is populated by JWT strategy's `validate()` method
   - Contains user info including their role

4. **Role Verification:**
   - `some()` checks if user has ANY of the required roles (OR logic)
   - Returns `true` if match found, `false` otherwise

5. **Access Control:**
   - If user has required role: Allow access (return `true`)
   - If user lacks required role: Throw `ForbiddenException` with descriptive message

**Example Error Response:**
```json
{
  "statusCode": 403,
  "message": "Access denied. Required roles: admin, manager. Your role: customer",
  "error": "Forbidden"
}
```

## Step 5: Update JWT Strategies

Modify JWT strategies to include the user's role in the JWT payload and request object.

**src/auth/strategies/jwt-customer.strategy.ts**

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
    // Find user in database
    const customer = await this.customerRepository.findOne({
      where: { id: payload.sub },
    });

    if (!customer) {
      throw new UnauthorizedException('User not found');
    }

    // Return user object that will be attached to request.user
    // This object is used by RolesGuard to check permissions
    return {
      id: customer.id,
      email: customer.email,
      name: customer.name,
      role: customer.role,      // Include role for RBAC
      userType: 'customer',
    };
  }
}
```

**src/auth/strategies/jwt-driver.strategy.ts**

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
    // Find user in database
    const driver = await this.driverRepository.findOne({
      where: { id: payload.sub },
    });

    if (!driver) {
      throw new UnauthorizedException('User not found');
    }

    // Return user object that will be attached to request.user
    // This object is used by RolesGuard to check permissions
    return {
      id: driver.id,
      email: driver.email,
      name: driver.name,
      role: driver.role,        // Include role for RBAC
      userType: 'driver',
    };
  }
}
```

**What Changed:**
- Added `role: customer.role` / `role: driver.role` to returned object
- This makes the role available in `request.user.role`
- RolesGuard uses this to verify permissions

**Flow:**
1. User sends request with JWT token
2. JWT strategy validates token and decodes payload
3. Strategy queries database to get user with their role
4. Strategy returns user object (including role) → attached to `request.user`
5. RolesGuard reads `request.user.role` and checks against `@Roles()` decorator

## Step 6: Update Types

Update authentication types to include role information.

**src/auth/types/auth.types.ts**

```ts
import { Role } from '../enums/role.enum';

export interface JwtPayload {
  sub: string;                     // User ID
  email: string;
  name: string;
  role: Role;                      // User's role
  userType: 'customer' | 'driver';
}

export interface AuthenticatedUser {
  id: string;
  email: string;
  name: string;
  role: Role;                      // User's role
  userType: 'customer' | 'driver';
}
```

**Why include role in JWT payload?**
- **Fast Authorization:** Role is available without database query
- **Stateless:** All needed info is in the token
- **Reduced DB Load:** Don't need to fetch user on every request

**Security Consideration:**
- JWT payload is NOT encrypted (it's base64 encoded)
- Don't store sensitive data in JWT
- Roles are safe to include as they're not secret
- Always verify role against database for critical operations

## Step 7: Update Auth Service

Modify auth service to include role when generating JWT tokens.

**src/auth/auth.service.ts** (Updated login methods)

```ts
// Customer Login
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

  // Include role in JWT payload
  const payload: JwtPayload = {
    sub: customer.id,
    email: customer.email,
    name: customer.name,
    role: customer.role,           // Add role to token
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

// Driver Login
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

  // Include role in JWT payload
  const payload: JwtPayload = {
    sub: driver.id,
    email: driver.email,
    name: driver.name,
    role: driver.role,             // Add role to token
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
```

**What happens during login:**
1. User provides email/password
2. Service validates credentials
3. Service creates JWT payload **with role**
4. JWT token is signed and returned to user
5. User includes token in subsequent requests
6. JWT strategy decodes token and extracts role
7. RolesGuard checks if user's role matches required roles

## Step 8: Using RBAC in Controllers

Apply role-based access control to your controller endpoints.

### Example: Admin Controller

**src/admin/admin.controller.ts**

```ts
import { Controller, Get, Post, Delete, Param, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { AuthenticatedUser } from '../auth/types/auth.types';

@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)  // Apply both guards
@Roles(Role.ADMIN)  // All routes require ADMIN role
export class AdminController {
  
  @Get('users')
  getAllUsers(@CurrentUser() user: AuthenticatedUser) {
    // Only admins can access this
    return { message: 'List of all users', admin: user.name };
  }

  @Get('statistics')
  getStatistics() {
    // Only admins can access this
    return { message: 'System statistics' };
  }

  @Delete('users/:id')
  deleteUser(@Param('id') id: string) {
    // Only admins can delete users
    return { message: `User ${id} deleted` };
  }
}
```

**Guard Order Matters:**
1. `JwtCustomerGuard` - Validates JWT and populates `request.user`
2. `RolesGuard` - Checks `request.user.role` against `@Roles()` decorator

### Example: Mixed Roles Controller

**src/orders/orders.controller.ts**

```ts
import { Controller, Get, Post, Patch, Param, Body, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { AuthenticatedUser } from '../auth/types/auth.types';

@Controller('orders')
@UseGuards(JwtCustomerGuard, RolesGuard)  // Apply to entire controller
export class OrdersController {
  
  @Post()
  @Roles(Role.CUSTOMER)  // Only customers can create orders
  createOrder(@CurrentUser() user: AuthenticatedUser, @Body() createOrderDto) {
    return { message: 'Order created', customerId: user.id };
  }

  @Get('my-orders')
  @Roles(Role.CUSTOMER)  // Only customers can view their orders
  getMyOrders(@CurrentUser() user: AuthenticatedUser) {
    return { message: 'Your orders', customerId: user.id };
  }

  @Get('all')
  @Roles(Role.ADMIN, Role.MANAGER)  // Admins OR managers can view all orders
  getAllOrders() {
    return { message: 'All orders in the system' };
  }

  @Patch(':id/status')
  @Roles(Role.ADMIN, Role.MANAGER, Role.DRIVER)  // Multiple roles allowed
  updateOrderStatus(@Param('id') id: string, @Body() statusDto) {
    return { message: `Order ${id} status updated` };
  }

  @Delete(':id')
  @Roles(Role.ADMIN)  // Only admins can delete orders
  deleteOrder(@Param('id') id: string) {
    return { message: `Order ${id} deleted` };
  }
}
```

**Access Matrix:**

| Endpoint | Admin | Manager | Driver | Customer |
|----------|-------|---------|--------|----------|
| POST /orders | ❌ | ❌ | ❌ | ✅ |
| GET /orders/my-orders | ❌ | ❌ | ❌ | ✅ |
| GET /orders/all | ✅ | ✅ | ❌ | ❌ |
| PATCH /orders/:id/status | ✅ | ✅ | ✅ | ❌ |
| DELETE /orders/:id | ✅ | ❌ | ❌ | ❌ |

### Example: Public + Protected Routes

**src/products/products.controller.ts**

```ts
import { Controller, Get, Post, Patch, Delete, Param, Body, UseGuards } from '@nestjs/common';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Role } from '../auth/enums/role.enum';

@Controller('products')
export class ProductsController {
  
  // Public route - No guards
  @Get()
  getAllProducts() {
    return { message: 'Public product list' };
  }

  // Public route - No guards
  @Get(':id')
  getProduct(@Param('id') id: string) {
    return { message: `Product ${id} details` };
  }

  // Protected route - Admin only
  @Post()
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN, Role.MANAGER)
  createProduct(@Body() createProductDto) {
    return { message: 'Product created' };
  }

  // Protected route - Admin only
  @Patch(':id')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN, Role.MANAGER)
  updateProduct(@Param('id') id: string, @Body() updateProductDto) {
    return { message: `Product ${id} updated` };
  }

  // Protected route - Admin only
  @Delete(':id')
  @UseGuards(JwtCustomerGuard, RolesGuard)
  @Roles(Role.ADMIN)
  deleteProduct(@Param('id') id: string) {
    return { message: `Product ${id} deleted` };
  }
}
```

**Route Types:**
- **Public Routes:** No `@UseGuards()` - anyone can access
- **Protected Routes:** Require authentication + specific roles
- Mix both in same controller by applying guards at method level

## Step 9: Global vs Route-level Guards

### Option 1: Route-level (Recommended)

Apply guards to specific controllers/routes:

```ts
@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)  // Controller-level
@Roles(Role.ADMIN)
export class AdminController {
  // All routes protected
}

@Controller('products')
export class ProductsController {
  
  @Get()  // Public
  getAll() {}

  @Post()
  @UseGuards(JwtCustomerGuard, RolesGuard)  // Route-level
  @Roles(Role.ADMIN)
  create() {}  // Protected
}
```

**Advantages:**
- Fine-grained control
- Mix public and protected routes easily
- Clear which routes are protected
- Easy to maintain

### Option 2: Global Guards

Apply guards globally in `main.ts`:

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

**Advantages:**
- Applied automatically to all routes
- DRY (Don't Repeat Yourself)
- Centralized configuration

**Disadvantages:**
- Need to explicitly mark public routes
- Less flexible
- Harder to debug

### Option 3: Hybrid Approach

Combine global authentication with route-level role checks:

**main.ts:**
```ts
// Global authentication guard
app.useGlobalGuards(new JwtAuthGuard());
```

**Controllers:**
```ts
@Controller('products')
export class ProductsController {
  
  @Get()
  @Public()  // Custom decorator to skip auth
  getAll() {}

  @Post()
  @Roles(Role.ADMIN)  // Add role requirement
  create() {}
}
```

**Custom @Public() decorator:**

**src/auth/decorators/public.decorator.ts**

```ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

## Advanced: Permission-based Access Control

For more granular control, implement permission-based access:

**src/auth/enums/permission.enum.ts**

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

**src/auth/constants/role-permissions.ts**

```ts
import { Role } from '../enums/role.enum';
import { Permission } from '../enums/permission.enum';

export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  [Role.ADMIN]: [
    // Admins have all permissions
    ...Object.values(Permission),
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

**src/auth/decorators/permissions.decorator.ts**

```ts
import { SetMetadata } from '@nestjs/common';
import { Permission } from '../enums/permission.enum';

export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

**src/auth/guards/permissions.guard.ts**

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

    // Get permissions for user's role
    const userPermissions = ROLE_PERMISSIONS[user.role] || [];

    // Check if user has all required permissions
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

**Usage:**

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

## Best Practices

### 1. **Always Use Guards in Correct Order**

```ts
// ✅ Correct Order
@UseGuards(JwtCustomerGuard, RolesGuard)

// ❌ Wrong Order - RolesGuard runs before authentication
@UseGuards(RolesGuard, JwtCustomerGuard)
```

**Why?** JwtCustomerGuard populates `request.user`, which RolesGuard needs.

### 2. **Use Enums for Type Safety**

```ts
// ✅ Good - Type safe, autocomplete works
@Roles(Role.ADMIN, Role.MANAGER)

// ❌ Bad - Prone to typos, no autocomplete
@Roles('admin', 'manager')
```

### 3. **Prefer Class-level Guards When Possible**

```ts
// ✅ Good - DRY, all routes protected
@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)
@Roles(Role.ADMIN)
export class AdminController {
  @Get('users') getUsers() {}
  @Get('stats') getStats() {}
}

// ❌ Repetitive - same guards on every route
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

### 4. **Validate Roles Against Database for Critical Operations**

```ts
@Delete('users/:id')
@Roles(Role.ADMIN)
async deleteUser(@Param('id') id: string, @CurrentUser() user: AuthenticatedUser) {
  // Double-check role from database for critical operations
  const currentUser = await this.userRepository.findOne({ where: { id: user.id } });
  
  if (currentUser.role !== Role.ADMIN) {
    throw new ForbiddenException('Role verification failed');
  }
  
  // Proceed with deletion
  return this.userService.delete(id);
}
```

**Why?** JWT tokens can become stale if user's role changes. For critical operations, verify against database.

### 5. **Use Descriptive Error Messages**

```ts
// ✅ Good - Helpful error message
throw new ForbiddenException(
  `Access denied. Required roles: ${requiredRoles.join(', ')}. Your role: ${user.role}`
);

// ❌ Bad - Generic message
throw new ForbiddenException('Access denied');
```

### 6. **Document Role Requirements**

```ts
/**
 * Delete a user from the system
 * @requires Role.ADMIN - Only administrators can delete users
 * @param id - User ID to delete
 */
@Delete(':id')
@Roles(Role.ADMIN)
async deleteUser(@Param('id') id: string) {
  // ...
}
```

### 7. **Test Role-based Access**

```ts
describe('AdminController', () => {
  it('should allow admin to access dashboard', async () => {
    const adminToken = 'valid-admin-token';
    const response = await request(app.getHttpServer())
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

## Common Pitfalls

### 1. **Forgetting to Apply RolesGuard**

```ts
// ❌ Wrong - @Roles() won't work without RolesGuard
@Controller('admin')
@Roles(Role.ADMIN)
export class AdminController {}

// ✅ Correct
@Controller('admin')
@UseGuards(JwtCustomerGuard, RolesGuard)
@Roles(Role.ADMIN)
export class AdminController {}
```

### 2. **Wrong Guard Order**

```ts
// ❌ Wrong - RolesGuard can't access request.user
@UseGuards(RolesGuard, JwtCustomerGuard)

// ✅ Correct - JwtCustomerGuard populates request.user first
@UseGuards(JwtCustomerGuard, RolesGuard)
```

### 3. **Not Including Role in JWT Strategy**

```ts
// ❌ Wrong - Role not included
async validate(payload: JwtPayload) {
  return {
    id: user.id,
    email: user.email,
    // Missing: role: user.role
  };
}

// ✅ Correct
async validate(payload: JwtPayload) {
  return {
    id: user.id,
    email: user.email,
    role: user.role,  // RolesGuard needs this
  };
}
```

### 4. **Hardcoding Roles**

```ts
// ❌ Bad - Magic strings
@Roles('admin', 'manager')

// ✅ Good - Type-safe enum
@Roles(Role.ADMIN, Role.MANAGER)
```

### 5. **Not Handling Edge Cases**

```ts
// ❌ Bad - Assumes user exists
const hasRole = requiredRoles.some((role) => user.role === role);

// ✅ Good - Checks if user exists
if (!user || !user.role) {
  throw new ForbiddenException('User not authenticated');
}
const hasRole = requiredRoles.some((role) => user.role === role);
```

## Summary

Role-based Access Control in NestJS involves:

1. **Define Roles** - Create enum with all possible roles
2. **Store Roles** - Add role field to user entities
3. **Create Decorator** - `@Roles()` to mark required roles
4. **Create Guard** - `RolesGuard` to verify user has required role
5. **Update Strategies** - Include role in JWT validation
6. **Update Types** - Add role to interfaces
7. **Update Auth Service** - Include role in JWT payload
8. **Apply to Controllers** - Use `@UseGuards()` and `@Roles()`

**Key Concepts:**
- **Roles** = What user types exist (admin, customer, driver)
- **@Roles() Decorator** = Marks endpoints with required roles
- **RolesGuard** = Enforces role requirements
- **JWT Strategy** = Provides user's role to guard
- **Order Matters** = Authentication guard → Role guard

**Security Tips:**
- Always validate JWT before checking roles
- Use enums for type safety
- Double-check roles from DB for critical operations
- Provide clear error messages
- Test role-based access thoroughly

```