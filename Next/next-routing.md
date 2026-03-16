# Routing in Next.js (App Router)

Folders define the URL structure. Special filenames define behaviour at each segment.

---

## Basic Routing

```
app/
├── page.tsx              → /
├── about/
│   └── page.tsx          → /about
└── users/
    └── page.tsx          → /users
```

`page.tsx` is the magic filename. A folder only becomes a publicly accessible route when it has a `page.tsx` inside it.

---

## Dynamic Routes

Square brackets create a dynamic segment — same concept as `:id` in NestJS or `@PathVariable` in Spring.

```
app/
└── users/
    ├── page.tsx           → /users
    └── [id]/
        └── page.tsx       → /users/1, /users/42, /users/abc
```

```tsx
// app/users/[id]/page.tsx
export default function UserPage({ params }: { params: { id: string } }) {
  return <div>User ID: {params.id}</div>;
}
```

---

## Nested Layouts

Each folder can have its own `layout.tsx`. Child routes inherit all parent layouts — they stack from the root downward.

```
app/
├── layout.tsx             → root layout (always present)
└── dashboard/
    ├── layout.tsx         → dashboard layout (wraps all dashboard routes)
    ├── page.tsx           → /dashboard        (root + dashboard layout)
    ├── settings/
    │   └── page.tsx       → /dashboard/settings  (root + dashboard layout)
    └── analytics/
        └── page.tsx       → /dashboard/analytics (root + dashboard layout)
```

---

## Route Groups

Parentheses in a folder name create a route group. The name is **ignored in the URL**. Used to organise files and apply different layouts to different sections without changing URLs.

```
app/
├── (auth)/
│   ├── layout.tsx         → auth layout (no navbar, centered card)
│   ├── login/
│   │   └── page.tsx       → /login
│   └── register/
│       └── page.tsx       → /register
└── (main)/
    ├── layout.tsx         → main layout (full navbar, sidebar)
    └── dashboard/
        └── page.tsx       → /dashboard
```

Real use case in this project: auth pages (login, register) have no navbar. Dashboard pages have a full sidebar. Route groups let you apply separate layouts to each without affecting the URL.

---

## Catch-All Routes

`[...slug]` catches any number of path segments as an array.

```
app/
└── docs/
    └── [...slug]/
        └── page.tsx       → /docs/a, /docs/a/b, /docs/a/b/c
```

```tsx
export default function DocsPage({ params }: { params: { slug: string[] } }) {
  // /docs/a/b/c → params.slug = ['a', 'b', 'c']
}
```

Used in this project for the NextAuth catch-all API route:

```
app/
└── api/
    └── auth/
        └── [...nextauth]/
            └── route.ts   → handles all /api/auth/* paths (GET + POST)
```

```ts
// app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

---

## Special Files Cheatsheet

| File            | Purpose                                                    |
| --------------- | ---------------------------------------------------------- |
| `page.tsx`      | The UI for a route — makes the folder publicly accessible  |
| `layout.tsx`    | Persistent shell wrapping all child routes in that segment |
| `loading.tsx`   | Suspense-based loading UI shown while the page is fetching |
| `error.tsx`     | Error boundary for that route segment                      |
| `not-found.tsx` | 404 UI for that segment                                    |
| `route.ts`      | API endpoint (GET, POST, etc.) — no UI, server-only        |

---
