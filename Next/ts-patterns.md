# TypeScript Patterns

This project uses TypeScript strictly throughout. This file covers the key patterns: typed API responses, the `AppException` class, module augmentation, and lookup maps.

---

## API Response Types

The backend wraps every response in a consistent envelope. Typing this envelope once means every service call is typed correctly.

```ts
// types/api.types.ts

export interface ApiSuccessResponse<T = unknown> {
  statusCode: number;
  success: true;
  data: T;
  message: string;
}

export interface ApiErrorResponse {
  success: false;
  error: ApiError;
}

export type ApiError = {
  message: string;
  statusCode: number;
  timestamp: string;
};

export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}
```

Service calls use the generic type parameter to type what `data` contains:

```ts
apiClient.get<Customer>("/customer/profile"); // returns Promise<Customer>
apiClient.get<PaginatedResponse<Order>>("/orders"); // returns Promise<PaginatedResponse<Order>>
```

---

## The AppException Class

All API errors are normalised into a single typed class. This makes error handling predictable everywhere.

```ts
// types/api.types.ts
export class AppException extends Error {
  public readonly statusCode: number;
  public readonly timestamp: string;

  constructor(error: ApiError) {
    super(error.message);
    this.name = "AppException";
    this.statusCode = error.statusCode;
    this.timestamp = error.timestamp;
  }
}
```

The Axios response interceptor throws `AppException` for every error response. Hooks and components can check `instanceof AppException` to handle errors predictably:

```ts
} catch (error) {
  if (error instanceof AppException) {
    console.error(`[Auth Error] ${error.statusCode}: ${error.message}`)
    throw new Error(error.message)   // surface message to the UI
  }
  throw new Error('Something went wrong')
}
```

Two utility functions handle the common cases:

```ts
export function isAppException(error: unknown): error is AppException {
  return error instanceof AppException;
}

export function getErrorMessage(error: unknown): string {
  if (isAppException(error)) return error.message;
  if (error instanceof Error) return error.message;
  return "Something went wrong";
}
```

`getErrorMessage` is a safe fallback used in hooks to extract a displayable string from any thrown value.

---

## Module Augmentation — Extending NextAuth Types

NextAuth's built-in types do not include `role`, `userType`, or `accessToken`. Module augmentation extends the library's interfaces without modifying it.

```ts
// types/next-auth.d.ts
import { DefaultSession } from "next-auth";
import { ROLES, UserType } from "./auth.types";

declare module "next-auth" {
  interface Session {
    accessToken: string;
    user: {
      id: string;
      role: ROLES;
      userType: UserType;
    } & DefaultSession["user"]; // preserve name, email, image from default
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string;
    role: ROLES;
    userType: UserType;
    accessToken: string;
  }
}
```

After this, `session.user.role` and `session.accessToken` are valid TypeScript — no type assertions needed.

---

## Auth Types and Enums

```ts
// types/auth.types.ts
export enum ROLES {
  CUSTOMER = "CUSTOMER",
  DRIVER = "DRIVER",
  ADMIN = "ADMIN",
}

export enum VEHICLE_TYPE {
  CAR = "car",
  BIKE = "bike",
}

export type UserType = "customer" | "driver" | "admin";
```

`ROLES` and `VEHICLE_TYPE` are string enums — their values are sent to the backend as strings. `UserType` is a union type used for frontend-only routing and provider selection.

---

## Lookup Maps with Record

`Record<KeyType, ValueType>` creates a typed object where every key in `KeyType` must have a value. If a new `UserType` is added, TypeScript immediately flags any map that is missing the new key.

```ts
// types/map.ts
export const PROVIDER_MAP: Record<UserType, string> = {
  customer: "customer-login",
  driver: "driver-login",
  admin: "admin-login",
};

export const REDIRECT_MAP: Record<UserType, string> = {
  customer: "/customer/dashboard",
  driver: "/driver/dashboard",
  admin: "/admin/dashboard",
};

type RoleConfig = {
  label: string;
  register?: { href: string; label: string };
};

export const ROLE_CONFIG: Record<UserType, RoleConfig> = {
  customer: {
    label: "Customer",
    register: { href: "/register/customer", label: "Create an account" },
  },
  driver: {
    label: "Driver",
    register: { href: "/register/driver", label: "Join as a driver" },
  },
  admin: { label: "Admin" }, // no register link for admin
};
```

`ROLE_CONFIG` is used to drive the UI (labels, register links) from data rather than hard-coded JSX conditionals.

---

## DTO Interfaces

DTOs (Data Transfer Objects) type what is sent to the API. They mirror the backend's expected request bodies.

```ts
export interface RegisterCustomerDTO {
  name: string;
  email: string;
  password: string;
  address: string;
  role?: ROLES;
  profile_image_url?: string;
}

export interface RegisterDriverDTO {
  name: string;
  email: string;
  password: string;
  phone: string;
  vehicle_type: VEHICLE_TYPE;
  profile_image_url?: string;
}
```

These are separate from the Zod form schemas. The schema includes `confirmPassword` for UI validation; the DTO does not — it is stripped in the hook before the service call.

---
