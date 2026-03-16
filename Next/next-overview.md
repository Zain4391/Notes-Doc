# Next.js Overview

Next.js is a full-stack React framework. The App Router (v13+) is the modern way to build — components are Server-first by default and you opt into the browser with `"use client"`.

---

## The Three Rendering Worlds

| Type                 | Where it runs            | Has hooks/state? | Use for                              |
| -------------------- | ------------------------ | ---------------- | ------------------------------------ |
| **Server Component** | Server only              | No               | Data fetching, DB calls, heavy logic |
| **Client Component** | Browser (hydrated)       | Yes              | Interactivity, events, hooks         |
| **SSG / ISR**        | Build time / revalidated | —                | Static content, landing pages        |

By default every component in the App Router is a **Server Component**. You opt into the browser world with `"use client"` at the very top of the file.

---

## The Backend Dev Mental Model

> A **Server Component** is like a controller + view. It runs on the server, can talk to your DB directly, and sends HTML down.
> A **Client Component** is your regular React — it lives in the browser.

```
NestJS / Spring              Next.js App Router
───────────────              ──────────────────
Controller + View   ≈        Server Component (async, fetches data, renders HTML)
Frontend JS         ≈        Client Component ("use client", hooks, events)
```

---

## Server Component (default)

```tsx
// app/restaurants/page.tsx — no "use client", runs on server
export default async function RestaurantsPage() {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/restaurants`, {
    cache: "no-store",
  });
  const restaurants = await res.json();

  return (
    <ul>
      {restaurants.map((r) => (
        <li key={r.id}>{r.name}</li>
      ))}
    </ul>
  );
}
```

Key points:

- It is an `async` function — you `await` data directly, no `useEffect` needed.
- Cannot use hooks, event handlers, or browser APIs.
- Renders HTML on the server and sends it to the browser.

---

## Client Component

```tsx
// components/SearchBar.tsx
"use client";

import { useState } from "react";

export default function SearchBar({
  onSearch,
}: {
  onSearch: (q: string) => void;
}) {
  const [query, setQuery] = useState("");

  return (
    <input
      value={query}
      onChange={(e) => {
        setQuery(e.target.value);
        onSearch(e.target.value);
      }}
      placeholder="Search..."
    />
  );
}
```

Key points:

- `"use client"` must be the very first line of the file.
- All hooks work here — `useState`, `useEffect`, custom hooks, etc.
- Rendered on the server once (for the initial HTML), then hydrated in the browser.

---

## Mixing Server and Client Components

Most common pattern — Server Component as the data-fetching shell, Client Components for interactive islands.

```tsx
// app/restaurants/page.tsx — Server Component
import SearchBar from "./SearchBar"; // Client Component — totally fine to import here

export default async function RestaurantsPage() {
  const restaurants = await fetchRestaurants();

  return (
    <div>
      <SearchBar onSearch={(q) => console.log(q)} />
      <ul>
        {restaurants.map((r) => (
          <li key={r.id}>{r.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### The one rule to remember

```
Server Component  ✅ can import  Client Component
Client Component  ❌ cannot import Server Component
```

The server can hand things down to the browser. The browser cannot pull back up to the server.

---

## Project Structure (Food Delivery Frontend)

```
app/                  → Next.js routes and layouts (App Router)
components/
  ui/                 → shadcn/ui primitives (Button, Input, etc.)
  shared/             → shared application components
hooks/                → custom hooks — orchestration layer
lib/                  → auth config (NextAuth), axios client, utilities
schemas/              → Zod validation schemas
services/             → raw API call layer
store/                → Zustand global state stores
types/                → TypeScript type and interface definitions
```

This mirrors the backend layering you already know:

```
Backend               Frontend
──────────            ────────
Controller    ≈       Component (renders UI, calls hooks)
Service       ≈       Custom Hook (orchestrates logic)
Repository    ≈       Service file (raw API calls)
DTOs          ≈       types/ + schemas/
```

---
