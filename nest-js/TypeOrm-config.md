This is a complete guide on using TypeOrm with Nest JS and Supabase.

# Prerequisites
Make sure you have Node.js and npm/yarn/pnpm installed.

## Install dependencies

```bash
npm install --save @nestjs/typeorm typeorm pg
npm install --save @supabase/supabase-js
```

# Step 1: Make typeorm and supabase config files inside Config folder

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

// Supabase

import { createClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.SUPABASE_URL || '';
const supabaseKey = process.env.SUPABASE_ANON_KEY || '';

export const supabaseClient = createClient(supabaseUrl, supabaseKey);

```

# Step 2: Make data source file inside project root

```ts
import { DataSource } from "typeorm";
import { config } from "dotenv";

config();

export const AppDataSource = new DataSource({
    type: "postgres",
    url: process.env.MIGRATION_URL,
    ssl: process.env.DB_SSL === 'false' ? false : { rejectUnauthorized: false },

    entities: ['src/**/*.entity{.ts,.js}'],
    
    migrations: ['src/migrations/*{.ts,.js}'],
    
    synchronize: false,
    
    logging: true,
});

```

# Step 3: Make entity files inside module folders

For example **src/users/entity/customer-entity.ts** file

```ts

import { Order } from "../../orders/entities/order.entity";
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, OneToMany } from "typeorm";
import { ROLES } from "../../common/enums/roles.enum";

@Entity("customers")
export class Customer {

    @PrimaryGeneratedColumn('uuid')
    id: string;

    @Column({ length: 100 })
    name: string;

    @Column({ unique: true, length: 100})
    email: string;

    @Column()
    password: string;

    @Column({ unique: true, length: 255, nullable: true })
    profile_image_url: string;

    @Column({ length: 255 })
    address: string;

    // ADD Relation: One Customer has Many Orders
    @OneToMany(() => Order, (order) => order.customer)
    orders: Order[]

    @Column({
        type: 'enum',
        enum: ROLES,
        default: ROLES.CUSTOMER
    })
    role: ROLES;    

    @CreateDateColumn()
    created_at: Date;

    @UpdateDateColumn()
    updated_at: Date;
}

```
## **Key point**:
- The **Order** entity will be defined in it's own **src/order/entity/order-entity.ts** file

# Step 4: Running Migrations

Follow the steps to generate tables for your database

## Step 1: Add the followin scripts inside your **package.json** file:

```json
{
    "typeorm": "typeorm-ts-node-commonjs",
    "migration:generate": "npm run typeorm -- migration:generate -d src/data-source.ts",
    "migration:run": "npm run typeorm -- migration:run -d src/data-source.ts",
    "migration:revert": "npm run typeorm -- migration:revert -d src/data-source.ts",
    "migration:show": "npm run typeorm -- migration:show -d src/data-source.ts"
}
```
## Step 2: Run the commands:

```bash
# generates the migration
npm run migration:generate <PATH_TO_MIGRATIONS_FOLDER/<NAME_OF_MIGRATION> 

# runs the migration to create tables
npm run migration:run 
```
# Conclusion

This set up is for TypeOrm + Supabase. For other hosting providers, checkout their documentations.