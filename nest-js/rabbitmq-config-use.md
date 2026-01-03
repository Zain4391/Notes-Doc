# RabbitMQ Config and Use

This guide/note is about using rabbitmq with the following pattern:
1. Topic exchange and routing keys (events)
2. Each service sends a message to one shared consumer
3. The consumer then routes the message to the appropriate service
4. The listening service has a dedicated queue.

### Example:
```bash

Topic Exchange: "food-delivery-exchange"
├── orders-service-queue (binds to: order.*, driver.*)
├── restaurants-service-queue (binds to: order.placed)
└── delivery-service-queue (binds to: order.ready)

```
# Pre-req
First thing's first, the dependencies to be installed:

```bash

npm install --save @nestjs/microservices amqplib amqp-connection-manager

```
Now we can proceed.

# Step 1: Environment Set up

Make sure to have the following environment variables:

```bash
RABBITMQ_URL=amqp://localhost:5672
RABBITMQ_EXCHANGE=food-delivery-exchange
RABBITMQ_EXCHANGE_TYPE=topic

# Queue names
RABBITMQ_ORDERS_QUEUE=orders-service-queue
RABBITMQ_RESTAURANTS_QUEUE=restaurants-service-queue
RABBITMQ_DELIVERY_QUEUE=delivery-service-queue

```
# Step 2: Config file

Inside **src/rabbitmq/rabbitmq.config.ts**:

```ts
import { MicroserviceOptions, RmqOptions, Transport } from "@nestjs/microservices";

interface QueueConfig {
    queue: string;
    routingKeys: string[]
}

export function getRabbitMQConfig(config: QueueConfig): MicroserviceOptions {

    return {
        transport: Transport.RMQ,
        options: {
            urls: [process.env.RABBITMQ_URL as string],
            queue: config.queue,
            noAck: false, // Manual acks
            queueOptions: {
                durable: true,
                arguments: {
                    'x-message-ttl': 86400000 // 24 hr message TTL
                }
            },

            exchange: process.env.RABBITMQ_EXCHANGE as string,
            exchangeType: 'topic',
            routingKey: config.routingKeys
        }
    } as unknown as RmqOptions;
}


// Config for each listener
// routingKeys = Events
export const QUEUE_CONFIGS = {
    ORDERS: {
        queue: process.env.RABBITMQ_ORDERS_QUEUE || 'order-service-queue',
        routingKeys: [
            'order.confirmed',
            'driver.assigned',
            'order.delivered'
        ]
    },
    RESTAURANTS: {
        queue: process.env.RABBITMQ_RESTAURANTS_QUEUE || 'restaurants-service-queue',
        routingKeys: [
            'order.placed'
        ]
    },
    DELIVERY: {
        queue: process.env.RABBITMQ_DELIVERY_QUEUE || 'delivery-service-queue',
        routingKeys: [
            'order.ready',
            'order.picked.up'
        ]
    }
}

```

# Step 3: Make the RabbitMQ service file

Inside **src/rabbitmq/rabbitmq.service.ts**:

```ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { ClientProxy, ClientProxyFactory, Transport, RmqOptions } from '@nestjs/microservices';

@Injectable()
export class RabbitMQService implements OnModuleInit, OnModuleDestroy {
  private client: ClientProxy;

  constructor(private configService: ConfigService) {
    const options: RmqOptions = {
      transport: Transport.RMQ,
      options: {
        urls: [this.configService.getOrThrow<string>('RABBITMQ_URL')],
        exchange: this.configService.getOrThrow<string>('RABBITMQ_EXCHANGE'),
        exchangeType: 'topic',
        queueOptions: {
          durable: true,
        },
      },
    };

    this.client = ClientProxyFactory.create(options);
  }

  async onModuleInit() {
    await this.client.connect();
    console.log('RabbitMQ Publisher connected');
  }

  async onModuleDestroy() {
    await this.client.close();
  }

  emitEvent(routingKey: string, data: any) {
    this.client.emit(routingKey, data);
    console.log(`Event emitted [${routingKey}]:`, data);
  }
}

```

