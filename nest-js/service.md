# NestJS Service Layer - Complete Guide

## What is a Service?

A **Service** in NestJS is a provider class that contains business logic, data manipulation, and application functionality. Services are the heart of your application where the actual work happens.

**Key Characteristics:**
- Decorated with `@Injectable()` decorator
- Contains business logic (what controllers should NOT have)
- Manages data operations (CRUD operations)
- Can be injected into controllers, other services, guards, interceptors, etc.
- Typically singleton scope (one instance shared across the app)
- Testable and reusable

**Why Services?**
- **Separation of Concerns** - Controllers handle HTTP, services handle logic
- **Reusability** - Services can be used in multiple controllers
- **Testability** - Easy to unit test in isolation
- **Maintainability** - Business logic centralized in one place
- **Dependency Injection** - Easy to mock and swap implementations

---

## Basic Service Structure

### 1. Simple Service (No Database)

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }

  calculateSum(a: number, b: number): number {
    return a + b;
  }
}
```

**Key Points:**
- Must have `@Injectable()` decorator
- Export the class so it can be imported in modules
- Methods contain business logic
- Can be simple (no dependencies) or complex (with dependencies)

---

### 2. Service with Dependencies (Dependency Injection)

```ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { LoggerService } from '../logger/logger.service';

@Injectable()
export class UserService {
  constructor(
    private configService: ConfigService,  // Inject ConfigService
    private logger: LoggerService,          // Inject LoggerService
  ) {}

  getMaxUsers(): number {
    const max = this.configService.get<number>('MAX_USERS');
    this.logger.log(`Max users: ${max}`);
    return max;
  }
}
```

**How Dependency Injection Works:**
1. NestJS sees `UserService` needs `ConfigService` and `LoggerService`
2. It creates instances of those services (or reuses existing ones)
3. Injects them into `UserService` constructor
4. `UserService` can now use them via `this.configService` and `this.logger`

---

## Service with TypeORM (Database Operations)

For this guide, we'll use the entities from our JWT authentication example:
- **Customer Entity** - Represents customers
- **DeliveryDriver Entity** - Represents delivery drivers

### Customer Entity Review

```ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';

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

  @CreateDateColumn()
  created_at: Date;
}
```

### Delivery Driver Entity Review

```ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn } from 'typeorm';

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

  @CreateDateColumn()
  created_at: Date;
}
```

---

## Complete Service Example - Customers Service

### DTOs (Data Transfer Objects)

First, let's create DTOs for our service operations:

**src/customers/dto/create-customer.dto.ts**

```ts
import { IsEmail, IsNotEmpty, IsString, IsOptional, MinLength } from 'class-validator';

export class CreateCustomerDto {
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
```

**src/customers/dto/update-customer.dto.ts**

```ts
import { IsEmail, IsString, IsOptional, MinLength } from 'class-validator';
import { PartialType } from '@nestjs/mapped-types';
import { CreateCustomerDto } from './create-customer.dto';

// PartialType makes all fields optional
export class UpdateCustomerDto extends PartialType(CreateCustomerDto) {
  @IsString()
  @IsOptional()
  name?: string;

  @IsEmail()
  @IsOptional()
  email?: string;

  @IsString()
  @IsOptional()
  address?: string;

  @IsString()
  @IsOptional()
  profile_image_url?: string;
}
```

**src/customers/dto/query-customer.dto.ts**

```ts
import { IsOptional, IsString, IsInt, Min, Max, IsIn } from 'class-validator';
import { Type } from 'class-transformer';

export class QueryCustomerDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  @IsIn(['name', 'email', 'created_at'])
  sortBy?: string = 'created_at';

