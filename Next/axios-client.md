# Axios HTTP Client

This project wraps Axios into a typed `apiClient` with two interceptors — one for injecting the JWT token, one for unwrapping the API response envelope and normalising errors.

---

## The Axios Instance

```ts
// lib/axios.ts
const axiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL || "http://localhost:3000",
  timeout: 10000,
  headers: { "Content-Type": "application/json" },
});
```

One instance, one base URL. All service files use this instance so the base URL is never repeated.

---

## Request Interceptor — JWT Injection

```ts
axiosInstance.interceptors.request.use(
  async (config: InternalAxiosRequestConfig) => {
    const session = await getSession(); // reads NextAuth session
    if (session?.accessToken) {
      config.headers.set("Authorization", `Bearer ${session.accessToken}`);
    }
    return config;
  },
  (error: AxiosError) => Promise.reject(error),
);
```

Every outgoing request automatically gets the JWT from the NextAuth session attached as a Bearer token. No service file needs to manually set the `Authorization` header.

Why `getSession()` and not Zustand? `getSession()` works in both Client Components and Server Components. Zustand is browser-only.

---

## Response Interceptor — Envelope Unwrapping + Typed Errors

The backend wraps every response in an envelope:

```json
{
  "statusCode": 200,
  "success": true,
  "data": { ... },
  "message": "OK"
}
```

The interceptor unwraps it so service files receive `data` directly, not the envelope:

```ts
axiosInstance.interceptors.response.use(
  (response: AxiosResponse<ApiSuccessResponse<unknown>>) => {
    return response.data.data as AxiosResponse; // unwrap the envelope
  },
  (error: AxiosError<ApiErrorResponse>) => {
    if (error.response?.data?.error) {
      throw new AppException(error.response.data.error); // typed error
    }
    throw new AppException({
      message: error.message ?? "Network error",
      statusCode: error.response?.status ?? 500,
      timestamp: new Date().toISOString(),
    });
  },
);
```

Every error thrown is an `AppException` — a typed class that includes `statusCode` and `timestamp`. Hooks and components can `instanceof AppException` to handle errors predictably.

---

## The Typed apiClient

Instead of using `axiosInstance` directly and typecasting everywhere, a thin `apiClient` object exposes all five HTTP methods with generics:

```ts
export const apiClient = {
  get: <T>(url: string, config?: object): Promise<T> =>
    axiosInstance.get(url, config),
  post: <T>(url: string, data?: unknown, config?: object): Promise<T> =>
    axiosInstance.post(url, data, config),
  put: <T>(url: string, data?: unknown, config?: object): Promise<T> =>
    axiosInstance.put(url, data, config),
  patch: <T>(url: string, data?: unknown, config?: object): Promise<T> =>
    axiosInstance.patch(url, data, config),
  delete: <T>(url: string, config?: object): Promise<T> =>
    axiosInstance.delete(url, config),
};
```

Usage in a service file — no casting needed:

```ts
// services/customer.service.ts
getProfile: () => apiClient.get<Customer>('/customer/profile'),
```

TypeScript knows the return type is `Promise<Customer>` without any assertion.

---

## Why This Approach

| Concern             | How it is handled                                  |
| ------------------- | -------------------------------------------------- |
| Auth header         | Request interceptor reads NextAuth session         |
| Response envelope   | Response interceptor unwraps `data`                |
| Error normalisation | All errors become `AppException` with typed fields |
| Return types        | Generic `apiClient` methods — no manual casting    |
| Base URL            | Set once in `axios.create()`                       |

---
