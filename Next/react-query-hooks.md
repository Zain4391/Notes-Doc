# React Query + Next.js — Patterns, Bugs & Lessons

Reference from the Food Delivery Frontend project (March 2026). Covers query hooks, mutation hooks, query key rules, the `isHydrated` pattern, paginated response shapes, and every real bug hit during the dashboard build.

---

## Folder Structure

```
hooks/
├── queries/
│   ├── useOrders.ts
│   ├── useCustomer.ts
│   ├── useDriver.ts
│   ├── useAdmin.ts
│   ├── useRestaurant.ts
│   └── useProfileImage.ts
└── mutations/
    ├── useOrderMutations.ts
    ├── useCustomerMutations.ts
    ├── useDriverMutations.ts
    ├── useAdminMutations.ts
    └── useRestaurantMutations.ts
```

The philosophy:

```
Services  = raw API calls, no UI concern
Hooks     = reusable data-fetching logic, no page concern
Pages     = consume hooks, handle UI
```

---

## The `isHydrated` Pattern — Blocking Queries Until the Token Is Ready

This is the most important pattern in the whole codebase. Without it, every authenticated query fires on mount before the JWT token is in the Zustand store, sending requests with no `Authorization` header and getting 403 responses from the backend.

### Why it happens

NextAuth stores the session. Zustand stores the token for Axios. They are synced by `AsyncBridge` in `provider.tsx` using a `useEffect`. But React Query hooks fire **synchronously on component mount**, and `useEffect` runs **after paint**. There’s always a gap.

```
Component mounts
  → React Query fires immediately → token is null → 403
  → useEffect runs (too late)
  → token synced into Zustand
```

### The fix — `isHydrated` flag

Add `isHydrated: boolean` to the Zustand auth store. `AsyncBridge` sets it to `true` after NextAuth resolves (status is no longer `"loading"`). Every query hook gates on it:

```ts
// store/auth.store.ts
export const useIsHydrated = () => useAuthStore((state) => state.isHydrated);
```

```ts
// AsyncBridge in provider.tsx
useEffect(() => {
  if (status === "loading") return;
  if (status === "authenticated" && session?.accessToken) {
    setAuth({ ... }, session.accessToken);
  }
  if (status === "unauthenticated") clearAuth();
  setHydrated(); // ← signals all queries to fire
}, [session, status]);
```

```ts
// Every query hook
export function useOrders(params?: OrderListParams) {
  const isHydrated = useIsHydrated();
  return useQuery({
    queryKey: ["orders", params],
    queryFn: () => orderService.getAllOrders(params),
    enabled: isHydrated, // ← blocks until token is ready
  });
}
```

### Composing `isHydrated` with other conditions

```ts
// Block until hydrated AND id exists
enabled: isHydrated && Boolean(id);

// Block until hydrated AND user is the right role
enabled: isHydrated && userType === "admin";

// Block until hydrated AND external flag
enabled: isHydrated && (options?.enabled ?? true);
```

**Important:** `isHydrated` is NOT persisted to localStorage. It must be set fresh on every page load by `AsyncBridge`. If you persist it, the flag stays `true` across sessions and the gap problem returns.

---

## Query Hooks

### Basic pattern

```ts
import { useQuery } from "@tanstack/react-query";
import { orderService } from "@/services/order.service";
import { useIsHydrated } from "@/store/auth.store";

export function useOrders(params?: OrderListParams) {
  const isHydrated = useIsHydrated();
  return useQuery({
    queryKey: ["orders", params],
    queryFn: () => orderService.getAllOrders(params),
    enabled: isHydrated,
  });
}

export function useOrder(id: string) {
  const isHydrated = useIsHydrated();
  return useQuery({
    queryKey: ["orders", id],
    queryFn: () => orderService.getOrderById(id),
    enabled: isHydrated && Boolean(id),
  });
}
```

### Optional `options` parameter for external control

```ts
export function useCustomers(
  params?: CustomerListParams,
  options?: { enabled?: boolean },
) {
  const isHydrated = useIsHydrated();
  return useQuery({
    queryKey: ["customers", params],
    queryFn: () => adminService.getAllCustomers(params),
    enabled: isHydrated && (options?.enabled ?? true),
  });
}

// usage — only fires if user is admin
const { data } = useCustomers({ limit: 1 }, { enabled: isAdmin });
```

### Role-gated profile hooks

