# React Query Hooks — Patterns, Rules & Mistakes

Reference from the Food Delivery Frontend project. Covers query hooks, mutation hooks, query key rules, and real mistakes made during implementation.

---

## Folder Structure

```
hooks/
├── queries/
│   ├── useOrders.ts
│   ├── useCustomer.ts
│   ├── useDriver.ts
│   └── useRestaurant.ts
└── mutations/
    ├── useOrderMutations.ts
    ├── useCustomerMutations.ts
    ├── useDriverMutations.ts
    └── useRestaurantMutations.ts
```

The philosophy:

```
Services  = raw API calls, no UI concern
Hooks     = reusable data-fetching logic, no page concern
Pages     = consume hooks, handle UI
```

---

## Query Hooks

### Basic pattern

```ts
import { useQuery } from "@tanstack/react-query";
import { orderService, OrderListParams } from "@/services/order.service";

export function useOrders(params?: OrderListParams) {
  return useQuery({
    queryKey: ["orders", params],
    queryFn: () => orderService.getAllOrders(params),
  });
}

export function useOrder(id: string) {
  return useQuery({
    queryKey: ["orders", id],
    queryFn: () => orderService.getOrderById(id),
    enabled: Boolean(id),   // ← don't fire if id is empty
  });
}
```

### `enabled` flag

Any hook that takes an `id` or a condition must have `enabled`:

```ts
// ✅ correct — only fires when customerId exists
export function useCustomerOrders(customerId: string, params?: OrderListParams) {
  return useQuery({
    queryKey: ["orders", "customer", customerId, params],
    queryFn: () => orderService.getOrdersByCustomer(customerId, params),
    enabled: Boolean(customerId),
  });
}
```

### Optional `options` parameter

When you need to conditionally disable a hook from the call site (e.g. admin-only queries):

```ts
export function useCustomers(
  params?: CustomerListParams,
  options?: { enabled?: boolean },
) {
  return useQuery({
    queryKey: ["customers", params],
    queryFn: () => adminService.getAllCustomers(params),
    enabled: options?.enabled ?? true,
  });
}

// usage — only fires if user is admin
const { data } = useCustomers({ limit: 1 }, { enabled: isAdmin });
```

---

## Query Key Rules ⚠️

This is where mistakes were made. The query key determines caching AND invalidation. Follow these rules strictly.

### Rule 1 — All keys for a resource share the same root

```ts
// ✅ correct — all start with "orders"
queryKey: ["orders", params]
queryKey: ["orders", id]
queryKey: ["orders", "customer", customerId, params]
queryKey: ["orders", "driver", driverId, params]
queryKey: ["orders", "restaurant", restaurantId, params]

// ❌ wrong — different roots, invalidating ["orders"] won't catch these
queryKey: ["customer", { customerId, params }]
queryKey: ["driver", { driverId, params }]
queryKey: ["restaurant", { restaurantId, params }]
```

### Rule 2 — Flatten params into the array, don't group into an object

```ts
// ✅ correct
queryKey: ["orders", "customer", customerId, params]

// ❌ wrong — grouping hides the id from the key hierarchy
queryKey: ["orders", "customer", { customerId, params }]
```

### Rule 3 — Don't use generic roots that clash with other resources

```ts
// ❌ wrong — "driver" root will clash with useDrivers hook keys
queryKey: ["driver", { driverId, params }]

// ✅ correct — nested under "orders"
queryKey: ["orders", "driver", driverId, params]
```

### Why this matters — `invalidateQueries` uses prefix matching

```ts
// This invalidates EVERYTHING starting with "orders"
queryClient.invalidateQueries({ queryKey: ["orders"] });

// With correct keys, this catches:
// ["orders", params]
// ["orders", id]
// ["orders", "customer", customerId, params]
// ["orders", "driver", driverId, params]
// etc.
```

---

## Mutation Hooks

### Basic pattern

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { orderService } from "@/services/order.service";
import { AppException } from "@/types/api.types";
import { CreateOrderDTO, OrderStatus } from "@/types/order.types";

export function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (payload: CreateOrderDTO) => orderService.createOrder(payload),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["orders"] });
    },
    onError: (error: AppException) => {
      console.error("[useCreateOrder]", error.message);
    },
  });
}
```

### Multi-arg mutations — use a DTO, not an inline type

```ts
// ✅ correct — define a DTO in types file, import it
export interface UpdateOrderStatusDTO {
  id: string;
  status: OrderStatus;
}

export function useUpdateOrderStatus() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (payload: UpdateOrderStatusDTO) =>
      orderService.updateOrderStatus(payload.id, payload.status),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["orders"] });
    },
    onError: (error: AppException) => {
      console.error("[useUpdateOrderStatus]", error.message);
    },
  });
}

