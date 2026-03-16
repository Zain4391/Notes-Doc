# Custom Hooks Pattern

This project uses a three-layer architecture that mirrors backend layering. The goal is to keep components dumb, hooks focused, and services pure.

---

## The Three Layers

```
Component       → renders UI, calls hooks, knows nothing about HTTP
  ↓
Custom Hook     → orchestrates logic, loading/error state, navigation
  ↓
Service         → raw API calls, typed, no state
```

This maps directly to what you know from the backend:

```
Backend              Frontend
───────              ────────
Controller    ≈      Component
Service       ≈      Custom Hook
Repository    ≈      Service file
```

---

## Layer 1 — Service (raw API calls)

Service files contain only API calls. No state, no hooks, no navigation. Just typed functions that hit endpoints.

```ts
// services/customer.service.ts
export const customerService = {
  getProfile: () => apiClient.get<Customer>("/customer/profile"),

  updateProfile: (id: string, data: UpdateCustomerDTO) =>
    apiClient.put<Customer>(`/customer/update/${id}`, data),

  getOrders: (id: string, params?: { page?: number; limit?: number }) =>
    apiClient.get<PaginatedResponse<Order>>(`/customer/admin/orders/${id}`, {
      params,
    }),

  uploadProfileImage: (id: string, file: File) => {
    const formData = new FormData();
    formData.append("file", file);
    return apiClient.post<Customer>(
      `/customer/upload-profile-image/${id}`,
      formData,
      { headers: { "Content-Type": "multipart/form-data" } },
    );
  },
};
```

---

## Layer 2a — Hook with manual state (login)

For flows that don't need React Query caching, use a hook with manual `useState`.

```ts
// hooks/useLogin.ts
export function useLogin(userType: UserType) {
  const router = useRouter();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const login = async (values: LoginFormValues) => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await signIn(PROVIDER_MAP[userType], {
        email: values.email,
        password: values.password,
        redirect: false,
      });
      if (result?.error) {
        setError(result.error);
        return;
      }
      router.push(REDIRECT_MAP[userType]);
      router.refresh();
    } catch (err) {
      setError(getErrorMessage(err));
    } finally {
      setIsLoading(false);
    }
  };

  return { login, isLoading, error };
}
```

What this hook does:

- Accepts a `userType` to select the correct NextAuth provider via `PROVIDER_MAP`.
- Manages its own `isLoading` and `error` state.
- `finally` ensures `isLoading` always resets — even on unexpected throws.
- The component calling this hook never touches `signIn` or routing directly.

---

## Layer 2b — Hook with React Query mutation (registration)

For mutations that benefit from React Query's lifecycle (`isPending`, `isError`, `onSuccess`), use `useMutation`.

```ts
// hooks/useRegister.ts
export function useRegisterCustomer() {
  const router = useRouter();

  return useMutation({
    mutationFn: (values: RegisterCustomerFormValues) => {
      const { confirmPassword, ...payload } = values; // strip UI-only field before sending
      return authService.registerCustomer(payload);
    },
    onSuccess: () => {
      router.push("/login?registered=customer");
    },
  });
}
```

Usage in a component:

```tsx
"use client"
const mutation = useRegisterCustomer()

<form onSubmit={handleSubmit(data => mutation.mutate(data))}>
  {mutation.isPending && <Spinner />}
  {mutation.isError   && <p>{mutation.error.message}</p>}
  <button type="submit">Register</button>
</form>
```

---

## Layer 3 — Component (dumb, renders only)

```tsx
// app/login/page.tsx (simplified)
"use client";

export default function LoginPage() {
  const { login, isLoading, error } = useLogin("customer");
  const { register, handleSubmit } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
  });

  return (
    <form onSubmit={handleSubmit(login)}>
      <input {...register("email")} />
      <input {...register("password")} type="password" />
      {error && <p className="text-red-500">{error}</p>}
      <button disabled={isLoading}>
        {isLoading ? "Signing in..." : "Sign In"}
      </button>
    </form>
  );
}
```

The component knows nothing about `signIn`, `PROVIDER_MAP`, routing, or error extraction. It only renders.

---

## Why This Pattern

- **Testable** — hooks can be tested in isolation without rendering a component.
- **Reusable** — `useLogin('driver')` reuses the same hook for a different user type.
- **Maintainable** — changing the API endpoint means changing the service file only.
- **Clean components** — components stay small and easy to read.

---