When you call multiple profile hooks in the same component (e.g. `useProfileImage` calls all three), gate each on `userType` to prevent the wrong role hitting the wrong endpoint:

```ts
export function useCustomerProfile() {
  const isHydrated = useIsHydrated();
  const userType = useUserType();
  return useQuery({
    queryKey: ["customers", "profile"],
    queryFn: () => customerService.getProfile(),
    enabled: isHydrated && userType === "customer", // ← won't fire for drivers or admins
  });
}
```

---

## Query Key Rules ⚠️

The query key determines caching AND invalidation. Follow these rules strictly.

### Rule 1 — All keys for a resource share the same root

```ts
// ✅ correct — all start with "orders"
queryKey: ["orders", params];
queryKey: ["orders", id];
queryKey: ["orders", "customer", customerId, params];
queryKey: ["orders", "driver", driverId, params];

// ❌ wrong — invalidating ["orders"] won't catch these
queryKey: ["customer", { customerId, params }];
queryKey: ["driver", { driverId, params }];
```

### Rule 2 — Flatten params into the array

```ts
// ✅ correct
queryKey: ["orders", "customer", customerId, params];

// ❌ wrong — grouping hides the id
queryKey: ["orders", "customer", { customerId, params }];
```

### Rule 3 — `invalidateQueries` uses prefix matching

```ts
// Invalidates ALL orders queries
queryClient.invalidateQueries({ queryKey: ["orders"] });
// Catches: ["orders", params], ["orders", id], ["orders", "customer", ...], etc.
```

---

## `PaginatedResponse<T>` — Match Your Library's Actual Shape

This caused every paginated list to render empty in this project. The frontend had a custom type that didn't match what the backend actually returned.

### `nestjs-typeorm-paginate` actual output

```json
{
  "items": [...],
  "meta": {
    "totalItems": 50,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  },
  "links": { "first": "...", "previous": "", "next": "...", "last": "..." }
}
```

### Wrong type (what was in the codebase)

```ts
// ❌ This is a made-up shape — the backend never returns this
interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
}
// data?.data → undefined
// data?.total → undefined
// Every list shows empty state despite data being in the DB
```

### Correct type

```ts
// ✅ Matches nestjs-typeorm-paginate
interface PaginationMeta {
  totalItems?: number;
  itemCount: number;
  itemsPerPage: number;
  totalPages?: number;
  currentPage: number;
}

interface PaginatedResponse<T> {
  items: T[];
  meta: PaginationMeta;
  links?: { first?: string; previous?: string; next?: string; last?: string };
}
```

### Usage in pages

```ts
const customers = data?.items ?? []; // ✅ not data?.data
const total = data?.meta.totalItems ?? 0; // ✅ not data?.total
const totalPages = Math.ceil(total / PAGE_SIZE);
```

**Lesson:** Before writing a frontend type for a paginated response, check what the pagination library actually returns. Don't guess or invent a shape.

---

## Mutation Hooks

### Basic pattern

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

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

### Multi-arg mutations — always use a DTO

`mutate()` only accepts one argument. Wrap multiple values in a DTO:

```ts
// ✅ correct
export function useUpdateOrderStatus() {
  return useMutation({
    mutationFn: (payload: UpdateOrderStatusDTO) =>
      orderService.updateOrderStatus(payload.id, payload.status),
  });
}
updateStatus({ id: "uuid", status: "confirmed" });

// ❌ wrong — inline type, not importable, not reusable
mutationFn: ({ id, status }: { id: string; status: OrderStatus }) => ...
```

### All mutation callbacks

```ts
useMutation({
  onSuccess: (data, variables, context) => {}, // mutation succeeded
  onError: (error, variables, context) => {}, // mutation failed
  onSettled: (data, error, variables) => {}, // always — like finally
  onMutate: async (variables) => {}, // before — optimistic updates
});
```

`onSettled` is the right place to invalidate when you want to refetch regardless of success/failure.

### File upload mutations — disable the Axios timeout

File uploads via `multipart/form-data` go through the backend to Supabase Storage (two hops). This regularly exceeds a 10-second Axios timeout.

```ts
// lib/axios.ts — in the request interceptor
if (config.data instanceof FormData) {
  config.timeout = 0; // unlimited for file uploads
}
```

Without this, uploads complete on the server but the frontend throws `timeout of 10000ms exceeded` and the mutation shows an error even though the image was saved.

