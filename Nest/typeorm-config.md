# TypeORM + Supabase Configuration

Connecting a NestJS application to a Supabase-hosted PostgreSQL database with TypeORM, defining entities, and managing schema changes with migrations.

**Contents**

1. [Overview](#overview)
2. [Packages](#packages)
3. [Config files](#config-files)
4. [Data source (for the CLI)](#data-source-for-the-cli)
5. [Entities](#entities)
6. [Migrations](#migrations)
7. [Quick reference](#quick-reference)
8. [Common mistakes](#common-mistakes)

Related: [Services](service.md) (repositories, queries, pagination) · [Modules](nest-module.md) (`TypeOrmModule.forFeature`)

---

## Overview

Two separate configurations are needed, because the app and the TypeORM CLI start up differently:

| Concern | File | Used by | Connection variable |
|---------|------|---------|---------------------|
| Runtime connection | `src/config/database.config.ts` | The NestJS app (via `@nestjs/config`) | `DATABASE_URL` |
| Supabase client | `src/config/` (same folder) | App code that calls Supabase APIs | `SUPABASE_URL`, `SUPABASE_ANON_KEY` |
| CLI connection | `src/data-source.ts` | `typeorm` CLI (migrations) | `MIGRATION_URL` |

> The CLI runs outside the Nest application, so it cannot read Nest's `ConfigService`. It needs its own `DataSource` and loads `.env` with `dotenv`.

---

## Packages

Node.js and a package manager (npm, yarn or pnpm) must be installed.

```bash
npm install --save @nestjs/typeorm typeorm pg
npm install --save @supabase/supabase-js
```

| Package | Purpose |
|---------|---------|
| `@nestjs/typeorm` | NestJS integration for TypeORM (`TypeOrmModule`). |
| `typeorm` | The ORM itself. |
| `pg` | PostgreSQL driver (Supabase is PostgreSQL). |
| `@supabase/supabase-js` | Supabase client for APIs other than the raw DB connection (auth, storage, ...). |

---

## Config files

Create these inside the `src/config/` folder.

### Database config

Registered under the `database` namespace with `registerAs`, so values are read later with `configService.get('database')`.

```ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  type: 'postgres' as const,
  url: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: process.env.NODE_ENV === 'development',
  autoLoadEntities: true,
  logging: process.env.NODE_ENV === 'development',
}));
```

| Option | Value | Description |
|--------|-------|-------------|
| `type` | `'postgres'` | Database driver. `as const` keeps the literal type for TypeORM's typings. |
| `url` | `DATABASE_URL` | Full connection string from the Supabase dashboard. |
| `ssl` | `{ rejectUnauthorized: false }` in production | Supabase requires SSL for remote connections. Disabled locally. |
| `entities` | glob of `*.entity.ts/js` | Where TypeORM looks for entity classes. |
| `synchronize` | `true` in development only | Auto-creates/alters tables from entities. **Never enable in production**, it can drop data. |
| `autoLoadEntities` | `true` | Entities registered through `TypeOrmModule.forFeature()` are loaded automatically. |
| `logging` | `true` in development only | Prints every SQL query to the console. |

### Supabase client

```ts
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.SUPABASE_URL || '';
const supabaseKey = process.env.SUPABASE_ANON_KEY || '';

export const supabaseClient = createClient(supabaseUrl, supabaseKey);
```

---

## Data source (for the CLI)

Create `src/data-source.ts`. The migration scripts in [Migrations](#migrations) point at this path, so keep the two consistent.

```ts
import { DataSource } from 'typeorm';
import { config } from 'dotenv';

config();

export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.MIGRATION_URL,
  ssl: process.env.DB_SSL === 'false' ? false : { rejectUnauthorized: false },

  entities: ['src/**/*.entity{.ts,.js}'],
  migrations: ['src/migrations/*{.ts,.js}'],

  synchronize: false,
  logging: true,
});
```

| Option | Description |
|--------|-------------|
| `config()` | Loads `.env` into `process.env` (the CLI does not do this on its own). |
| `url` | Use `MIGRATION_URL`. With Supabase this is typically the **direct** connection (not the pooled one), since migrations need a session-level connection. |
| `ssl` | SSL is on unless `DB_SSL=false`. |
| `entities` | Entities the CLI compares against the database when generating migrations. |
| `migrations` | Where migration files are stored and read from. |
| `synchronize` | Must stay `false`. Schema changes go through migrations only. |

---

## Entities

Entities live inside their module folder. Example: `src/users/entity/customer-entity.ts`

```ts
import { Order } from '../../orders/entities/order.entity';
import {
  Entity, Column, PrimaryGeneratedColumn,
  CreateDateColumn, UpdateDateColumn, OneToMany,
} from 'typeorm';
import { ROLES } from '../../common/enums/roles.enum';

@Entity('customers')
export class Customer {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 100 })
  name: string;

  @Column({ unique: true, length: 100 })
  email: string;

  @Column()
  password: string;

  @Column({ unique: true, length: 255, nullable: true })
  profile_image_url: string;

  @Column({ length: 255 })
  address: string;

  // Relation: one Customer has many Orders
  @OneToMany(() => Order, (order) => order.customer)
  orders: Order[];

  @Column({ type: 'enum', enum: ROLES, default: ROLES.CUSTOMER })
  role: ROLES;

  @CreateDateColumn()
  created_at: Date;

  @UpdateDateColumn()
  updated_at: Date;
}
```

### Decorators used

| Decorator | Description |
|-----------|-------------|
| `@Entity('customers')` | Marks the class as a table. The argument is the table name. |
| `@PrimaryGeneratedColumn('uuid')` | Auto-generated primary key. `'uuid'` generates a UUID instead of an incrementing integer. |
| `@Column(options)` | A regular column. Common options: `length`, `unique`, `nullable`, `default`, `type`. |
| `@Column({ type: 'enum', enum })` | Enum column backed by a TypeScript enum. |
| `@OneToMany(() => Target, (t) => t.inverse)` | The "one" side of a one-to-many relation. Holds no column itself. |
| `@CreateDateColumn()` | Set automatically on insert. |
| `@UpdateDateColumn()` | Updated automatically on every save. |

> **Key point:** the `Order` entity is defined in its own file, `src/order/entity/order-entity.ts`, and must declare the matching `@ManyToOne` side pointing back to `Customer`.

---

## Migrations

Migrations generate and apply the table definitions.

### 4.1 Add scripts to `package.json`

```json
{
  "scripts": {
    "typeorm": "typeorm-ts-node-commonjs",
    "migration:generate": "npm run typeorm -- migration:generate -d src/data-source.ts",
    "migration:run": "npm run typeorm -- migration:run -d src/data-source.ts",
    "migration:revert": "npm run typeorm -- migration:revert -d src/data-source.ts",
    "migration:show": "npm run typeorm -- migration:show -d src/data-source.ts"
  }
}
```

| Script | Description |
|--------|-------------|
| `typeorm` | Runs the TypeORM CLI with TypeScript support. |
| `migration:generate` | Compares entities with the database and writes a migration file for the difference. |
| `migration:run` | Applies all pending migrations. |
| `migration:revert` | Reverts the **last** applied migration. |
| `migration:show` | Lists migrations and whether they have been applied. |

### 4.2 Run the commands

```bash
# Generate a migration (path = folder + migration name)
npm run migration:generate -- src/migrations/<MigrationName>

# Apply it to create/update the tables
npm run migration:run
```

> The extra `--` is needed so npm passes the path through to the TypeORM CLI.

### Typical workflow

1. Change or add an entity.
2. `migration:generate` creates the SQL for the change. Review the generated file.
3. `migration:run` applies it.
4. Made a mistake? `migration:revert` undoes the last one.

---

## Quick reference

| Task | Command / Location |
|------|--------------------|
| Install packages | `npm install --save @nestjs/typeorm typeorm pg @supabase/supabase-js` |
| App DB config | `src/config/` using `DATABASE_URL` |
| CLI DB config | `src/data-source.ts` using `MIGRATION_URL` |
| Entity files | `src/<module>/entity/*.entity.ts` |
| Create migration | `npm run migration:generate -- src/migrations/<Name>` |
| Apply migrations | `npm run migration:run` |
| Undo last migration | `npm run migration:revert` |
| Check status | `npm run migration:show` |

---

> This setup is for TypeORM + Supabase. For other hosting providers, check their own documentation (connection strings and SSL requirements differ).

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| `synchronize: true` outside development | TypeORM alters the schema on startup and can drop data | Keep it `false` in production and use migrations |
| Missing `dotenv` `config()` in `data-source.ts` | The CLI does not load `.env`, so `MIGRATION_URL` is undefined | Call `config()` at the top |
| Running `npm run migration:generate src/migrations/Name` without `--` | npm does not forward the path to the TypeORM CLI | `npm run migration:generate -- src/migrations/Name` |
| Entities glob not matching | The CLI generates an empty migration or finds no tables | Check `entities` in `data-source.ts` (`src/**/*.entity.ts`) against the real file names |
| Committing `.env` with `DATABASE_URL` | Leaks database credentials | Keep `.env` in `.gitignore` |
| Declaring `@OneToMany` without the matching `@ManyToOne` | The relation is incomplete and the join column is missing | Define both sides |
