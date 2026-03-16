# NextAuth — Multi-Provider Authentication

This project uses NextAuth v4 with three separate Credentials providers — one per user type (Customer, Driver, Admin). Each provider has its own `authorize` function that hits a different backend endpoint.

---

## Why Multiple Providers

Each user type lives in a different table and returns a different shape of user data. Separate providers mean separate `authorize` logic — no `if/else` chains based on role inside a single provider.

```ts
// lib/auth.ts (simplified)
providers: [
  Credentials({ id: 'customer-login', ... async authorize() { /* hit /auth/customer/login */ } }),
  Credentials({ id: 'driver-login',   ... async authorize() { /* hit /auth/driver/login  */ } }),
  Credentials({ id: 'admin-login',    ... async authorize() { /* hit /auth/admin/login   */ } }),
]
```

The `id` on each provider is what `PROVIDER_MAP` references when calling `signIn()`.

---

## The authorize Function

Each provider's `authorize` function:

1. Calls the backend login endpoint via `apiClient`
2. Receives `{ access_token, user }` from the backend
3. Returns a `User` object that NextAuth stores in the JWT

```ts
Credentials({
  id: "customer-login",
  name: "Customer",
  credentials: {
    email: { label: "Email", type: "email" },
    password: { label: "Password", type: "password" },
  },
  async authorize(credentials) {
    try {
      const response = await apiClient.post<AuthResponseDTO>(
        "/api/auth/customer/login",
        { email: credentials?.email, password: credentials?.password },
      );
      const { access_token, user } = response;

      return {
        id: user.id,
        name: user.name,
        email: user.email,
        accessToken: access_token,
        role: ROLES.CUSTOMER,
        userType: "customer" as const,
      };
    } catch (error) {
      if (error instanceof AppException) throw new Error(error.message);
      throw new Error("Something went wrong");
    }
  },
});
```

Key points:

- `authorize` must return `null` (failed) or a user object (success). Throwing `new Error(message)` surfaces the message on the client.
- `AppException` is caught and its message is re-thrown so the UI gets meaningful errors.
- `role` and `userType` are added here — they are not in the default NextAuth `User` type, which is why module augmentation is needed.

---

## JWT and Session Callbacks

NextAuth uses two callbacks to pass custom fields through the token lifecycle:

```ts
callbacks: {
  // Runs when JWT is created or updated. Attach custom fields from User to token.
  async jwt({ token, user }) {
    if (user) {
      const typedUser = user as User
      token.id          = typedUser.id
      token.accessToken = typedUser.accessToken
      token.role        = typedUser.role
      token.userType    = typedUser.userType
    }
    return token
  },

  // Runs when session is read. Copy fields from token into the session object.
  async session({ session, token }) {
    session.user.id       = token.id as string
    session.user.role     = token.role as ROLES
    session.user.userType = token.userType as UserType
    session.accessToken   = token.accessToken as string
    return session
  },
}
```

Flow:

```
authorize() returns User
  → jwt() copies fields onto the JWT token (stored as a cookie)
      → session() copies fields onto the session object
          → useSession() / getSession() returns the session in components
```

---

## Module Augmentation — Extending NextAuth Types

NextAuth's default `Session`, `User`, and `JWT` types do not include `role`, `userType`, or `accessToken`. Module augmentation extends them without modifying the library.

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
    } & DefaultSession["user"]; // keeps name, email, image from default
  }
}

interface User {
  id: string;
  name: string;
  email: string;
  role: ROLES;
  userType: UserType;
  accessToken: string;
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

Without this, TypeScript would error when accessing `session.user.role` or `session.accessToken`.

---

## Zustand Sync Bridge

NextAuth manages the session. Zustand manages client-side auth state. They need to stay in sync. The `AsyncBridge` component (in `provider.tsx`) handles this:

```ts
function AsyncBridge() {
  const { data: session, status } = useSession();
  const { setAuth, clearAuth } = useAuthStore();

  useEffect(() => {
    if (status === "loading") return;

    if (status === "authenticated" && session) {
      setAuth(
        {
          id: session.user.id,
          name: session.user.name ?? "",
          email: session.user.email ?? "",
          role: session.user.role,
          userType: session.user.userType,
        },
        session.accessToken,
      );
    }

    if (status === "unauthenticated") clearAuth();
  }, [session, status, setAuth, clearAuth]);

  return null; // renders nothing — only syncs state
}
```

`AsyncBridge` is a renderless component. It only runs side effects. When the NextAuth session changes, Zustand reflects it immediately.

---

## Auth Routing Maps

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
```

These maps turn conditional logic into data. Adding a new user type means updating the map — not hunting down `if/else` branches across multiple files. TypeScript's `Record<UserType, string>` enforces that every user type has an entry.

---