---

## Component Usage

```tsx
// Query
const { data, isLoading, isError } = useOrders({ page: 1, limit: 10 });
const orders = data?.items ?? []; // always use ?. and ?? []
const total = data?.meta.totalItems ?? 0;

// Mutation — single arg
const { mutate: deleteOrder, isPending } = useDeleteOrder();
deleteOrder("some-uuid");

// Mutation — DTO arg
const { mutate: updateStatus } = useUpdateOrderStatus();
updateStatus({ id: "some-uuid", status: "confirmed" });
```

---

## Zustand + React Query — `getState()` vs hooks

| Context                                           | Use                                   |
| ------------------------------------------------- | ------------------------------------- |
| React component / hook                            | `useAuthStore()`, `useIsAdmin()` etc. |
| Outside React (Axios interceptor, plain function) | `useAuthStore.getState()`             |

```ts
// ✅ Axios interceptor — NOT a React component
axiosInstance.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) config.headers.set("Authorization", `Bearer ${token}`);
  return config;
});

// ✅ React component — use the hook
const isAdmin = useIsAdmin();
const { data } = useCustomers({ limit: 1 }, { enabled: isAdmin });
```

`useAuthStore.getState()` reads Zustand imperatively outside React’s render cycle. Calling hooks inside interceptors throws.

---

## Real Mistakes Made (all 8 original + 6 new from March 2026 dashboard build)

### ❌ Mistake 1 — Wrong query key roots

```ts
// root is "customer" not "orders" — invalidateQueries(["orders"]) misses this
queryKey: ["customer", { customerId, params }];
```

---

### ❌ Mistake 2 — Importing non-existent DTOs

```ts
import { AssignDriverDTO, UpdateOrderStatusDTO } from "@/types/order.types";
// These didn't exist yet. Define DTOs in the types file before importing.
```

---

### ❌ Mistake 3 — Importing from wrong type file

```ts
// In useDriverMutations.ts — wrong file
import { UpdateProfileImageDTO } from "@/types/customer.types";
// ✅ correct
import { UpdateDriverProfileImgDTO } from "@/types/driver.types";
```

---

### ❌ Mistake 4 — Inconsistent invalidation keys

```ts
queryClient.invalidateQueries({ queryKey: ["order"] }); // singular — matches nothing
queryClient.invalidateQueries({ queryKey: ["order", "driver"] }); // made up
// ✅ always use the root: ["orders"], ["customers"], ["drivers"]
```

---

### ❌ Mistake 5 — Hook name clashes across mutation files

```ts
// useCustomerMutations.ts AND useDriverMutations.ts both exported:
export function useUpdateProfile() { ... }
export function useUpdatePassword() { ... }
// One shadows the other on import.
// ✅ prefix with resource: useUpdateDriverProfile, useUpdateDriverPassword
```

---

### ❌ Mistake 6 — Missing `enabled` on single-resource queries

```ts
// ❌ fires with empty string id
enabled: Boolean(id); // missing entirely
// ✅ always add for any hook that takes an id
```

---

### ❌ Mistake 7 — Profile query key too generic

```ts
// ❌ collides with useCustomers list query
queryKey: ["customers"];
// ✅ be specific
queryKey: ["customers", "profile"];
```

---

### ❌ Mistake 8 — Wrong params type for nested order queries

```ts
// ❌ CustomerListParams.sortBy = "name" | "email" | "created_at" — not order fields
export function useCustomerOrders(id: string, params?: CustomerListParams);
// ✅
export function useCustomerOrders(id: string, params?: OrderListParams);
```

---

### ❌ Mistake 9 — Queries firing before JWT token is in Zustand (403 on every load)

**Root cause:** `AsyncBridge` syncs the NextAuth session into Zustand via `useEffect`, which runs after paint. React Query hooks fire on mount. There’s always a window where the token is null.

**Symptom:** Every authenticated endpoint returns 403 on first page load. The backend logs show requests with no Authorization header.