  @IsOptional()
  @IsString()
  @IsIn(['ASC', 'DESC'])
  sortOrder?: 'ASC' | 'DESC' = 'DESC';
}
```

---

### Customers Service Implementation

**src/customers/customers.service.ts**

```ts
import { Injectable, NotFoundException, ConflictException, BadRequestException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, Like, ILike } from 'typeorm';
import { Customer } from './entities/customer.entity';
import { CreateCustomerDto } from './dto/create-customer.dto';
import { UpdateCustomerDto } from './dto/update-customer.dto';
import { QueryCustomerDto } from './dto/query-customer.dto';
import * as bcrypt from 'bcrypt';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {}

  // ==================== CREATE ====================
  
  /**
   * Create a new customer
   * @param createCustomerDto - Customer data
   * @returns Created customer
   */
  async create(createCustomerDto: CreateCustomerDto): Promise<Customer> {
    // Check if email already exists
    const existingCustomer = await this.customerRepository.findOne({
      where: { email: createCustomerDto.email },
    });

    if (existingCustomer) {
      throw new ConflictException('Email already exists');
    }

    // Hash password
    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(createCustomerDto.password, saltRounds);

    // Create customer entity
    const customer = this.customerRepository.create({
      ...createCustomerDto,
      password: hashedPassword,
    });

    // Save to database
    return await this.customerRepository.save(customer);
  }

  // ==================== READ ====================

  /**
   * Find all customers with pagination and sorting
   * @param queryDto - Query parameters for filtering, pagination, sorting
   * @returns Paginated list of customers
   */
  async findAll(queryDto: QueryCustomerDto) {
    const { search, page, limit, sortBy, sortOrder } = queryDto;

    // Calculate pagination
    const skip = (page - 1) * limit;

    // Build query
    const queryBuilder = this.customerRepository.createQueryBuilder('customer');

    // Apply search filter (if provided)
    if (search) {
      queryBuilder.where(
        'customer.name ILIKE :search OR customer.email ILIKE :search OR customer.address ILIKE :search',
        { search: `%${search}%` }
      );
    }

    // Apply sorting
    queryBuilder.orderBy(`customer.${sortBy}`, sortOrder);

    // Apply pagination
    queryBuilder.skip(skip).take(limit);

    // Execute query
    const [customers, total] = await queryBuilder.getManyAndCount();

    // Calculate pagination metadata
    const totalPages = Math.ceil(total / limit);
    const hasNextPage = page < totalPages;
    const hasPrevPage = page > 1;

    return {
      data: customers,
      meta: {
        total,
        page,
        limit,
        totalPages,
        hasNextPage,
        hasPrevPage,
      },
    };
  }

  /**
   * Find one customer by ID
   * @param id - Customer ID
   * @returns Customer entity
   */
  async findOne(id: string): Promise<Customer> {
    const customer = await this.customerRepository.findOne({
      where: { id },
    });

    if (!customer) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }

    return customer;
  }

  /**
   * Find customer by email
   * @param email - Customer email
   * @returns Customer entity or null
   */
  async findByEmail(email: string): Promise<Customer | null> {
    return await this.customerRepository.findOne({
      where: { email },
    });
  }

  // ==================== UPDATE ====================

  /**
   * Update customer by ID
   * @param id - Customer ID
   * @param updateCustomerDto - Updated customer data
   * @returns Updated customer
   */
  async update(id: string, updateCustomerDto: UpdateCustomerDto): Promise<Customer> {
    // Check if customer exists
    const customer = await this.findOne(id);

    // If email is being updated, check if new email already exists
    if (updateCustomerDto.email && updateCustomerDto.email !== customer.email) {
      const existingCustomer = await this.findByEmail(updateCustomerDto.email);
      if (existingCustomer) {
        throw new ConflictException('Email already exists');
      }
    }

    // Hash password if provided
    if (updateCustomerDto.password) {
      const saltRounds = 10;
      updateCustomerDto.password = await bcrypt.hash(updateCustomerDto.password, saltRounds);
    }

    // Merge and save
    Object.assign(customer, updateCustomerDto);
    return await this.customerRepository.save(customer);
  }

  // ==================== DELETE ====================

  /**
   * Remove customer by ID
   * @param id - Customer ID
   * @returns Deletion result
   */
  async remove(id: string): Promise<void> {
    const customer = await this.findOne(id);
    await this.customerRepository.remove(customer);
  }

  /**
   * Soft delete customer by ID (alternative)
   * @param id - Customer ID
   * @returns Update result
   */
  async softDelete(id: string): Promise<void> {
    const result = await this.customerRepository.softDelete(id);
    
    if (result.affected === 0) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }
  }

  // ==================== ADVANCED QUERIES ====================

  /**
   * Count total customers
   * @returns Total number of customers
   */
  async count(): Promise<number> {
    return await this.customerRepository.count();
  }

  /**
   * Find customers by address (partial match)
   * @param address - Address to search
   * @returns Array of customers
   */
  async findByAddress(address: string): Promise<Customer[]> {
    return await this.customerRepository.find({
      where: {
        address: ILike(`%${address}%`), // Case-insensitive LIKE
      },
    });
  }

  /**
   * Find recently created customers
   * @param limit - Number of customers to return
   * @returns Array of recent customers
   */
  async findRecent(limit: number = 10): Promise<Customer[]> {
    return await this.customerRepository.find({
      order: {
        created_at: 'DESC',
      },
      take: limit,
    });
  }
}
```

---

## Delivery Drivers Service with Advanced Features

**src/delivery-drivers/dto/create-driver.dto.ts**

```ts
import { IsEmail, IsNotEmpty, IsString, IsOptional, MinLength, IsEnum } from 'class-validator';
import { VEHICLE_TYPE } from '../entities/delivery-driver.entity';

