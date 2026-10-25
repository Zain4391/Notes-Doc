# RabbitMQ: Configuration and Usage

Event-driven communication between modules or services with RabbitMQ: a topic exchange, one dedicated queue per listening service, a shared publisher, typed event DTOs and `@EventPattern` handlers.

**Contents**

1. [Architecture](#architecture)
2. [Packages](#packages)
3. [Environment variables](#environment-variables)
4. [Config file](#config-file)
5. [Publisher service](#publisher-service)
6. [RabbitMQ module](#rabbitmq-module)
7. [Wiring main.ts](#wiring-maints)
8. [Event DTOs](#event-dtos)
9. [Emitting events](#emitting-events)
10. [Listening to events](#listening-to-events)
11. [Common mistakes](#common-mistakes)

Related: [Modules](nest-module.md) · [Controllers](controller.md)

---

## Architecture

The pattern used here:

1. A **topic exchange** routes messages by **routing key** (the event name, e.g. `order.placed`).
2. Services publish events to the exchange through one shared publisher.
3. The exchange routes each event to every queue whose bindings match its routing key.
4. Each listening service owns a **dedicated queue**.

```text
Topic Exchange: "food-delivery-exchange"
├── orders-service-queue      (binds to: order.confirmed, driver.assigned, order.delivered)
├── restaurants-service-queue (binds to: order.placed)
└── delivery-service-queue    (binds to: order.ready, order.picked.up)
```

| Term | Meaning |
|------|---------|
| **Exchange** | Receives published messages and routes them to queues. |
| **Topic exchange** | Routes by pattern-matching the routing key (`order.*` matches `order.placed`). |
| **Routing key** | The event name attached to a message. |
| **Queue** | Buffer that stores messages for one consumer. Bound to the exchange with routing keys. |
| **Binding** | The rule linking a queue to an exchange for given routing keys. |

---

## Packages

```bash
npm install --save @nestjs/microservices amqplib amqp-connection-manager
```

| Package | Purpose |
|---------|---------|
| `@nestjs/microservices` | NestJS microservice transports, including RMQ. |
| `amqplib` | AMQP client for RabbitMQ. |
| `amqp-connection-manager` | Connection handling with automatic reconnection. |

---

## Environment variables

```bash
RABBITMQ_URL=amqp://localhost:5672
RABBITMQ_EXCHANGE=food-delivery-exchange
RABBITMQ_EXCHANGE_TYPE=topic

# Queue names
RABBITMQ_ORDERS_QUEUE=orders-service-queue
RABBITMQ_RESTAURANTS_QUEUE=restaurants-service-queue
RABBITMQ_DELIVERY_QUEUE=delivery-service-queue
```

| Variable | Description |
|----------|-------------|
| `RABBITMQ_URL` | Connection URL of the broker. |
| `RABBITMQ_EXCHANGE` | Name of the exchange events are published to. |
| `RABBITMQ_EXCHANGE_TYPE` | Exchange type (`topic` for this pattern). |
| `RABBITMQ_*_QUEUE` | One queue name per listening service. |

---

## Config file

`src/rabbitmq/rabbitmq.config.ts`

Builds the microservice options for a listener, and defines which events each service listens to.

```ts
import { MicroserviceOptions, RmqOptions, Transport } from '@nestjs/microservices';

interface QueueConfig {
  queue: string;
  routingKeys: string[];
}

export function getRabbitMQConfig(config: QueueConfig): MicroserviceOptions {
  return {
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RABBITMQ_URL as string],
      queue: config.queue,
      noAck: false, // manual acknowledgements
      queueOptions: {
        durable: true,
        arguments: {
          'x-message-ttl': 86400000, // 24 hour message TTL
        },
      },

      exchange: process.env.RABBITMQ_EXCHANGE as string,
      exchangeType: 'topic',
      routingKey: config.routingKeys,
    },
  } as unknown as RmqOptions;
}

// Config for each listener. routingKeys = the events it receives.
export const QUEUE_CONFIGS = {
  ORDERS: {
    queue: process.env.RABBITMQ_ORDERS_QUEUE || 'order-service-queue',
    routingKeys: ['order.confirmed', 'driver.assigned', 'order.delivered'],
  },
  RESTAURANTS: {
    queue: process.env.RABBITMQ_RESTAURANTS_QUEUE || 'restaurants-service-queue',
    routingKeys: ['order.placed'],
  },
  DELIVERY: {
    queue: process.env.RABBITMQ_DELIVERY_QUEUE || 'delivery-service-queue',
    routingKeys: ['order.ready', 'order.picked.up'],
  },
};
```

### Options

| Option | Description |
|--------|-------------|
| `transport` | `Transport.RMQ` selects the RabbitMQ transport. |
| `urls` | Broker URL(s). |
| `queue` | Queue this listener consumes from. |
| `noAck` | `false` means messages must be acknowledged manually; unacknowledged messages are redelivered. |
| `queueOptions.durable` | Queue survives broker restarts. |
| `queueOptions.arguments['x-message-ttl']` | Messages expire after this many ms (`86400000` = 24 h). |
| `exchange` / `exchangeType` | Exchange to bind to and its type. |
| `routingKey` | List of routing keys (events) the queue is bound to. |

### Listener map (this project)

| Config | Listens to |
|--------|-----------|
| `ORDERS` | `order.confirmed`, `driver.assigned`, `order.delivered` |
| `RESTAURANTS` | `order.placed` |
| `DELIVERY` | `order.ready`, `order.picked.up` |

---

## Publisher service

`src/rabbitmq/rabbitmq.service.ts`

Wraps a `ClientProxy` so any module can emit an event with one call.

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

| Member | Description |
|--------|-------------|
| `ClientProxyFactory.create(options)` | Creates the RabbitMQ client. |
| `getOrThrow()` | Reads a config value and throws at startup if it is missing. |
| `onModuleInit()` | Connects when the module initializes. |
| `onModuleDestroy()` | Closes the connection on shutdown. |
| `emitEvent(routingKey, data)` | Publishes an event (fire-and-forget) to the exchange using `routingKey`. |

---

## RabbitMQ module

`src/rabbitmq/rabbitmq.module.ts`

```ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { RabbitMQService } from './rabbitmq.service';

@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'RABBITMQ_SERVICE',
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => ({
          transport: Transport.RMQ,
          options: {
            urls: [configService.getOrThrow<string>('RABBITMQ_URL')],
            queue: configService.getOrThrow<string>('RABBITMQ_QUEUE'),
            queueOptions: {
              durable: true,
            },
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
  providers: [RabbitMQService],
  controllers: [],
  exports: [RabbitMQService],
})
export class RabbitMQModule {}
```

- `ClientsModule.registerAsync` registers a client under the token `RABBITMQ_SERVICE`, built from `ConfigService`.
- `exports: [RabbitMQService]` lets any module that imports `RabbitMQModule` inject the publisher.
- Common mistakes: `transport` takes the `Transport.RMQ` enum value; the variable is `RABBITMQ_URL` (not `RABBIT_URL`).

---

## Wiring main.ts

Connects one microservice listener per service, then starts them alongside the HTTP server.

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
  console.log('RabbitMQ microservices started');

  // Start HTTP server
  await app.listen(process.env.PORT ?? 3000);
  console.log(`HTTP Server running on port ${process.env.PORT ?? 3000}`);
}
bootstrap();
```

| Call | Description |
|------|-------------|
| `app.connectMicroservice(options)` | Attaches a microservice listener to the hybrid application. |
| `app.startAllMicroservices()` | Starts every attached listener. Call before `listen()`. |
| `app.listen(port)` | Starts the HTTP server. |

---

## Event DTOs

Events are typed as DTOs. The payload can technically be any type, but DTOs keep events validated and consistent.

### Recommended folder layout

```text
├── events/
│   ├── delivery/     # events emitted by delivery
│   ├── restaurant/   # events emitted by restaurants
│   └── order/        # events emitted by orders
└── common/events/
    └── base.event.ts # base event DTO extended by all others
```

Placement is the developer's choice; this layout is a recommendation.

### Base event

Every event gets an `eventId` and `timestamp` automatically.

```ts
import { IsDateString, IsString, IsUUID } from 'class-validator';

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

### Order events

```ts
import { Type } from 'class-transformer';
import { IsArray, IsNumber, IsOptional, IsString, IsUUID, ValidateNested } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';
import { OrderItemDTO } from '../../orders/dto/create-order.dto';

export class OrderPlacementEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() customerId: string;
  @IsUUID() restaurantId: string;
  @IsString() deliveryAddress: string;

  @IsString()
  @IsOptional()
  specialInstruction?: string;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => OrderItemDTO)
  items: OrderItemDTO[];

  @IsNumber() totalAmount: number;

  constructor(partial: Partial<OrderPlacementEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.placed';
  }
}
```

### Restaurant events

```ts
import { IsDateString, IsUUID } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class OrderConfirmedEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() restaurantId: string;
  @IsDateString() estimatedDeliveryTime: string;

  constructor(partial: Partial<OrderConfirmedEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.confirmed';
  }
}
```

```ts
import { IsString, IsUUID } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class OrderReadyEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() restaurantId: string;
  @IsString() deliveryAddress: string;

  constructor(partial: Partial<OrderReadyEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.ready';
  }
}
```

### Delivery events

```ts
import { IsString, IsUUID, IsDateString } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class DriverAssignedEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() driverId: string;
  @IsString() driverName: string;
  @IsString() driverPhone: string;
  @IsDateString() estimatedDeliveryTime: string;
  @IsDateString() assignedAt: string;

  constructor(partial: Partial<DriverAssignedEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'driver.assigned';
    this.assignedAt = new Date().toISOString();
  }
}
```

```ts
import { IsDateString, IsUUID } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class OrderDeliveredEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() driverId: string;
  @IsDateString() deliveredAt: string;

  constructor(partial: Partial<OrderDeliveredEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.delivered';
    this.deliveredAt = new Date().toISOString();
  }
}
```

```ts
import { IsDateString, IsUUID } from 'class-validator';
import { BaseEventDTO } from 'src/common/events/base-event.dto';

export class OrderPickedUpEvent extends BaseEventDTO {
  @IsUUID() orderId: string;
  @IsUUID() driverId: string;
  @IsUUID() customerId: string;
  @IsDateString() pickedUpAt: string;

  constructor(partial: Partial<OrderPickedUpEvent>) {
    super();
    Object.assign(this, partial);
    this.eventType = 'order.picked.up';
    this.pickedUpAt = new Date().toISOString();
  }
}
```

### Event summary

| Event DTO | Routing key | Emitted by | Consumed by |
|-----------|-------------|-----------|-------------|
| `OrderPlacementEvent` | `order.placed` | Orders | Restaurants |
| `OrderConfirmedEvent` | `order.confirmed` | Restaurants | Orders |
| `OrderReadyEvent` | `order.ready` | Restaurants | Delivery |
| `OrderPickedUpEvent` | `order.picked.up` | Delivery | Delivery queue (per config) |
| `DriverAssignedEvent` | `driver.assigned` | Delivery | Orders |
| `OrderDeliveredEvent` | `order.delivered` | Delivery | Orders |

> Each DTO constructor sets its own `eventType`, so the routing key and DTO always match.

---

## Emitting events

Emit from the service method after the business logic has completed.

```ts
this.rabbitMQService.emitEvent('<EVENT_TYPE>', event);
```

### Example

```ts
const event = new OrderPlacementEvent({
  orderId: completeOrder.id,
  customerId: completeOrder.customer_id,
  restaurantId: completeOrder.restaurant_id,
  items: completeOrder.orderItems,
  totalAmount: completeOrder.total_amount,
  deliveryAddress: completeOrder.delivery_address,
});

this.rabbitMQService.emitEvent('order.placed', event);
```

Inject `RabbitMQService` into the service constructor, and import `RabbitMQModule` in the module that uses it.

---

## Listening to events

> **Note:** a controller *listens* for a specific event, so the service behind that controller must have the matching handler method(s).

Register a handler in the controller with `@EventPattern('<EVENT_TYPE>')`.

```ts
@EventPattern('order.confirmed')
async handleOrderConfirmed(@Payload() data: OrderConfirmedEvent) {
  this.logger.log(`Received order.confirmed event:`, data);
  await this.orderService.handleOrderConfirmed(data);
}
```

| Decorator | Description |
|-----------|-------------|
| `@EventPattern(routingKey)` | Handles messages published with that routing key. Fire-and-forget: no response is sent back. |
| `@Payload()` | Extracts the message body as the handler argument. |

The routing key in `@EventPattern` must also appear in that service's `routingKeys` in [`QUEUE_CONFIGS`](#config-file), otherwise the queue is never bound to it.

---

## Common mistakes

| Mistake | Why it matters | Fix |
|---|---|---|
| The RabbitMQ module reads `RABBITMQ_QUEUE`, which the environment list does not define | `getOrThrow` fails at startup | Add the variable, or drop the `queue` option from the publisher client |
| Handler registered with `@EventPattern('x')` but `'x'` is missing from the service's `routingKeys` | The queue is never bound to that key, so the handler never fires | Add the key to the matching entry in `QUEUE_CONFIGS` |
| `noAck: false` but messages are never acknowledged | Unacknowledged messages are redelivered | Acknowledge once the handler has finished |
| Event DTO `eventType` differs from the routing key used in `emitEvent()` | Consumers and logs disagree about the event | Let the DTO constructor set `eventType` and pass the same string as the routing key |
| Controller listens for an event, but its service has no handler method | The handler calls a method that does not exist | Add the method to the service behind that controller |

> This is **one** way of implementing a message broker for a **modular monolith**. It does not have to be followed exactly; implementations change as requirements change.