# Step 4: Update the RabbitMQ module

Inside **src/rabbitmq/rabbitmq.module.ts**:

```ts

import { Module } from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { ClientsModule, Transport } from "@nestjs/microservices";
import { RabbitMQService } from "./rabbitmq.service";

@Module({
    imports: [
        ClientsModule.registerAsync([
            {
                name: "RABBITMQ_SERVICE",
                imports: [ConfigModule],
                useFactory: (configService: ConfigService) => ({
                    transport: Transport.RMQ,  // lowercase 'transport', not 'Transport'
                    options: {
                        urls: [configService.getOrThrow<string>('RABBITMQ_URL')],  // Fixed typo: RABBIT_URL -> RABBITMQ_URL
                        queue: configService.getOrThrow<string>('RABBITMQ_QUEUE'),  // Use getOrThrow for consistency
                        queueOptions: {
                            durable: true
                        }
                    }
                }),
                inject: [ConfigService]
            }
        ])
    ],
    providers: [RabbitMQService],
    controllers: [],
    exports: [RabbitMQService]
})
export class RabbitMQModule {}

```
# Step 5: Update the main.ts file

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './common/filter/http-exception.filter';
import { ValidationPipe } from '@nestjs/common';
import { getRabbitMQConfig, QUEUE_CONFIGS } from './rabbitmq/rabbitmq.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Connect multiple microservice listeners (one per service)
  app.connectMicroservice(getRabbitMQConfig(QUEUE_CONFIGS.ORDERS));
  app.connectMicroservice(getRabbitMQConfig(QUEUE_CONFIGS.RESTAURANTS));
  app.connectMicroservice(getRabbitMQConfig(QUEUE_CONFIGS.DELIVERY));

  // Global exception filter registration
  app.useGlobalFilters(new GlobalExceptionFilter());

  // Validation for DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  // Start all microservice listeners
  await app.startAllMicroservices();
  console.log('🐰 RabbitMQ microservices started');

  // Start HTTP server
  await app.listen(process.env.PORT ?? 3000);
  console.log(`🚀 HTTP Server running on port ${process.env.PORT ?? 3000}`);
}
bootstrap();

```

# Usage Examples

Once the set up is done, the service methods can be used. Despite the service's data being of ***any*** type, we pass the event DTOs in the services

Placing the DTOs in folder is the developer's choice, but the following pattern is recommended:

```bash

├── events/
|   ├── delivery/ # delivery emitted event DTOs
│   ├── restaurant/ # Restaurant emitted event DTOs
│   └── order/ # Order emitted event DTOs

├── common/events/
|   ├── base.event.ts # Base event DTO extended by others

```

## DTOs examples

Base DTO:

```ts
import { IsDateString, IsString, IsUUID } from "class-validator";

export abstract class BaseEventDTO {

    @IsUUID()
    eventId: string;

    @IsDateString()
    timestamp: string;

    @IsString()
    eventType: string;

    constructor() {
        this.eventId = crypto.randomUUID();
        this.timestamp = new Date().toISOString();
    }
}

```

Inherited DTOs:

Order Events:

```ts
import { Type } from "class-transformer";
import { IsArray, IsNumber, IsOptional, IsString, IsUUID, ValidateNested } from "class-validator";
import { BaseEventDTO } from "src/common/events/base-event.dto";
import { OrderItemDTO } from "../../orders/dto/create-order.dto";

export class OrderPlacementEvent extends BaseEventDTO {

    @IsUUID()
    orderId: string;

    @IsUUID()
    customerId: string;

    @IsUUID()
    restaurantId: string;

    @IsString()
    deliveryAddress: string;

    @IsString()
    @IsOptional()
    specialInstruction?: string;

    @IsArray()
    @ValidateNested({ each: true })
    @Type(() => OrderItemDTO)
    items: OrderItemDTO[];