export class CreateDriverDto {
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

**src/delivery-drivers/dto/update-driver.dto.ts**

```ts
import { IsEmail, IsString, IsOptional, IsBoolean, IsEnum } from 'class-validator';
import { PartialType } from '@nestjs/mapped-types';
import { CreateDriverDto } from './create-driver.dto';
import { VEHICLE_TYPE } from '../entities/delivery-driver.entity';

export class UpdateDriverDto extends PartialType(CreateDriverDto) {
  @IsString()
  @IsOptional()
  name?: string;

  @IsEmail()
  @IsOptional()
  email?: string;

  @IsString()
  @IsOptional()
  phone?: string;

  @IsEnum(VEHICLE_TYPE)
  @IsOptional()
  vehicle_type?: VEHICLE_TYPE;

  @IsBoolean()
  @IsOptional()
  is_available?: boolean;

  @IsString()
  @IsOptional()
  profile_image_url?: string;
}
```

**src/delivery-drivers/dto/query-driver.dto.ts**

```ts
import { IsOptional, IsString, IsInt, Min, Max, IsIn, IsBoolean } from 'class-validator';
import { Type, Transform } from 'class-transformer';
import { VEHICLE_TYPE } from '../entities/delivery-driver.entity';

export class QueryDriverDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  @IsIn(['name', 'email', 'phone', 'created_at', 'is_available'])
  sortBy?: string = 'created_at';

  @IsOptional()
  @IsString()
  @IsIn(['ASC', 'DESC'])
  sortOrder?: 'ASC' | 'DESC' = 'DESC';

  // Filter by availability
  @IsOptional()
  @Transform(({ value }) => value === 'true' || value === true)
  @IsBoolean()
  is_available?: boolean;

  // Filter by vehicle type
  @IsOptional()
  @IsIn(Object.values(VEHICLE_TYPE))
  vehicle_type?: VEHICLE_TYPE;
}
```

**src/delivery-drivers/delivery-drivers.service.ts**

```ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, ILike } from 'typeorm';
import { DeliveryDriver, VEHICLE_TYPE } from './entities/delivery-driver.entity';
import { CreateDriverDto } from './dto/create-driver.dto';
import { UpdateDriverDto } from './dto/update-driver.dto';
import { QueryDriverDto } from './dto/query-driver.dto';
import * as bcrypt from 'bcrypt';

@Injectable()
export class DeliveryDriversService {
  constructor(
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
  ) {}

  // ==================== CREATE ====================

  async create(createDriverDto: CreateDriverDto): Promise<DeliveryDriver> {
    const existingDriver = await this.driverRepository.findOne({
      where: { email: createDriverDto.email },
    });

    if (existingDriver) {
      throw new ConflictException('Email already exists');
    }

    const saltRounds = 10;
    const hashedPassword = await bcrypt.hash(createDriverDto.password, saltRounds);

    const driver = this.driverRepository.create({
      ...createDriverDto,
      password: hashedPassword,
    });

    return await this.driverRepository.save(driver);
  }

  // ==================== READ WITH ADVANCED FILTERING ====================

  async findAll(queryDto: QueryDriverDto) {
    const { search, page, limit, sortBy, sortOrder, is_available, vehicle_type } = queryDto;

    const skip = (page - 1) * limit;

    const queryBuilder = this.driverRepository.createQueryBuilder('driver');

    // Search filter
    if (search) {
      queryBuilder.where(
        'driver.name ILIKE :search OR driver.email ILIKE :search OR driver.phone ILIKE :search',
        { search: `%${search}%` }
      );
    }

    // Availability filter
    if (is_available !== undefined) {
      queryBuilder.andWhere('driver.is_available = :is_available', { is_available });
    }

    // Vehicle type filter
    if (vehicle_type) {
      queryBuilder.andWhere('driver.vehicle_type = :vehicle_type', { vehicle_type });
    }

    // Sorting
    queryBuilder.orderBy(`driver.${sortBy}`, sortOrder);

    // Pagination
    queryBuilder.skip(skip).take(limit);

    const [drivers, total] = await queryBuilder.getManyAndCount();

    const totalPages = Math.ceil(total / limit);

    return {
      data: drivers,
      meta: {
        total,
        page,
        limit,
        totalPages,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1,
      },
    };
  }

  async findOne(id: string): Promise<DeliveryDriver> {
    const driver = await this.driverRepository.findOne({
      where: { id },
    });

    if (!driver) {
      throw new NotFoundException(`Driver with ID ${id} not found`);
    }

    return driver;
  }

  async findByEmail(email: string): Promise<DeliveryDriver | null> {
    return await this.driverRepository.findOne({
      where: { email },
    });
  }

  // ==================== UPDATE ====================

  async update(id: string, updateDriverDto: UpdateDriverDto): Promise<DeliveryDriver> {
    const driver = await this.findOne(id);

    if (updateDriverDto.email && updateDriverDto.email !== driver.email) {
      const existingDriver = await this.findByEmail(updateDriverDto.email);
      if (existingDriver) {
        throw new ConflictException('Email already exists');
      }
    }

    if (updateDriverDto.password) {
      const saltRounds = 10;
      updateDriverDto.password = await bcrypt.hash(updateDriverDto.password, saltRounds);
    }

    Object.assign(driver, updateDriverDto);
    return await this.driverRepository.save(driver);
  }

  // ==================== DELETE ====================

  async remove(id: string): Promise<void> {
    const driver = await this.findOne(id);
    await this.driverRepository.remove(driver);
  }

  // ==================== DRIVER-SPECIFIC METHODS ====================

  /**
   * Toggle driver availability
   * @param id - Driver ID
   * @returns Updated driver
   */
  async toggleAvailability(id: string): Promise<DeliveryDriver> {
    const driver = await this.findOne(id);
    driver.is_available = !driver.is_available;
    return await this.driverRepository.save(driver);
  }

  /**
   * Find available drivers by vehicle type
   * @param vehicle_type - Type of vehicle
   * @returns Array of available drivers
   */
  async findAvailableByVehicleType(vehicle_type: VEHICLE_TYPE): Promise<DeliveryDriver[]> {
    return await this.driverRepository.find({
      where: {
        is_available: true,
        vehicle_type: vehicle_type,
      },
      order: {
        created_at: 'ASC', // First in, first out
      },
    });
  }

  /**
   * Get statistics about drivers
   * @returns Driver statistics
   */
  async getStatistics() {
    const total = await this.driverRepository.count();
    const available = await this.driverRepository.count({
      where: { is_available: true },
    });
    const unavailable = total - available;

    // Count by vehicle type
    const byVehicleType = await this.driverRepository
      .createQueryBuilder('driver')
      .select('driver.vehicle_type', 'vehicle_type')
      .addSelect('COUNT(*)', 'count')
      .groupBy('driver.vehicle_type')
      .getRawMany();

    return {
      total,
      available,
      unavailable,
      byVehicleType,
    };
  }

  /**
   * Bulk update driver availability
   * @param ids - Array of driver IDs
   * @param is_available - Availability status
   * @returns Update result
   */
  async bulkUpdateAvailability(ids: string[], is_available: boolean) {
    const result = await this.driverRepository
      .createQueryBuilder()
      .update(DeliveryDriver)
      .set({ is_available })
      .where('id IN (:...ids)', { ids })
      .execute();

    return {
      affected: result.affected,
      is_available,
    };
  }
}
```

---

## Pagination with nestjs-typeorm-paginate Package

The `nestjs-typeorm-paginate` package provides a clean, standardized way to handle pagination in NestJS with TypeORM.

### Installation

```bash
npm install nestjs-typeorm-paginate
```

### Benefits

- **Standardized Response Format** - Consistent pagination response structure
- **Less Boilerplate** - Reduces manual pagination code
- **Works with Repository & QueryBuilder** - Flexible implementation
- **Automatic Metadata** - Handles page calculation automatically
- **Link Generation** - Can generate pagination links

---

### Basic Usage with Repository

**src/customers/customers.service.ts**

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { paginate, Pagination, IPaginationOptions } from 'nestjs-typeorm-paginate';
import { Customer } from './entities/customer.entity';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {}

