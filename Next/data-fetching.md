# Data Fetching in Next.js

The golden rule:

```
Static or non-interactive data?  → fetch in Server Component
Data driven by user interaction? → fetch in Client Component via React Query
```

---

## 1. Server Component Fetching

Server Components are `async` functions — you `await` data directly. One `cache` option on the `fetch` call controls the entire rendering strategy.

```tsx
// app/restaurants/page.tsx
export default async function RestaurantsPage() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/restaurants`, {
    cache: "no-store", // always fresh   → SSR
    // cache: 'force-cache'       // cache forever  → SSG
    // next: { revalidate: 60 }   // refresh every 60s → ISR
  });
  const data = await res.json();

  return (
    <ul>
      {data.map((r) => (
        <li key={r.id}>{r.name}</li>
      ))}
    </ul>
  );
}
```

No `useEffect`. No loading state. No API round-trip from the browser. The HTML arrives pre-populated.

---

## 2. Client-Side Fetching with React Query

For data that depends on user interaction (search, filters, pagination, mutations), React Query is the standard.

### Setup — Provider wrapper

```tsx
// app/provider.tsx  (from this project)
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { staleTime: 60 * 1000, retry: 1 },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

function getQueryClient() {
  if (typeof window === "undefined") return makeQueryClient(); // server: always fresh
  if (!browserQueryClient) browserQueryClient = makeQueryClient(); // browser: reuse
  return browserQueryClient;
}

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

The `getQueryClient` pattern avoids creating a new QueryClient on every server render while keeping one stable instance in the browser.

### Query hook (GET request)

```tsx
// hooks/useOrders.ts
import { useQuery } from "@tanstack/react-query";
import { customerService } from "@/services/customer.service";

export function useOrders(id: string) {
  return useQuery({
    queryKey: ["orders", id],
    queryFn: () => customerService.getOrders(id),
  });
}

// component usage
const { data, isPending, isError } = useOrders(customerId);
```

### Mutation hook (POST / PUT request)

```tsx
// hooks/useRegister.ts  (from this project)
import { useMutation } from '@tanstack/react-query'
import { authService } from '@/services/auth.service'
import { useRouter } from 'next/navigation'

export function useRegisterCustomer() {
  const router = useRouter()

  return useMutation({
    mutationFn: (values: RegisterCustomerFormValues) => {
      const { confirmPassword, ...payload } = values   // strip UI-only field
      return authService.registerCustomer(payload)
    },
    onSuccess: () => {
      router.push('/login?registered=customer')
    },
  })
}

// component usage
const mutation = useRegisterCustomer()
mutation.mutate(formValues)
if (mutation.isPending) // show spinner
if (mutation.isError)   // show error
```

### React Query vs raw useEffect

|                       | `useEffect` + `useState` | React Query            |
| --------------------- | ------------------------ | ---------------------- |
| Caching               | Manual                   | Automatic              |
| Background refetch    | Manual                   | Built-in               |
| Loading / error state | Manual `useState`        | `isPending`, `isError` |
| Deduplication         | No                       | Yes                    |
| Use when              | Simple one-off           | Anything non-trivial   |

---

## 3. The Service → Hook → Component Pattern

All data fetching in this project goes through three layers:

```
Component
  → calls useOrders() hook
      → calls customerService.getOrders()
          → hits the API via apiClient
              → returns typed data back up the chain
```

```ts
// services/customer.service.ts
export const customerService = {
  getOrders: (id: string, params?: { page?: number; limit?: number }) =>
    apiClient.get<PaginatedResponse<Order>>(`/customer/admin/orders/${id}`, {
      params,
    }),
};

// hooks/useOrders.ts
export function useOrders(id: string) {
  return useQuery({
    queryKey: ["orders", id],
    queryFn: () => customerService.getOrders(id),
  });
}

// components/OrderList.tsx
("use client");
const { data, isPending } = useOrders(customerId);
```

The component never knows about `fetch` or `axios`. The hook never knows about the URL structure. Each layer has one job.

---
