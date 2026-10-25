# NestJS Service Layer

Writing services: structure and dependency injection, TypeORM CRUD, DTOs, pagination, sorting, filtering, aggregation, and service-level concerns (errors, transactions, caching, logging, testing).

**Contents**

1. [What is a service](#what-is-a-service)
2. [Basic service structure](#basic-service-structure)
3. [Entities used in this guide](#entities-used-in-this-guide)
4. [Customers service](#customers-service)
5. [Delivery drivers service](#delivery-drivers-service)
6. [Pagination with nestjs-typeorm-paginate](#pagination-with-nestjs-typeorm-paginate)
7. [QueryBuilder aliases vs table names](#querybuilder-aliases-vs-table-names)
8. [Pagination patterns](#pagination-patterns)
9. [Sorting patterns](#sorting-patterns)
10. [Advanced query patterns](#advanced-query-patterns)
11. [Service best practices](#service-best-practices)
12. [Service testing](#service-testing)
13. [Key takeaways](#key-takeaways)
14. [Common patterns summary](#common-patterns-summary)
15. [Common mistakes](#common-mistakes)

Related: [Controllers](controller.md) · [Custom Exceptions](custom-exception.md) · [TypeORM + Supabase](typeorm-config.md) · [JWT Authentication](jwt-authentication.md)

---

## What is a service

A **service** is a provider class that holds business logic, data manipulation and application functionality. Controllers handle HTTP; services do the actual work.

| Characteristic | Detail |
|----------------|--------|
| Decorator | `@Injectable()` |
| Contains | Business logic (what controllers should **not** have) and data operations (CRUD). |
| Injected into | Controllers, other services, guards, interceptors, etc. |
| Scope | Singleton by default (one instance shared across the app). |

| Why services | Benefit |
|--------------|---------|
| Separation of concerns | Controllers handle HTTP, services handle logic. |
| Reusability | One service can serve several controllers. |
| Testability | Easy to unit test in isolation. |
| Maintainability | Business logic lives in one place. |
| Dependency injection | Easy to mock and swap implementations. |

---

## Basic service structure

### Simple service (no dependencies)

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

- Must have `@Injectable()`.
- Must be exported so a module can list it in `providers`.
- Methods contain the business logic.

### Service with dependencies (dependency injection)

```ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { LoggerService } from '../logger/logger.service';

@Injectable()
export class UserService {
  constructor(
    private configService: ConfigService,  // injected
    private logger: LoggerService,         // injected
  ) {}

  getMaxUsers(): number {
    const max = this.configService.get<number>('MAX_USERS');
    this.logger.log(`Max users: ${max}`);
    return max;
  }
}
```

**How injection works**

1. Nest sees that `UserService` needs `ConfigService` and `LoggerService`.
2. It creates those instances (or reuses existing ones).
3. It passes them into the constructor.
4. `UserService` uses them through `this.configService` and `this.logger`.

---

## Entities used in this guide

The examples use the entities from the JWT authentication guide (`jwt-authentication.md`).

### Customer

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

### Delivery driver

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

  @Column({ type: 'enum', enum: VEHICLE_TYPE })
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

## Customers service

### DTOs

`src/customers/dto/create-customer.dto.ts`

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

`src/customers/dto/update-customer.dto.ts`

```ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateCustomerDto } from './create-customer.dto';

// PartialType copies every field of CreateCustomerDto, with its validators, and makes all of them optional
export class UpdateCustomerDto extends PartialType(CreateCustomerDto) {}
```

> `PartialType` already marks each field optional and keeps its validation decorators, so redeclaring the fields (with `@IsOptional()`) is not needed. Add fields only for properties that do not exist on the create DTO (e.g. `is_available` for drivers).

`src/customers/dto/query-customer.dto.ts`

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

| Decorator | Purpose |
|-----------|---------|
| `@Type(() => Number)` | Converts the query string value (always a string) to a number. |
| `@IsIn([...])` | Whitelists values, so `sortBy` cannot inject arbitrary column names. |
| `= 1`, `= 10`, ... | Defaults used when the parameter is missing (needs `transform: true` in `ValidationPipe`). |

### Service

`src/customers/customers.service.ts`

```ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, ILike } from 'typeorm';
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

  /** Create a new customer. */
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

    // Create entity, then save
    const customer = this.customerRepository.create({
      ...createCustomerDto,
      password: hashedPassword,
    });

    return await this.customerRepository.save(customer);
  }

  // ==================== READ ====================

  /** Find all customers with search, pagination and sorting. */
  async findAll(queryDto: QueryCustomerDto) {
    const { search, page, limit, sortBy, sortOrder } = queryDto;

    const skip = (page - 1) * limit;

    const queryBuilder = this.customerRepository.createQueryBuilder('customer');

    // Search filter (if provided)
    if (search) {
      queryBuilder.where(
        'customer.name ILIKE :search OR customer.email ILIKE :search OR customer.address ILIKE :search',
        { search: `%${search}%` },
      );
    }

    queryBuilder.orderBy(`customer.${sortBy}`, sortOrder);
    queryBuilder.skip(skip).take(limit);

    const [customers, total] = await queryBuilder.getManyAndCount();

    const totalPages = Math.ceil(total / limit);

    return {
      data: customers,
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

  /** Find one customer by ID, or throw 404. */
  async findOne(id: string): Promise<Customer> {
    const customer = await this.customerRepository.findOne({ where: { id } });

    if (!customer) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }

    return customer;
  }

  /** Find a customer by email, or null. */
  async findByEmail(email: string): Promise<Customer | null> {
    return await this.customerRepository.findOne({ where: { email } });
  }

  // ==================== UPDATE ====================

  /** Update a customer by ID. */
  async update(id: string, updateCustomerDto: UpdateCustomerDto): Promise<Customer> {
    const customer = await this.findOne(id);

    // If the email changes, make sure the new one is free
    if (updateCustomerDto.email && updateCustomerDto.email !== customer.email) {
      const existingCustomer = await this.findByEmail(updateCustomerDto.email);
      if (existingCustomer) {
        throw new ConflictException('Email already exists');
      }
    }

    // Hash the password if provided
    if (updateCustomerDto.password) {
      const saltRounds = 10;
      updateCustomerDto.password = await bcrypt.hash(updateCustomerDto.password, saltRounds);
    }

    // Merge and save
    Object.assign(customer, updateCustomerDto);
    return await this.customerRepository.save(customer);
  }

  // ==================== DELETE ====================

  /** Hard delete a customer by ID. */
  async remove(id: string): Promise<void> {
    const customer = await this.findOne(id);
    await this.customerRepository.remove(customer);
  }

  /** Soft delete (alternative). Requires a @DeleteDateColumn() on the entity. */
  async softDelete(id: string): Promise<void> {
    const result = await this.customerRepository.softDelete(id);

    if (result.affected === 0) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }
  }

  // ==================== ADVANCED QUERIES ====================

  /** Count total customers. */
  async count(): Promise<number> {
    return await this.customerRepository.count();
  }

  /** Find customers by address (case-insensitive partial match). */
  async findByAddress(address: string): Promise<Customer[]> {
    return await this.customerRepository.find({
      where: { address: ILike(`%${address}%`) },
    });
  }

  /** Find the most recently created customers. */
  async findRecent(limit: number = 10): Promise<Customer[]> {
    return await this.customerRepository.find({
      order: { created_at: 'DESC' },
      take: limit,
    });
  }
}
```

### Method reference

| Method | Purpose | Throws |
|--------|---------|--------|
| `create(dto)` | Create with hashed password | `ConflictException` (email exists) |
| `findAll(query)` | Search, sort, paginate | none |
| `findOne(id)` | Get by ID | `NotFoundException` |
| `findByEmail(email)` | Get by email, or `null` | none |
| `update(id, dto)` | Partial update, re-hash password | `NotFoundException`, `ConflictException` |
| `remove(id)` | Hard delete | `NotFoundException` |
| `softDelete(id)` | Mark as deleted | `NotFoundException` |
| `count()` | Total rows | none |
| `findByAddress(address)` | Partial address match | none |
| `findRecent(limit)` | Newest first | none |

### Repository methods used

| Method | Description |
|--------|-------------|
| `create(obj)` | Builds an entity instance (does not save). |
| `save(entity)` | Inserts or updates. |
| `findOne({ where })` | First match or `null`. |
| `find({ where, order, take })` | Many results. |
| `remove(entity)` | Hard delete of an entity instance. |
| `softDelete(id)` | Sets the delete-date column. |
| `count()` | Number of rows. |
| `createQueryBuilder(alias)` | Starts a query builder. |
| `ILike(pattern)` | Case-insensitive `LIKE` (PostgreSQL). |

---

## Delivery drivers service

### DTOs

`src/delivery-drivers/dto/create-driver.dto.ts`

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

`src/delivery-drivers/dto/update-driver.dto.ts`

```ts
import { IsBoolean, IsOptional } from 'class-validator';
import { PartialType } from '@nestjs/mapped-types';
import { CreateDriverDto } from './create-driver.dto';

export class UpdateDriverDto extends PartialType(CreateDriverDto) {
  // Not part of CreateDriverDto, so it is declared here
  @IsBoolean()
  @IsOptional()
  is_available?: boolean;
}
```

`src/delivery-drivers/dto/query-driver.dto.ts`

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

  // Filter by availability ("true" in the query string becomes boolean true)
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

### Service

`src/delivery-drivers/delivery-drivers.service.ts`

```ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
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
        { search: `%${search}%` },
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

    queryBuilder.orderBy(`driver.${sortBy}`, sortOrder);
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
    const driver = await this.driverRepository.findOne({ where: { id } });

    if (!driver) {
      throw new NotFoundException(`Driver with ID ${id} not found`);
    }

    return driver;
  }

  async findByEmail(email: string): Promise<DeliveryDriver | null> {
    return await this.driverRepository.findOne({ where: { email } });
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

  /** Toggle driver availability. */
  async toggleAvailability(id: string): Promise<DeliveryDriver> {
    const driver = await this.findOne(id);
    driver.is_available = !driver.is_available;
    return await this.driverRepository.save(driver);
  }

  /** Find available drivers by vehicle type (oldest first). */
  async findAvailableByVehicleType(vehicle_type: VEHICLE_TYPE): Promise<DeliveryDriver[]> {
    return await this.driverRepository.find({
      where: { is_available: true, vehicle_type },
      order: { created_at: 'ASC' }, // first in, first out
    });
  }

  /** Driver statistics. */
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

    return { total, available, unavailable, byVehicleType };
  }

  /** Bulk update availability for many drivers. */
  async bulkUpdateAvailability(ids: string[], is_available: boolean) {
    const result = await this.driverRepository
      .createQueryBuilder()
      .update(DeliveryDriver)
      .set({ is_available })
      .where('id IN (:...ids)', { ids })
      .execute();

    return { affected: result.affected, is_available };
  }
}
```

### Driver-specific methods

| Method | Purpose |
|--------|---------|
| `toggleAvailability(id)` | Flip `is_available`. |
| `findAvailableByVehicleType(type)` | Available drivers of one vehicle type, oldest first. |
| `getStatistics()` | Totals, available/unavailable, and counts per vehicle type. |
| `bulkUpdateAvailability(ids, flag)` | One `UPDATE ... WHERE id IN (...)` for many drivers. |

---

## Pagination with nestjs-typeorm-paginate

The `nestjs-typeorm-paginate` package gives a standard pagination response with little boilerplate.

```bash
npm install nestjs-typeorm-paginate
```

| Benefit | Description |
|---------|-------------|
| Standard response | `items`, `meta` and `links` in a consistent shape. |
| Less boilerplate | No manual skip/take/total math. |
| Flexible | Works with a repository **or** a QueryBuilder. |
| Automatic metadata | Page counts calculated for you. |
| Links | Can generate first/previous/next/last URLs. |

### Basic usage with a repository

`src/customers/customers.service.ts`

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

  /** Simple pagination. */
  async findAll(options: IPaginationOptions): Promise<Pagination<Customer>> {
    return paginate<Customer>(this.customerRepository, options);
  }

  /** Pagination with sorting. */
  async findAllSorted(
    options: IPaginationOptions,
    sortBy: string = 'created_at',
    sortOrder: 'ASC' | 'DESC' = 'DESC',
  ): Promise<Pagination<Customer>> {
    return paginate<Customer>(this.customerRepository, options, {
      order: { [sortBy]: sortOrder },
    });
  }
}
```

### Response format

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

| Field | Meaning |
|-------|---------|
| `items` | Entities on the current page. |
| `meta.totalItems` | Total records in the database. |
| `meta.itemCount` | Items on the current page. |
| `meta.itemsPerPage` | The requested limit. |
| `meta.totalPages` | Pages available. |
| `meta.currentPage` | Current page number. |
| `links.first` / `last` | First and last page URLs. |
| `links.previous` / `next` | Adjacent page URLs (empty on the first / last page). |

### Usage with QueryBuilder (advanced)

`src/customers/dto/pagination-query.dto.ts`

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

`src/customers/customers.service.ts`

```ts
import { paginate, Pagination, IPaginationOptions } from 'nestjs-typeorm-paginate';
import { PaginationQueryDto } from './dto/pagination-query.dto';

/** Advanced pagination with search and sorting. */
async findAllAdvanced(query: PaginationQueryDto): Promise<Pagination<Customer>> {
  const { page, limit, search, sortBy, sortOrder } = query;

  const queryBuilder = this.customerRepository.createQueryBuilder('customer');

  if (search) {
    queryBuilder.where(
      'customer.name ILIKE :search OR customer.email ILIKE :search OR customer.address ILIKE :search',
      { search: `%${search}%` },
    );
  }

  queryBuilder.orderBy(`customer.${sortBy}`, sortOrder);

  return paginate<Customer>(queryBuilder, {
    page,
    limit,
    route: '/customers', // optional: used to generate links
  });
}

/** Pagination with relations. */
async findAllWithOrders(options: IPaginationOptions): Promise<Pagination<Customer>> {
  const queryBuilder = this.customerRepository
    .createQueryBuilder('customer')
    .leftJoinAndSelect('customer.orders', 'order')
    .orderBy('customer.created_at', 'DESC');

  return paginate<Customer>(queryBuilder, options);
}

/** Pagination with a custom field selection. */
async findAllMinimal(options: IPaginationOptions): Promise<Pagination<Partial<Customer>>> {
  const queryBuilder = this.customerRepository
    .createQueryBuilder('customer')
    .select(['customer.id', 'customer.name', 'customer.email'])
    .orderBy('customer.created_at', 'DESC');

  return paginate(queryBuilder, options);
}
```

### Delivery drivers example

`src/delivery-drivers/dto/driver-pagination-query.dto.ts`

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

`src/delivery-drivers/delivery-drivers.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { paginate, Pagination } from 'nestjs-typeorm-paginate';
import { DeliveryDriver, VEHICLE_TYPE } from './entities/delivery-driver.entity';
import { DriverPaginationQueryDto } from './dto/driver-pagination-query.dto';

@Injectable()
export class DeliveryDriversService {
  constructor(
    @InjectRepository(DeliveryDriver)
    private driverRepository: Repository<DeliveryDriver>,
  ) {}

  /** Paginate drivers with filters. */
  async findAll(query: DriverPaginationQueryDto): Promise<Pagination<DeliveryDriver>> {
    const { page, limit, search, is_available, vehicle_type, sortBy, sortOrder } = query;

    const queryBuilder = this.driverRepository.createQueryBuilder('driver');

    if (search) {
      queryBuilder.where(
        'driver.name ILIKE :search OR driver.email ILIKE :search OR driver.phone ILIKE :search',
        { search: `%${search}%` },
      );
    }

    if (is_available !== undefined) {
      queryBuilder.andWhere('driver.is_available = :is_available', { is_available });
    }

    if (vehicle_type) {
      queryBuilder.andWhere('driver.vehicle_type = :vehicle_type', { vehicle_type });
    }

    queryBuilder.orderBy(`driver.${sortBy}`, sortOrder);

    return paginate<DeliveryDriver>(queryBuilder, {
      page,
      limit,
      route: '/delivery-drivers',
    });
  }

  /** Available drivers, paginated. */
  async findAvailable(
    page: number = 1,
    limit: number = 10,
    vehicle_type?: VEHICLE_TYPE,
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

### Controllers

`src/customers/customers.controller.ts`

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
    @Query('limit') limit: number = 10,
  ): Promise<Pagination<Partial<Customer>>> {
    return this.customersService.findAllMinimal({ page, limit });
  }
}
```

`src/delivery-drivers/delivery-drivers.controller.ts`

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
    @Query('vehicle_type') vehicle_type?: string,
  ): Promise<Pagination<DeliveryDriver>> {
    return this.driversService.findAvailable(page, limit, vehicle_type as any);
  }
}
```

> Individual `@Query('page')` values arrive as strings. Prefer the DTO approach (`@Query() query: Dto` with `@Type(() => Number)`), or add `ParseIntPipe`.

### API examples

| Request | Purpose |
|---------|---------|
| `GET /customers?page=1&limit=10` | Basic pagination |
| `GET /customers?page=1&limit=10&search=john` | With search |
| `GET /customers?page=1&limit=10&sortBy=name&sortOrder=ASC` | With sorting |
| `GET /delivery-drivers?page=2&limit=20&is_available=true&vehicle_type=bike&sortBy=created_at&sortOrder=DESC` | Complex query |
| `GET /delivery-drivers?search=john&is_available=true` | Search among available drivers |

### Custom pagination options

```ts
import { IPaginationOptions } from 'nestjs-typeorm-paginate';

const customOptions: IPaginationOptions = {
  page: 1,
  limit: 10,
  route: 'http://example.com/customers', // for link generation
  cacheQueries: true,                    // enable query caching
};

await paginate<Customer>(repository, customOptions);
```

### Manual vs package

```ts
// Manual: more code, manual calculations
const skip = (page - 1) * limit;
const [data, total] = await repository.findAndCount({ skip, take: limit });
const totalPages = Math.ceil(total / limit);
return { data, meta: { total, page, limit, totalPages } };

// Package: automatic
return paginate<Customer>(repository, { page, limit });
```

### Custom response serialization

`src/common/serializers/pagination.serializer.ts`

```ts
import { Pagination } from 'nestjs-typeorm-paginate';

export class PaginationSerializer {
  static serialize<T>(pagination: Pagination<T>, transformer?: (item: T) => any) {
    return {
      success: true,
      data: transformer ? pagination.items.map(transformer) : pagination.items,
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

Usage:

```ts
@Get()
async findAll(@Query() query: PaginationQueryDto) {
  const pagination = await this.customersService.findAllAdvanced(query);

  return PaginationSerializer.serialize(pagination, (customer) => ({
    id: customer.id,
    name: customer.name,
    email: customer.email,
    // password and other sensitive fields are left out
  }));
}
```

### Performance tips

| Tip | Example |
|-----|---------|
| Select only needed fields | `queryBuilder.select(['driver.id', 'driver.name', 'driver.is_available'])` |
| Add indexes on filtered/sorted columns | `@Entity('customers') @Index(['email']) @Index(['created_at'])` |
| Cap the page size | `@Max(100) limit?: number = 10;` |
| Cache queries | `paginate(queryBuilder, { page, limit, cacheQueries: true })` |

---

## QueryBuilder aliases vs table names

The string passed to `createQueryBuilder()` is an **alias**, not the table name.

```ts
@Entity('customers')   // table name in the database
export class Customer {}

const queryBuilder = this.customerRepository.createQueryBuilder('customer');
//                                                               ^^^^^^^^ alias only

queryBuilder.where('customer.name = :name', { name: 'John' }); // use the alias in the query
```

| Component | Purpose | Example |
|-----------|---------|---------|
| `@Entity('customers')` | Table name | Table `customers` |
| `.createQueryBuilder('customer')` | Alias for the query | Alias `customer` |
| `customer.name` | Column reference through the alias | Column `name` |

The alias can be anything. The generated SQL still uses the real table name.

```ts
this.customerRepository.createQueryBuilder('customer'); // common
this.customerRepository.createQueryBuilder('c');        // short
this.customerRepository.createQueryBuilder('user');     // works, but confusing

// With joins: each joined entity gets its own alias
const queryBuilder = this.customerRepository
  .createQueryBuilder('customer')                    // main entity alias
  .leftJoinAndSelect('customer.orders', 'order')     // related entity alias
  .where('customer.name = :name', { name: 'John' })
  .andWhere('order.total > :amount', { amount: 100 });

// SELECT ... FROM customers customer
// LEFT JOIN orders order ON order.customer_id = customer.id
// WHERE customer.name = 'John' AND order.total > 100
```

**Alias conventions**

1. Use descriptive aliases (`customer`, not `c`).
2. Be consistent across the codebase.
3. Match the entity name in lowercase: `Customer` to `customer`, `DeliveryDriver` to `driver` (or `deliveryDriver`), `Order` to `order`.
4. Avoid misleading aliases.

---

## Pagination patterns

| Pattern | Pros | Cons |
|---------|------|------|
| **Offset-based** | Easy, jump to any page, simple UI | Slow with large offsets; inconsistent if rows change mid-browse |
| **Cursor-based** | Stable performance, no skipped/duplicated rows, good for infinite scroll | Cannot jump to a page; more complex |
| **Page-based with QueryBuilder** | Most flexible (search, filters, sort) | More code |

### 1. Offset-based (most common)

```ts
async findAllWithPagination(page: number = 1, limit: number = 10) {
  const skip = (page - 1) * limit;

  const [data, total] = await this.repository.findAndCount({
    skip,
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

### 2. Cursor-based (infinite scroll)

```ts
async findAllWithCursor(cursor?: string, limit: number = 10) {
  const queryBuilder = this.repository.createQueryBuilder('entity');

  // If a cursor is given, fetch records after it
  if (cursor) {
    queryBuilder.where('entity.id > :cursor', { cursor });
  }

  queryBuilder
    .orderBy('entity.id', 'ASC')
    .take(limit + 1); // one extra row tells us whether a next page exists

  const data = await queryBuilder.getMany();
  const hasNextPage = data.length > limit;

  if (hasNextPage) {
    data.pop(); // remove the extra row
  }

  const nextCursor = hasNextPage ? data[data.length - 1].id : null;

  return { data, meta: { nextCursor, hasNextPage, limit } };
}
```

> The cursor column must be **ordered**. Random UUIDs (`@PrimaryGeneratedColumn('uuid')`) do not sort by creation time, so `id > cursor` skips and repeats rows. Use an auto-increment id or a `created_at` column (ideally combined with `id` as a tie-breaker).

### 3. Page-based with QueryBuilder (most flexible)

```ts
async findAllAdvanced(queryDto: QueryDto) {
  const { page = 1, limit = 10, sortBy = 'created_at', sortOrder = 'DESC', search } = queryDto;

  const skip = (page - 1) * limit;

  const queryBuilder = this.repository.createQueryBuilder('entity');

  // Dynamic search
  if (search) {
    queryBuilder.where(
      'entity.name ILIKE :search OR entity.email ILIKE :search',
      { search: `%${search}%` },
    );
  }

  // Dynamic sorting (whitelisted)
  const allowedSortFields = ['name', 'email', 'created_at'];
  if (allowedSortFields.includes(sortBy)) {
    queryBuilder.orderBy(`entity.${sortBy}`, sortOrder);
  }

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

## Sorting patterns

### 1. Simple sorting

```ts
async findAll() {
  return await this.repository.find({ order: { created_at: 'DESC' } });
}
```

### 2. Multiple fields

```ts
async findAll() {
  return await this.repository.find({
    order: {
      is_available: 'DESC', // available first
      created_at: 'ASC',    // then oldest first
    },
  });
}
```

### 3. Dynamic sorting (query parameter)

Always whitelist sortable fields.

```ts
async findAll(sortBy: string = 'created_at', sortOrder: 'ASC' | 'DESC' = 'DESC') {
  const allowedSortFields = ['name', 'email', 'created_at', 'is_available'];

  if (!allowedSortFields.includes(sortBy)) {
    sortBy = 'created_at'; // default fallback
  }

  return await this.repository.find({ order: { [sortBy]: sortOrder } });
}
```

### 4. Sorting with relations

```ts
async findAllWithRelations() {
  return await this.repository
    .createQueryBuilder('customer')
    .leftJoinAndSelect('customer.orders', 'order')
    .orderBy('order.total', 'DESC')      // by order total
    .addOrderBy('customer.name', 'ASC')  // then by customer name
    .getMany();
}
```

### 5. Case-insensitive sorting

```ts
async findAllCaseInsensitive() {
  return await this.repository
    .createQueryBuilder('entity')
    .orderBy('LOWER(entity.name)', 'ASC')
    .getMany();
}
```

---

## Advanced query patterns

### Search across multiple fields

```ts
async search(searchTerm: string) {
  return await this.repository
    .createQueryBuilder('customer')
    .where(
      'customer.name ILIKE :term OR customer.email ILIKE :term OR customer.address ILIKE :term',
      { term: `%${searchTerm}%` },
    )
    .getMany();
}
```

### Date range filtering

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

### Conditional filters

```ts
async findWithFilters(filters: any) {
  const queryBuilder = this.repository.createQueryBuilder('driver');

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

### Aggregation

```ts
async getAggregatedData() {
  return await this.repository
    .createQueryBuilder('driver')
    .select('driver.vehicle_type', 'vehicle_type')
    .addSelect('COUNT(*)', 'total')
    .addSelect('COUNT(CASE WHEN driver.is_available = true THEN 1 END)', 'available')
    .groupBy('driver.vehicle_type')
    .getRawMany();
}
```

---

## Service best practices

### 1. Error handling

```ts
async findOne(id: string): Promise<Customer> {
  try {
    const customer = await this.customerRepository.findOne({ where: { id } });

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

For domain-specific errors with a consistent response format, see `custom-exception.md`.

### 2. Transactions

Use a `QueryRunner` for multi-step operations that must succeed or fail together.

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
      const customer = queryRunner.manager.create(Customer, customerData);
      await queryRunner.manager.save(customer);

      const order = queryRunner.manager.create(Order, {
        ...orderData,
        customer,
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

| Call | Description |
|------|-------------|
| `createQueryRunner()` | Gets a dedicated connection. |
| `startTransaction()` | Begins the transaction. |
| `commitTransaction()` / `rollbackTransaction()` | Persist or undo everything. |
| `release()` | Always release the connection (in `finally`). |

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

    // Not cached: read from the database
    const customer = await this.customerRepository.findOne({ where: { id } });

    if (!customer) {
      throw new NotFoundException(`Customer with ID ${id} not found`);
    }

    // Store in cache (TTL)
    await this.cacheManager.set(cacheKey, customer, 300);

    return customer;
  }

  async update(id: string, updateDto: UpdateCustomerDto): Promise<Customer> {
    const customer = await this.findOne(id);
    Object.assign(customer, updateDto);
    const updated = await this.customerRepository.save(customer);

    // Invalidate the cache
    await this.cacheManager.del(`customer:${id}`);

    return updated;
  }
}
```

> The TTL unit depends on the `cache-manager` version: seconds in v4 and below, **milliseconds** in v5+. `300` above means 5 minutes in seconds; for v5+ use `300_000`.

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
        this.customerRepository.create(createCustomerDto),
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

### 5. Validation inside services

```ts
async update(id: string, updateDto: UpdateCustomerDto): Promise<Customer> {
  // Validate ID format
  if (!this.isValidUUID(id)) {
    throw new BadRequestException('Invalid ID format');
  }

  const customer = await this.findOne(id);

  // Email uniqueness if being updated
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

> A `ParseUUIDPipe` on the controller parameter does the same check before the service is called.

---

## Service testing

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
      expect(mockRepository.findOne).toHaveBeenCalledWith({ where: { id: '123' } });
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

| Piece | Purpose |
|-------|---------|
| `Test.createTestingModule()` | Builds an isolated module for the test. |
| `getRepositoryToken(Customer)` | The DI token `@InjectRepository(Customer)` resolves to. |
| `useValue: mockRepository` | Replaces the real repository with Jest mocks. |
| `mockResolvedValue()` / `mockReturnValue()` | Control what each mocked method returns. |

> `create()` first looks up the email with `findOne`, so in the `create` test also set `mockRepository.findOne.mockResolvedValue(null)`; otherwise the leftover mock value from an earlier test can make it throw `ConflictException`.

---

## Key takeaways

1. **Services hold business logic.** Keep controllers thin.
2. **Use DTOs** to validate all input.
3. **Throw proper HTTP exceptions** (`NotFoundException`, `ConflictException`).
4. **Paginate every list endpoint.**
5. **Whitelist sort fields**; never pass raw user input into `orderBy`.
6. **Use QueryBuilder** for complex filters.
7. **Use transactions** for multi-step operations.
8. **Cache** frequently read data and invalidate on write.
9. **Log** important operations.
10. **Unit test** every service method.

## Common patterns summary

| Pattern | Use case | Example |
|---------|----------|---------|
| Offset pagination | Standard list views | `skip: (page - 1) * limit` |
| Cursor pagination | Infinite scroll | `where: { id: MoreThan(cursor) }` |
| Dynamic sorting | Flexible ordering | `order: { [sortBy]: sortOrder }` |
| Search / filter | Find records | `where: { name: ILike('%term%') }` |
| Bulk operations | Update many records | `update().where('id IN (:...ids)')` |
| Aggregation | Statistics | `groupBy().count()` |
| Transactions | Atomic operations | `queryRunner.startTransaction()` |
| Caching | Performance | `cacheManager.get/set()` |

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| `findOne({ where: { id } })` with `id` undefined | TypeORM drops the condition and returns the **first row** | Validate the ID (`ParseUUIDPipe`) before querying |
| Passing user input straight into `orderBy` | SQL injection through the sort field | Whitelist sort fields (`@IsIn([...])`) |
| Returning the entity with its password hash | Leaks the hash in the response | Return a response DTO or use `@Exclude()` with `ClassSerializerInterceptor` |
| Forgetting `await` on `save()` | The write may not finish before the response | Always `await` repository calls |
| Not releasing a `QueryRunner` | Leaks database connections | Release it in `finally` |
| Loading relations inside a loop | N+1 queries | Use `leftJoinAndSelect` or `relations` in one query |
| Cursor pagination on a random UUID column | Rows are skipped or repeated | Order by an auto-increment id or `created_at` |