  /**
   * Simple pagination with repository
   */
  async findAll(options: IPaginationOptions): Promise<Pagination<Customer>> {
    return paginate<Customer>(this.customerRepository, options);
  }

  /**
   * Pagination with sorting
   */
  async findAllSorted(
    options: IPaginationOptions,
    sortBy: string = 'created_at',
    sortOrder: 'ASC' | 'DESC' = 'DESC'
  ): Promise<Pagination<Customer>> {
    return paginate<Customer>(this.customerRepository, options, {
      order: {
        [sortBy]: sortOrder,
      },
    });
  }
}
```

**Response Format:**

```json
{
  "items": [
    {
      "id": "uuid-1",
      "name": "John Doe",
      "email": "john@example.com",
      "address": "123 Main St",
      "created_at": "2026-01-01T00:00:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 10,
    "currentPage": 1
  },
  "links": {
    "first": "http://example.com/customers?page=1&limit=10",
    "previous": "",
    "next": "http://example.com/customers?page=2&limit=10",
    "last": "http://example.com/customers?page=10&limit=10"
  }
}
```

---

### Usage with QueryBuilder (Advanced)

**src/customers/dto/pagination-query.dto.ts**

```ts
import { IsOptional, IsInt, Min, Max, IsString, IsIn } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationQueryDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsString()
  @IsIn(['name', 'email', 'created_at'])
  sortBy?: string = 'created_at';

  @IsOptional()
  @IsString()
  @IsIn(['ASC', 'DESC'])
  sortOrder?: 'ASC' | 'DESC' = 'DESC';
}
```

**src/customers/customers.service.ts**

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { paginate, Pagination, IPaginationOptions } from 'nestjs-typeorm-paginate';
import { Customer } from './entities/customer.entity';
import { PaginationQueryDto } from './dto/pagination-query.dto';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {}

  /**
   * Advanced pagination with search and sorting using QueryBuilder
   */
  async findAllAdvanced(query: PaginationQueryDto): Promise<Pagination<Customer>> {
    const { page, limit, search, sortBy, sortOrder } = query;

    const queryBuilder = this.customerRepository.createQueryBuilder('customer');

    // Apply search filter
    if (search) {
      queryBuilder.where(
        'customer.name ILIKE :search OR customer.email ILIKE :search OR customer.address ILIKE :search',
        { search: `%${search}%` }
      );
    }

    // Apply sorting
    queryBuilder.orderBy(`customer.${sortBy}`, sortOrder);

    // Paginate
    return paginate<Customer>(queryBuilder, {
      page,
      limit,
      route: '/customers', // Optional: for generating links
    });
  }

  /**
   * Pagination with relations
   */
  async findAllWithOrders(options: IPaginationOptions): Promise<Pagination<Customer>> {
    const queryBuilder = this.customerRepository
      .createQueryBuilder('customer')
      .leftJoinAndSelect('customer.orders', 'order')
      .orderBy('customer.created_at', 'DESC');

    return paginate<Customer>(queryBuilder, options);
  }

  /**
   * Pagination with custom select fields
   */
  async findAllMinimal(options: IPaginationOptions): Promise<Pagination<Partial<Customer>>> {
    const queryBuilder = this.customerRepository
      .createQueryBuilder('customer')
      .select(['customer.id', 'customer.name', 'customer.email'])
      .orderBy('customer.created_at', 'DESC');

    return paginate(queryBuilder, options);
  }
}
```

---

### Delivery Drivers Service Example

**src/delivery-drivers/dto/driver-pagination-query.dto.ts**

```ts
import { IsOptional, IsInt, Min, Max, IsString, IsIn, IsBoolean } from 'class-validator';
import { Type, Transform } from 'class-transformer';
import { VEHICLE_TYPE } from '../entities/delivery-driver.entity';

export class DriverPaginationQueryDto {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @Transform(({ value }) => value === 'true' || value === true)
  @IsBoolean()
  is_available?: boolean;

  @IsOptional()
  @IsIn(Object.values(VEHICLE_TYPE))
  vehicle_type?: VEHICLE_TYPE;

  @IsOptional()
  @IsString()
  @IsIn(['name', 'email', 'created_at', 'is_available'])
  sortBy?: string = 'created_at';

  @IsOptional()
  @IsString()
  @IsIn(['ASC', 'DESC'])
  sortOrder?: 'ASC' | 'DESC' = 'DESC';
}
```

**src/delivery-drivers/delivery-drivers.service.ts**

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { paginate, Pagination } from 'nestjs-typeorm-paginate';
import { DeliveryDriver } from './entities/delivery-driver.entity';
import { DriverPaginationQueryDto } from './dto/driver-pagination-query.dto';

@Injectable()
export class DeliveryDriversService {
  constructor(
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
  ) {}