**Fix:** `isHydrated` flag in Zustand. Set by `AsyncBridge` after session resolves. All hooks add `enabled: isHydrated`. See the [isHydrated Pattern](#the-ishydrated-pattern--blocking-queries-until-the-token-is-ready) section above.

```ts
// Every hook
const isHydrated = useIsHydrated();
return useQuery({ ..., enabled: isHydrated });
```

---

### ❌ Mistake 10 — `PaginatedResponse<T>` didn’t match the backend library

**Root cause:** Frontend type was `{ data: T[], total: number }`. Backend returns `nestjs-typeorm-paginate`’s `{ items: T[], meta: { totalItems, ... } }`. Every paginated list showed empty state despite data in the DB.

**Fix:** Updated `PaginatedResponse<T>` to match the actual library shape. All pages updated to use `.items` and `.meta.totalItems`.

**Also caught:** A typo `%${search}}%` (extra `}`) in the search query meant every search returned 0 results.

---

### ❌ Mistake 11 — Profile picture never showed (DTO field name mismatch)

**Root cause:** DB entity: `profile_image_url`. DTO property: `profile_img_url`. DTO constructor does `Object.assign(this, entity)` which copies `profile_image_url` — but the DTO’s `profile_img_url` was never set. API always returned `profile_img_url: undefined`.

**Fix (backend):** Added explicit mapping in both DTOs:

```ts
constructor(partial: Partial<CustomerResponseDTO> & { profile_image_url?: string }) {
  super(partial);
  Object.assign(this, partial);
  if (partial.profile_image_url !== undefined) {
    this.profile_img_url = partial.profile_image_url;
  }
}
```

**Lesson:** When entity column names differ from DTO property names, `Object.assign` silently fails — always add explicit mappings.

---

### ❌ Mistake 12 — Profile image never showed even after DTO fix

**Root cause:** `GET /customer/profile` returns `@CurrentUser()` — the `AuthenticatedUser` object from the JWT strategy’s `validate()` method. All three strategies did a fresh DB query per request but only returned `{ id, email, name, role, userType }`. `profile_image_url` was fetched but **silently dropped**.

**Fix (backend):** Added `profile_img_url` to `AuthenticatedUser` interface and mapped it in all three strategies:

```ts
return {
  id: customer.id,
  email: customer.email,
  name: customer.name,
  role: customer.role,
  userType: "customer",
  profile_img_url: customer.profile_image_url ?? undefined, // ← added
};
```

**Lesson:** When profile endpoints return JWT payload data (`@CurrentUser()`) rather than a DB query, check what the strategy’s `validate()` actually maps. Adding a field to the DTO is not enough if the strategy discards it before the controller runs.

---

### ❌ Mistake 13 — File upload timed out (10s Axios timeout)

**Symptom:** `[useUpdateProfileImage] timeout of 10000ms exceeded`. Backend confirmed the upload succeeded. Frontend showed an error.

**Root cause:** Axios global timeout of 10s. Image uploads route backend → Supabase Storage (two hops), regularly exceeding 10s.

**Fix:** In the Axios request interceptor, detect `FormData` and disable timeout:

```ts
if (config.data instanceof FormData) {
  config.timeout = 0; // unlimited for file uploads only
}
```

---

### ❌ Mistake 14 — `syncedRef` in `AsyncBridge` prevented re-sync

**Root cause:** Original `AsyncBridge` had `syncedRef.current = true` to run the sync only once. If the session wasn’t ready on the first `useEffect` run, the token was never synced on subsequent renders.

**Fix:** Removed `syncedRef`. `AsyncBridge` re-syncs on every `[session, status]` change. `setHydrated()` called unconditionally once status is no longer `"loading"`.

---

## The `useProfileImage` Pattern — Live Avatar Across the App

When you need the current user’s profile image in a shared component (like a Header) without knowing which role they are:

```ts
// hooks/queries/useProfileImage.ts
export function useProfileImage(): string {
  const isHydrated = useIsHydrated();
  const userType = useUserType();

  // All three called every render (rules of hooks)
  // but only one actually fetches because of the enabled flag
  const { data: adminProfile } = useAdminProfile();
  const { data: customerProfile } = useCustomerProfile();
  const { data: driverProfile } = useDriverProfile();

  if (!isHydrated) return "";

  switch (userType) {
    case "admin":
      return adminProfile?.profile_img_url ?? "";
    case "customer":
      return customerProfile?.profile_img_url ?? "";
    case "driver":
      return driverProfile?.profile_img_url ?? "";
    default:
      return "";
  }
}
```

Because each profile hook is already called on the settings page, the image is cached and the Header gets it for free with no extra network request. When the settings page upload mutation invalidates the query, the Header re-renders with the new image automatically.

---