// ❌ wrong — inline object type, not reusable, not importable
mutationFn: ({ id, status }: { id: string; status: OrderStatus }) => ...
```

`mutationFn` only receives one argument from `mutate()`. When you need multiple values, wrap them in a DTO object.

### `onError` — always add it

```ts
onError: (error: AppException) => {
  console.error("[hookName]", error.message);
},
```

Handle generic logging in the hook. Let the component handle UI feedback on top:

```ts
const { mutate, isError, error } = useUpdateOrderStatus();
// component decides whether to show a toast, inline message, etc.
```

### Mutation callbacks

```ts
useMutation({
  mutationFn: ...,
  onSuccess: (data, variables, context) => { },   // mutation succeeded
  onError:   (error, variables, context) => { },  // mutation failed
  onSettled: (data, error, variables, context) => { }, // always runs (like finally)
  onMutate:  async (variables) => { },             // runs BEFORE — used for optimistic updates
})
```

`onSettled` is useful when you want to invalidate regardless of success or failure.

---

## Component Usage

```tsx
// Query
const { data, isLoading, isError } = useOrders({ page: 1, limit: 10 });

// Mutation — single arg
const { mutate: deleteOrder, isPending } = useDeleteOrder();
deleteOrder("some-uuid");

// Mutation — DTO arg
const { mutate: updateStatus } = useUpdateOrderStatus();
updateStatus({ id: "some-uuid", status: "confirmed" });
```

---

## Real Mistakes Made

### ❌ Mistake 1 — Wrong query key roots

```ts
// Made this mistake in useOrders.ts
queryKey: ["customer", { customerId, params }]  // root is "customer" not "orders"
queryKey: ["driver", { driverId, params }]       // root is "driver" not "orders"
```

**Why it's wrong:** `invalidateQueries({ queryKey: ["orders"] })` in mutations wouldn't catch these. Customer and driver order queries would go stale and never refetch after a mutation.

---

### ❌ Mistake 2 — Importing non-existent DTOs

```ts
// Tried to import these which didn't exist in types
import { AssignDriverDTO, UpdateOrderStatusDTO } from "@/types/order.types";
```

**Fix:** Either define the DTOs in the types file first, or use inline types. We chose to define them — keeps things consistent with `CreateOrderDTO` and other DTOs.

---

### ❌ Mistake 3 — Importing from wrong type file

```ts
// In useDriverMutations.ts — imported customer DTOs for driver mutations
import { UpdateProfileImageDTO, UpdateProfilePasswordDTO } from "@/types/customer.types";
```

**Fix:** Driver has its own DTOs in `driver.types.ts`:
```ts
import { UpdateDriverProfileImgDTO, UpdateDriverProfilePasswordDTO } from "@/types/driver.types";
```

---

### ❌ Mistake 4 — Inconsistent invalidation keys in mutations

```ts
// useCreateOrder was invalidating ["order"] (singular) — doesn't match any key
queryClient.invalidateQueries({ queryKey: ["order"] });

// useAssignDriver was invalidating a made-up key
queryClient.invalidateQueries({ queryKey: ["order", "driver"] });
```

**Fix:** Always invalidate the root: `["orders"]`, `["customers"]`, `["drivers"]`, `["restaurants"]`.

---

### ❌ Mistake 5 — Hook names clashing across files

```ts
// useCustomerMutations.ts
export function useUpdateProfile() { ... }
export function useUpdatePassword() { ... }

// useDriverMutations.ts — same names!
export function useUpdateProfile() { ... }
export function useUpdatePassword() { ... }
```

**Why it's a problem:** When both are imported in the same file, one will shadow the other.

**Fix:** Prefix driver mutations with the resource name:
```ts
export function useUpdateDriverProfile() { ... }
export function useUpdateDriverPassword() { ... }
export function useUpdateDriverProfileImage() { ... }
```

---

### ❌ Mistake 6 — Missing `enabled` on single-resource queries

```ts
// ❌ will fire even with empty string id
export function useCustomerById(id: string) {
  return useQuery({
    queryKey: ["customers", id],
    queryFn: () => adminService.getCustomerById(id),
    // missing enabled!
  });
}

// ✅ correct
enabled: Boolean(id),
```

---

### ❌ Mistake 7 — Wrong profile query key

```ts
// ❌ ["customers"] collides with useCustomers list query key
export function useCustomerProfile() {
  return useQuery({
    queryKey: ["customers"],   // too generic, matches list query
    queryFn: () => customerService.getProfile(),
  });
}

// ✅ correct — be specific
queryKey: ["customers", "profile"],
```

---

### ❌ Mistake 8 — Wrong params type for nested orders

```ts
// ❌ used CustomerListParams for order filtering
export function useCustomerOrders(id: string, params?: CustomerListParams)

// ✅ correct — orders have their own params type
export function useCustomerOrders(id: string, params?: OrderListParams)
```

`CustomerListParams.sortBy` is constrained to `"name" | "email" | "created_at"` — those are customer fields, not order fields.

---

## Zustand + React Query — `getState()` vs hooks

| Context | Use |
|---|---|
| React component / hook | `useAuthStore()`, `useIsAdmin()` etc. |
| Outside React (Axios interceptor, useEffect body) | `useAuthStore.getState()` |

```ts
// ✅ Axios interceptor — NOT a React component, use getState()
axiosInstance.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.set("Authorization", `Bearer ${token}`);
  return config;
});

// ✅ React component — use the hook
const isAdmin = useIsAdmin();
const { data } = useCustomers({ limit: 1 }, { enabled: isAdmin });
```

`useAuthStore.getState()` reads Zustand's store imperatively outside React's render cycle. Calling hooks inside interceptors or non-React functions will throw.