  /**
   * Paginate drivers with advanced filters
   */
  async findAll(query: DriverPaginationQueryDto): Promise<Pagination<DeliveryDriver>> {
    const { page, limit, search, is_available, vehicle_type, sortBy, sortOrder } = query;

    const queryBuilder = this.driverRepository.createQueryBuilder('driver');

    // Search filter
    if (search) {
      queryBuilder.where(
        'driver.name ILIKE :search OR driver.email ILIKE :search OR driver.phone ILIKE :search',
        { search: `%${search}%` }
      );
    }

    // Availability filter
    if (is_available !== undefined) {
      queryBuilder.andWhere('driver.is_available = :is_available', { is_available });
    }

    // Vehicle type filter
    if (vehicle_type) {
      queryBuilder.andWhere('driver.vehicle_type = :vehicle_type', { vehicle_type });
    }

    // Sorting
    queryBuilder.orderBy(`driver.${sortBy}`, sortOrder);

    // Paginate
    return paginate<DeliveryDriver>(queryBuilder, {
      page,
      limit,
      route: '/delivery-drivers',
    });
  }

  /**
   * Find available drivers paginated
   */
  async findAvailable(
    page: number = 1,
    limit: number = 10,
    vehicle_type?: VEHICLE_TYPE
  ): Promise<Pagination<DeliveryDriver>> {
    const queryBuilder = this.driverRepository
      .createQueryBuilder('driver')
      .where('driver.is_available = :is_available', { is_available: true });

    if (vehicle_type) {
      queryBuilder.andWhere('driver.vehicle_type = :vehicle_type', { vehicle_type });
    }

    queryBuilder.orderBy('driver.created_at', 'ASC'); // FIFO

    return paginate<DeliveryDriver>(queryBuilder, { page, limit });
  }
}
```

---

### Controller Implementation

**src/customers/customers.controller.ts**

```ts
import { Controller, Get, Query, UseGuards } from '@nestjs/common';
import { CustomersService } from './customers.service';
import { Pagination } from 'nestjs-typeorm-paginate';
import { Customer } from './entities/customer.entity';
import { PaginationQueryDto } from './dto/pagination-query.dto';
import { JwtCustomerGuard } from '../auth/guards/jwt-customer.guard';

@Controller('customers')
@UseGuards(JwtCustomerGuard)
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Get()
  async findAll(@Query() query: PaginationQueryDto): Promise<Pagination<Customer>> {
    return this.customersService.findAllAdvanced(query);
  }

  @Get('minimal')
  async findAllMinimal(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 10
  ): Promise<Pagination<Partial<Customer>>> {
    return this.customersService.findAllMinimal({ page, limit });
  }
}
```

**src/delivery-drivers/delivery-drivers.controller.ts**

```ts
import { Controller, Get, Query, UseGuards } from '@nestjs/common';
import { DeliveryDriversService } from './delivery-drivers.service';
import { Pagination } from 'nestjs-typeorm-paginate';
import { DeliveryDriver } from './entities/delivery-driver.entity';
import { DriverPaginationQueryDto } from './dto/driver-pagination-query.dto';
import { JwtDriverGuard } from '../auth/guards/jwt-driver.guard';

@Controller('delivery-drivers')
@UseGuards(JwtDriverGuard)
export class DeliveryDriversController {
  constructor(private readonly driversService: DeliveryDriversService) {}

  @Get()
  async findAll(@Query() query: DriverPaginationQueryDto): Promise<Pagination<DeliveryDriver>> {
    return this.driversService.findAll(query);
  }

  @Get('available')
  async findAvailable(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 10,
    @Query('vehicle_type') vehicle_type?: string
  ): Promise<Pagination<DeliveryDriver>> {
    return this.driversService.findAvailable(page, limit, vehicle_type as any);
  }
}
```

---

### API Usage Examples

**1. Basic Pagination:**
```
GET /customers?page=1&limit=10
```

**2. With Search:**
```
GET /customers?page=1&limit=10&search=john
```

**3. With Sorting:**
```
GET /customers?page=1&limit=10&sortBy=name&sortOrder=ASC
```

**4. Complex Query:**
```
GET /delivery-drivers?page=2&limit=20&is_available=true&vehicle_type=bike&sortBy=created_at&sortOrder=DESC
```

**5. Search Available Drivers:**
```
GET /delivery-drivers?search=john&is_available=true
```

---

### Custom Pagination Options

```ts
import { IPaginationOptions } from 'nestjs-typeorm-paginate';

const customOptions: IPaginationOptions = {
  page: 1,
  limit: 10,
  route: 'http://example.com/customers', // For link generation
  cacheQueries: true, // Enable query caching
};

await paginate<Customer>(repository, customOptions);
```

---

### Pagination Metadata Explained

```ts
{
  "items": [...],          // Array of entities
  "meta": {
    "totalItems": 100,     // Total records in database
    "itemCount": 10,       // Items in current page
    "itemsPerPage": 10,    // Requested limit
    "totalPages": 10,      // Total pages available
    "currentPage": 1       // Current page number
  },
  "links": {
    "first": "...",        // First page URL
    "previous": "...",     // Previous page URL (empty if first page)
    "next": "...",         // Next page URL (empty if last page)
    "last": "..."          // Last page URL
  }
}
```

---

### Understanding QueryBuilder Aliases vs Table Names

**IMPORTANT:** The string you pass to `createQueryBuilder()` is an **alias**, NOT the table name!

```ts
// Entity definition - THIS defines the table name
@Entity('customers')  // ← Table name in database
export class Customer {
  // ...
}

// QueryBuilder - THIS is just an alias
const queryBuilder = this.customerRepository.createQueryBuilder('customer');
//                                                                 ^^^^^^^^
//                                                            This is an ALIAS

// You use the alias to reference the entity in the query
queryBuilder.where('customer.name = :name', { name: 'John' });
//                  ^^^^^^^^ Using the alias
```

**How It Works:**

| Component | Purpose | Example |
|-----------|---------|---------|
| `@Entity('customers')` | **Table name** in database | Table: `customers` |
| `.createQueryBuilder('customer')` | **Alias** for queries | Alias: `customer` |
| `customer.name` | Reference column using alias | Column: `name` |

**Real Example:**

```ts
// Entity with table name
@Entity('delivery_drivers')  // Table name: delivery_drivers
export class DeliveryDriver {
  @Column()
  name: string;
}

