# React Setup Basics

To initialize a new react project, use the following command:

```bash
npm create vite@latest my-react-app -- --template react-ts # replace my-react-app with original project name
```

Then follow the prompts and wait for your project to be initialized.

## Configure Tailwind

At the moment, the latest version of tailwind is v4. It is configured using the following steps:

```bash
npm install tailwindcss @tailwindcss/vite
```

And then in _==vite.config.ts==_ file

```ts
import { defineConfig } from 'vite'import tailwindcss from '@tailwindcss/vite'
export default defineConfig({ plugins: [ tailwindcss(), ],})
```

Then inside your global.css file add

```css
@import "tailwindcss";
```

## Framework to get Started

These steps are to be performed before writing any frontend UI code. They will take time but eventually help in the longer run

- Setup the models (Database tables <==> Frontend interfaces mapping)
- Set up axios instance
- Wire up React Query.
- Set up Zustand interfaces
- Set up design system (including themes, fonts etc)
- Set up theme context
- Implement the Zustand interfaces

## Code Samples for React Query set up

```ts
import { QueryClient } from "@tanstack/react-query";

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 1000 * 60 * 5,
    },
  },
});

export default queryClient;

// Inside Main.tsx file

import { StrictMode } from "react";
import { createRoot } from "react-dom/client";

import App from "./App.tsx";
import { QueryClientProvider } from "@tanstack/react-query";
import queryClient from "./query.config.ts";

createRoot(document.getElementById("root")!).render(
  <QueryClientProvider client={queryClient}>
    <StrictMode>
      <App />
    </StrictMode>
  </QueryClientProvider>,
);
```
