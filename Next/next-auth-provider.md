# NextAuth — Multi-Provider Authentication

This project uses **NextAuth v5 (beta)** with three separate Credentials providers — one per user type (Customer, Driver, Admin). Each provider has its own `authorize` function that hits a different backend endpoint.

> **Important:** The package is installed as `next-auth@beta`. Do NOT install `next-auth@^4.x` — v4 does not export `handlers`, `auth`, `signIn`, or `signOut` from a single config file, which is the pattern this project uses. Running `npm install next-auth@beta` is the correct install command.

---

## v4 vs v5 — Key Differences

| | v4 | v5 (beta) — this project |
|---|---|---|
| Route handler | `export default NextAuth(authOptions)` | `export const { GET, POST } = handlers` |
| Session (server) | `getServerSession(authOptions)` | `auth()` from `lib/auth.ts` |
| Session (client) | `useSession()` / `getSession()` | `useSession()` (unchanged) |
| Config location | Exported `authOptions` object | `NextAuth({...})` called once, exports named bindings |
| `handlers` export | ❌ Does not exist | ✅ `{ handlers, auth, signIn, signOut }` |

---

## Why Multiple Providers

Each user type lives in a different table and returns a different shape of user data. Separate providers mean separate `authorize` logic — no `if/else` chains based on role inside a single provider.

```ts
// lib/auth.ts (simplified)
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Credentials({ id: "customer-login", ... async authorize() { /* hit /auth/customer/login */ } }),
    Credentials({ id: "driver-login",   ... async authorize() { /* hit /auth/driver/login  */ } }),
    Credentials({ id: "admin-login",    ... async authorize() { /* hit /auth/admin/login   */ } }),
  ],
});
```

The `id` on each provider is what `PROVIDER_MAP` references when calling `signIn()`.

---

## Route Handler

```ts
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/lib/auth";

export const { GET, POST } = handlers;
```

This is the only route file needed. It delegates everything to NextAuth via the `handlers` export.

---

## The authorize Function

Each provider's `authorize` function:

1. Calls the backend login endpoint via `apiClient` (**server-side instance only — see Axios section below**)
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

## Two Axios Instances — Client vs Server

This is one of the most important architectural points in the project.

### The Problem

`lib/axios.ts` uses `getSession()` from `next-auth/react` in its request interceptor to attach a Bearer token. `next-auth/react` is a **browser-only package** — it cannot be imported in any file that runs on the server.

`lib/auth.ts` runs entirely on the server (it's the NextAuth config). If it imports from `lib/axios.ts`, the module evaluation crashes, `NextAuth(...)` never completes, and `handlers` comes back as `undefined` — causing the `Cannot destructure property 'GET' of handlers` error.

### The Fix — Two Separate Instances

```
lib/axios.ts          ← browser/client context
  uses getSession()     attaches Bearer token from session cookie
  used by: services, React Query hooks, components

lib/axios.server.ts   ← server context only
  no getSession()       no browser APIs
  used by: lib/auth.ts (NextAuth authorize callbacks) ONLY
```

```ts
// lib/axios.server.ts — minimal, no auth interceptor
const axiosServerInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10000,
  headers: { "Content-Type": "application/json" },
});
// Response interceptor only — same error handling as axios.ts
// NO request interceptor with getSession()
```

```ts
// lib/auth.ts — always import from axios.server, never from axios
import { apiClient } from "./axios.server"; // ✅
// import { apiClient } from "./axios";     // ❌ crashes on server
```

The `authorize` callbacks don't need a Bearer token — they ARE the login. There is no existing session to read at that point.

### Rule of Thumb

| Where the code runs | Which axios to import |
|---|---|
| React components, hooks, services | `lib/axios.ts` |
| `lib/auth.ts`, Server Actions, server Route Handlers | `lib/axios.server.ts` |

Never import `next-auth/react` in any file that gets evaluated on the server.

---

## JWT and Session Callbacks

NextAuth uses two callbacks to pass custom fields through the token lifecycle:

```ts
callbacks: {
  // Runs when JWT is created or updated. Attach custom fields from User to token.
  async jwt({ token, user }) {
    if (user) {
      const typedUser = user as User;
      token.id          = typedUser.id;
      token.accessToken = typedUser.accessToken;
      token.role        = typedUser.role;
      token.userType    = typedUser.userType;
    }
    return token;
  },

  // Runs when session is read. Copy fields from token into the session object.
  async session({ session, token }) {
    session.user.id       = token.id as string;
    session.user.role     = token.role as ROLES;
    session.user.userType = token.userType as UserType;
    session.accessToken   = token.accessToken as string;
    return session;
  },
},
```

Flow:

```
authorize() returns User
  → jwt() copies fields onto the JWT token (stored as a cookie)
      → session() copies fields onto the session object
          → useSession() / auth() returns the session in components
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

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3000   # NestJS backend port
NEXTAUTH_URL=http://localhost:4200          # Next.js dev server port (next dev -p 4200)
NEXTAUTH_SECRET=your-secret-here           # used by v4 compat layer
AUTH_SECRET=your-secret-here               # used by v5 — keep both
```

Key points:
- `NEXTAUTH_URL` must match the port Next.js actually runs on. This project uses `-p 4200`.
- `NEXT_PUBLIC_API_URL` points to the **backend** (NestJS on 3000), NOT the Next.js server.
- Both `NEXTAUTH_SECRET` and `AUTH_SECRET` should be set to the same value for compatibility.
- Missing `NEXTAUTH_URL` causes broken redirects and callback URL failures.

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
  driver:   "driver-login",
  admin:    "admin-login",
};

export const REDIRECT_MAP: Record<UserType, string> = {
  customer: "/customer/dashboard",
  driver:   "/driver/dashboard",
  admin:    "/admin/dashboard",
};
```

These maps turn conditional logic into data. Adding a new user type means updating the map — not hunting down `if/else` branches across multiple files. TypeScript's `Record<UserType, string>` enforces that every user type has an entry.

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Cannot destructure property 'GET' of handlers — undefined` | `next-auth@v4` installed, which has no `handlers` export | `npm install next-auth@beta` |
| `CLIENT_FETCH_ERROR — Unexpected token '<'` | NextAuth route crashing and returning HTML instead of JSON | Fix the root crash first (usually the handlers error above) |
| `useRouter` not mounted | Importing from `next/router` (Pages Router) in an App Router project | Change to `next/navigation` |
| Hook needs `"use client"` | React hook used in a Server Component | Add `"use client"` at top of the file |
| `Controller` not found in `react-hook-form` | Page treated as RSC — Next.js resolves server-safe bundle which strips client exports | Add `"use client"` to the page |
| `getSession` crash on server | `next-auth/react` imported in a server-side file | Use `lib/axios.server.ts` instead of `lib/axios.ts` in server-only files |