// Service using QueryBuilder
@Injectable()
export class DeliveryDriversService {
  async findAll() {
    // 'driver' is just an alias - you can name it anything!
    const queryBuilder = this.driverRepository.createQueryBuilder('driver');
    
    // Generated SQL will use the actual table name
    // SELECT * FROM delivery_drivers WHERE driver.name = 'John'
    queryBuilder.where('driver.name = :name', { name: 'John' });
    
    return queryBuilder.getMany();
  }
}
```

**Different Alias Examples:**

```ts
// All of these are valid - alias can be anything
this.customerRepository.createQueryBuilder('customer');  // ✅ Common
this.customerRepository.createQueryBuilder('c');         // ✅ Short
this.customerRepository.createQueryBuilder('cust');      // ✅ Abbreviated
this.customerRepository.createQueryBuilder('user');      // ✅ Works but confusing!

// The alias is what you use in your query methods
queryBuilder.where('customer.email = :email');  // Using 'customer' alias
queryBuilder.where('c.email = :email');        // Using 'c' alias
```

**With Joins:**

```ts
const queryBuilder = this.customerRepository
  .createQueryBuilder('customer')  // Main entity alias
  .leftJoinAndSelect('customer.orders', 'order')  // Related entity alias
  .where('customer.name = :name', { name: 'John' })
  .andWhere('order.total > :amount', { amount: 100 });

// Generated SQL:
// SELECT * FROM customers AS customer
// LEFT JOIN orders AS order ON order.customer_id = customer.id
// WHERE customer.name = 'John' AND order.total > 100
```

**Best Practices:**

1. **Use descriptive aliases** - `customer` is better than `c`
2. **Be consistent** - Use the same alias pattern throughout your codebase
3. **Match entity name** - If entity is `Customer`, alias could be `customer` (lowercase)
4. **Avoid confusion** - Don't use misleading aliases

**Common Pattern:**

```ts
// Singular lowercase of entity name
Customer entity → 'customer' alias
DeliveryDriver entity → 'driver' alias (or 'deliveryDriver')
Order entity → 'order' alias
```

---

### Comparison: Manual vs Package

**Manual Pagination:**
```ts
// More code, manual calculations
const skip = (page - 1) * limit;
const [data, total] = await repository.findAndCount({ skip, take: limit });
const totalPages = Math.ceil(total / limit);
return { data, meta: { total, page, limit, totalPages } };
```

**With nestjs-typeorm-paginate:**
```ts
// Clean, automatic calculations
return paginate<Customer>(repository, { page, limit });
```

---

### Advanced: Custom Response Serialization

**src/common/serializers/pagination.serializer.ts**

```ts
import { Pagination } from 'nestjs-typeorm-paginate';