    @IsNumber()
    totalAmount: number;

    constructor(partial: Partial<OrderPlacementEvent>) {
        super();
        Object.assign(this, partial);
        this.eventType = 'order.placed';
    }
}

```

Restaurant Events:

```ts
import { IsDateString, IsUUID } from "class-validator";
import { BaseEventDTO } from "src/common/events/base-event.dto";

export class OrderConfirmedEvent extends BaseEventDTO {

    @IsUUID()
    orderId: string;

    @IsUUID()
    restaurantId: string;

    @IsDateString()
    estimatedDeliveryTime: string;

    constructor(partial: Partial<OrderConfirmedEvent>) {
        super();
        Object.assign(this, partial);
        this.eventType = 'order.confirmed';
    }
}

import { IsString, IsUUID } from "class-validator";
import { BaseEventDTO } from "src/common/events/base-event.dto";

export class OrderReadyEvent extends BaseEventDTO {

    @IsUUID()
    orderId: string;

    @IsUUID()
    restaurantId: string;

    @IsString()
    deliveryAddress: string;

    constructor(partial: Partial<OrderReadyEvent>) {
        super();
        Object.assign(this, partial);
        this.eventType = 'order.ready';
    }
}

```

Delivery Events:

```ts
import { IsString, IsUUID, IsDateString } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class DriverAssignedEvent extends BaseEventDTO {

  @IsUUID()
  orderId: string;

  @IsUUID()
  driverId: string;

  @IsString()
  driverName: string;

  @IsString()
  driverPhone: string;

  @IsDateString()
  estimatedDeliveryTime: string; 

  @IsDateString()
  assignedAt: string;

  constructor(partial: Partial<DriverAssignedEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'driver.assigned';
    this.assignedAt = new Date().toISOString();
  }
}

import { IsDateString, IsUUID } from "class-validator";
import { BaseEventDTO } from "src/common/events/base-event.dto";

export class OrderDeliveredEvent extends BaseEventDTO {

  @IsUUID()
  orderId: string;

  @IsUUID()
  driverId: string;

  @IsDateString()
  deliveredAt: string;

  constructor(partial: Partial<OrderDeliveredEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.delivered';
    this.deliveredAt = new Date().toISOString();
  }
}

import { IsDateString, IsUUID } from "class-validator";
import { BaseEventDTO } from "src/common/events/base-event.dto";

export class OrderPickedUpEvent extends BaseEventDTO {

  @IsUUID()
  orderId: string;

  @IsUUID()
  driverId: string;

  @IsUUID()
  customerId: string;

  @IsDateString()
  pickedUpAt: string;

  constructor(partial: Partial<OrderPickedUpEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.picked.up';
    this.pickedUpAt = new Date().toISOString();
  }
}

```
## Using in service

Use them in the service method where the event needs to be emitted after a logic has been completed:

```ts
this.rabbitmqService.emit("<EVENT_TYPE>", event)
```

### For example:

```ts

const event = new OrderPlacementEvent({
            orderId: completeOrder.id,
            customerId: completeOrder.customer_id,
            restaurantId: completeOrder.restaurant_id,
            items: completeOrder.orderItems,
            totalAmount: completeOrder.total_amount,
            deliveryAddress: completeOrder.delivery_address
});

this.rabbitMQService.emitEvent('order.placed', event);

```

## Register in Controller

**NOTE:** Controller *LISTENS* for a specific event, so the service for that specific controller needs to have the required handler method(s).

Register an event inside the controller by using the **@EventPattern("<EVENT_TYPE>")** decorator

### For example:

```ts

@EventPattern('order.confirmed')
async handleOrderConfirmed(@Payload() data: OrderConfirmedEvent) {
    this.logger.log(`Received order.confirmed event:`, data);
    await this.orderService.handleOrderConfirmed(data);
}

```

# Conclusion

This is ONE of the ways of implementing a message broker for a **Modular Monolith** application. It does not mean that one has to follow this. Implementations change as requirements change.