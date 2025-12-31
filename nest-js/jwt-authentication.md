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