export class PaginationSerializer {
  static serialize<T>(pagination: Pagination<T>, transformer?: (item: T) => any) {
    return {
      success: true,
      data: transformer 
        ? pagination.items.map(transformer) 
        : pagination.items,
      pagination: {
        total: pagination.meta.totalItems,
        count: pagination.meta.itemCount,
        perPage: pagination.meta.itemsPerPage,
        currentPage: pagination.meta.currentPage,
        totalPages: pagination.meta.totalPages,
        hasMorePages: pagination.meta.currentPage < pagination.meta.totalPages,
      },
      links: pagination.links,
    };
  }
}
```

**Usage:**

```ts
@Get()
async findAll(@Query() query: PaginationQueryDto) {
  const pagination = await this.customersService.findAllAdvanced(query);
  
  return PaginationSerializer.serialize(pagination, (customer) => ({
    id: customer.id,
    name: customer.name,
    email: customer.email,
    // Exclude password and other sensitive fields
  }));
}
```

---

### Performance Tips

1. **Use Select to Limit Fields:**
```ts
queryBuilder.select(['driver.id', 'driver.name', 'driver.is_available']);
```

2. **Add Indexes:**
```ts
@Entity('customers')
@Index(['email'])
@Index(['created_at'])
export class Customer {
  // ...
}
```

3. **Limit Max Page Size:**
```ts
@Max(100) // Prevent users from requesting 1000+ items
limit?: number = 10;
```

4. **Use Query Caching:**
```ts
return paginate<Customer>(queryBuilder, {
  page,
  limit,
  cacheQueries: true,
});
```

---

## Pagination Patterns

### 1. Offset-Based Pagination (Most Common)

**Pros:**
- Easy to implement
- Jump to any page
- Simple UI (page numbers)

**Cons:**
- Performance degrades with large offsets
- Data inconsistency if records are added/deleted during pagination

```ts
async findAllWithPagination(page: number = 1, limit: number = 10) {
  const skip = (page - 1) * limit;

  const [data, total] = await this.repository.findAndCount({
    skip: skip,
    take: limit,
    order: { created_at: 'DESC' },
  });

  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNextPage: page < Math.ceil(total / limit),
      hasPrevPage: page > 1,
    },
  };
}
```

### 2. Cursor-Based Pagination (Better for Infinite Scroll)

**Pros:**
- Consistent performance
- No data duplication/skipping
- Good for real-time data

**Cons:**
- Can't jump to specific page
- More complex implementation

```ts
async findAllWithCursor(cursor?: string, limit: number = 10) {
  const queryBuilder = this.repository.createQueryBuilder('entity');

  // If cursor is provided, fetch records after it
  if (cursor) {
    queryBuilder.where('entity.id > :cursor', { cursor });
  }

  queryBuilder
    .orderBy('entity.id', 'ASC')
    .take(limit + 1); // Fetch one extra to check if there's a next page

  const data = await queryBuilder.getMany();
  const hasNextPage = data.length > limit;

  // Remove the extra item if it exists
  if (hasNextPage) {
    data.pop();
  }

  const nextCursor = hasNextPage ? data[data.length - 1].id : null;

  return {
    data,
    meta: {
      nextCursor,
      hasNextPage,
      limit,
    },
  };
}
```

### 3. Page-Based with Query Builder (Most Flexible)

```ts
async findAllAdvanced(queryDto: QueryDto) {
  const { page = 1, limit = 10, sortBy = 'created_at', sortOrder = 'DESC', search } = queryDto;

  const skip = (page - 1) * limit;

  const queryBuilder = this.repository.createQueryBuilder('entity');

  // Dynamic search
  if (search) {
    queryBuilder.where(
      'entity.name ILIKE :search OR entity.email ILIKE :search',
      { search: `%${search}%` }
    );
  }

  // Dynamic sorting
  const allowedSortFields = ['name', 'email', 'created_at'];
  if (allowedSortFields.includes(sortBy)) {
    queryBuilder.orderBy(`entity.${sortBy}`, sortOrder);
  }

  // Pagination
  queryBuilder.skip(skip).take(limit);

  const [data, total] = await queryBuilder.getManyAndCount();

  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNextPage: page < Math.ceil(total / limit),
      hasPrevPage: page > 1,
    },
  };
}
```

---

## Sorting Patterns

### 1. Simple Sorting

```ts
// Single field, single direction
async findAll() {
  return await this.repository.find({
    order: {
      created_at: 'DESC',
    },
  });
}
```

### 2. Multiple Field Sorting

```ts
// Multiple fields with different directions
async findAll() {
  return await this.repository.find({
    order: {
      is_available: 'DESC', // Available first
      created_at: 'ASC',    // Then oldest first
    },
  });
}
```

### 3. Dynamic Sorting (Query Parameter)

```ts
async findAll(sortBy: string = 'created_at', sortOrder: 'ASC' | 'DESC' = 'DESC') {
  // Whitelist allowed sort fields (security!)
  const allowedSortFields = ['name', 'email', 'created_at', 'is_available'];
  
  if (!allowedSortFields.includes(sortBy)) {
    sortBy = 'created_at'; // Default fallback
  }

  return await this.repository.find({
    order: {
      [sortBy]: sortOrder,
    },
  });
}
```

### 4. Complex Sorting with Relations

```ts
// Sort by related entity field
async findAllWithRelations() {
  return await this.repository
    .createQueryBuilder('customer')
    .leftJoinAndSelect('customer.orders', 'order')
    .orderBy('order.total', 'DESC') // Sort by order total
    .addOrderBy('customer.name', 'ASC') // Then by customer name
    .getMany();
}
```

### 5. Case-Insensitive Sorting

```ts
async findAllCaseInsensitive() {
  return await this.repository
    .createQueryBuilder('entity')
    .orderBy('LOWER(entity.name)', 'ASC')
    .getMany();
}
```

---

## Advanced Query Patterns

### 1. Search with Multiple Fields

```ts
async search(searchTerm: string) {
  return await this.repository
    .createQueryBuilder('customer')
    .where(
      'customer.name ILIKE :term OR customer.email ILIKE :term OR customer.address ILIKE :term',
      { term: `%${searchTerm}%` }
    )
    .getMany();
}
```

### 2. Date Range Filtering

```ts
async findByDateRange(startDate: Date, endDate: Date) {
  return await this.repository
    .createQueryBuilder('entity')
    .where('entity.created_at BETWEEN :start AND :end', {
      start: startDate,
      end: endDate,
    })
    .getMany();
}
```

### 3. Complex Filtering

```ts
async findWithFilters(filters: any) {
  const queryBuilder = this.repository.createQueryBuilder('driver');

  // Apply filters conditionally
  if (filters.is_available !== undefined) {
    queryBuilder.andWhere('driver.is_available = :is_available', {
      is_available: filters.is_available,
    });
  }

  if (filters.vehicle_type) {
    queryBuilder.andWhere('driver.vehicle_type = :vehicle_type', {
      vehicle_type: filters.vehicle_type,
    });
  }

  if (filters.minDate) {
    queryBuilder.andWhere('driver.created_at >= :minDate', {
      minDate: filters.minDate,
    });
  }

  return await queryBuilder.getMany();
}
```

### 4. Aggregation Queries

```ts
async getAggregatedData() {
  const result = await this.repository
    .createQueryBuilder('driver')
    .select('driver.vehicle_type', 'vehicle_type')
    .addSelect('COUNT(*)', 'total')
    .addSelect('COUNT(CASE WHEN driver.is_available = true THEN 1 END)', 'available')
    .groupBy('driver.vehicle_type')
    .getRawMany();

  return result;
}
```

---

## Service Best Practices

### 1. Error Handling

```ts
async findOne(id: string): Promise<Customer> {
  try {
    const customer = await this.customerRepository.findOne({
      where: { id },
    });

    if (!customer) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }

    return customer;
  } catch (error) {
    if (error instanceof NotFoundException) {
      throw error;
    }
    throw new InternalServerErrorException('Failed to fetch customer');
  }
}
```

### 2. Transaction Management

```ts
import { DataSource } from 'typeorm';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
    private dataSource: DataSource,
  ) {}

  async createCustomerWithOrder(customerData: any, orderData: any) {
    const queryRunner = this.dataSource.createQueryRunner();

    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // Create customer
      const customer = queryRunner.manager.create(Customer, customerData);
      await queryRunner.manager.save(customer);

      // Create order
      const order = queryRunner.manager.create(Order, {
        ...orderData,
        customer: customer,
      });
      await queryRunner.manager.save(order);

      await queryRunner.commitTransaction();

      return { customer, order };
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

### 3. Caching

```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CustomersService {
  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
    @Inject(CACHE_MANAGER)
    private cacheManager: Cache,
  ) {}

  async findOne(id: string): Promise<Customer> {
    // Try cache first
    const cacheKey = `customer:${id}`;
    const cached = await this.cacheManager.get<Customer>(cacheKey);

    if (cached) {
      return cached;
    }

    // If not in cache, fetch from database
    const customer = await this.customerRepository.findOne({
      where: { id },
    });

    if (!customer) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }

    // Store in cache
    await this.cacheManager.set(cacheKey, customer, 300); // 5 minutes TTL

    return customer;
  }

  async update(id: string, updateDto: UpdateCustomerDto): Promise<Customer> {
    const customer = await this.findOne(id);
    Object.assign(customer, updateDto);
    const updated = await this.customerRepository.save(customer);

    // Invalidate cache
    await this.cacheManager.del(`customer:${id}`);

    return updated;
  }
}
```

### 4. Logging

```ts
import { Logger } from '@nestjs/common';

@Injectable()
export class CustomersService {
  private readonly logger = new Logger(CustomersService.name);

  constructor(
    @InjectRepository(Customer)
    private customerRepository: Repository<Customer>,
  ) {}

  async create(createCustomerDto: CreateCustomerDto): Promise<Customer> {
    this.logger.log(`Creating customer with email: ${createCustomerDto.email}`);

    try {
      const customer = await this.customerRepository.save(
        this.customerRepository.create(createCustomerDto)
      );

      this.logger.log(`Customer created successfully: ${customer.id}`);
      return customer;
    } catch (error) {
      this.logger.error(`Failed to create customer: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

### 5. Validation

```ts
async update(id: string, updateDto: UpdateCustomerDto): Promise<Customer> {
  // Validate ID format
  if (!this.isValidUUID(id)) {
    throw new BadRequestException('Invalid ID format');
  }

  const customer = await this.findOne(id);

  // Validate email uniqueness if being updated
  if (updateDto.email && updateDto.email !== customer.email) {
    const existing = await this.findByEmail(updateDto.email);
    if (existing) {
      throw new ConflictException('Email already exists');
    }
  }

  Object.assign(customer, updateDto);
  return await this.customerRepository.save(customer);
}

private isValidUUID(id: string): boolean {
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  return uuidRegex.test(id);
}
```

---

## Service Testing

### Unit Test Example

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { CustomersService } from './customers.service';
import { Customer } from './entities/customer.entity';
import { Repository } from 'typeorm';
import { NotFoundException } from '@nestjs/common';

describe('CustomersService', () => {
  let service: CustomersService;
  let repository: Repository<Customer>;

  const mockCustomer = {
    id: '123',
    name: 'John Doe',
    email: 'john@example.com',
    password: 'hashed_password',
    address: '123 Main St',
    created_at: new Date(),
  };

  const mockRepository = {
    findOne: jest.fn(),
    find: jest.fn(),
    save: jest.fn(),
    create: jest.fn(),
    remove: jest.fn(),
    count: jest.fn(),
    createQueryBuilder: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CustomersService,
        {
          provide: getRepositoryToken(Customer),
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<CustomersService>(CustomersService);
    repository = module.get<Repository<Customer>>(getRepositoryToken(Customer));
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('findOne', () => {
    it('should return a customer when found', async () => {
      mockRepository.findOne.mockResolvedValue(mockCustomer);

      const result = await service.findOne('123');

      expect(result).toEqual(mockCustomer);
      expect(mockRepository.findOne).toHaveBeenCalledWith({
        where: { id: '123' },
      });
    });

    it('should throw NotFoundException when customer not found', async () => {
      mockRepository.findOne.mockResolvedValue(null);

      await expect(service.findOne('999')).rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    it('should create and return a customer', async () => {
      const createDto = {
        name: 'John Doe',
        email: 'john@example.com',
        password: 'password123',
        address: '123 Main St',
      };

      mockRepository.create.mockReturnValue(mockCustomer);
      mockRepository.save.mockResolvedValue(mockCustomer);

      const result = await service.create(createDto);

      expect(result).toEqual(mockCustomer);
      expect(mockRepository.save).toHaveBeenCalled();
    });
  });
});
```

---

## Key Takeaways

1. **Services = Business Logic** - Keep controllers thin, services fat
2. **Use DTOs** - Always validate input with DTOs
3. **Error Handling** - Use proper HTTP exceptions (NotFoundException, ConflictException)
4. **Pagination** - Always paginate list endpoints
5. **Sorting** - Make sorting dynamic and flexible
6. **Filtering** - Use QueryBuilder for complex filters
7. **Transactions** - Use for multi-step operations
8. **Caching** - Cache frequently accessed data
9. **Logging** - Log important operations
10. **Testing** - Unit test all service methods

---

## Common Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| **Offset Pagination** | Standard list views | `skip: (page-1)*limit` |
| **Cursor Pagination** | Infinite scroll | `where: { id: MoreThan(cursor) }` |
| **Dynamic Sorting** | Flexible ordering | `order: { [sortBy]: sortOrder }` |
| **Search/Filter** | Find records | `where: { name: ILike('%term%') }` |
| **Bulk Operations** | Update many records | `update().where('id IN (:...ids)')` |
| **Aggregation** | Statistics | `groupBy().count()` |
| **Transactions** | Atomic operations | `queryRunner.startTransaction()` |
| **Caching** | Performance | `cacheManager.get/set()` |

This service layer pattern provides a solid foundation for building scalable NestJS applications!
