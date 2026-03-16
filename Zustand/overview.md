# Zustand — State Management

This project uses Zustand v5 for client-side global state. Three stores cover auth, cart, and UI concerns. Two of them are persisted to localStorage.

---

## Why Zustand over Context or Redux

- No boilerplate — no actions, reducers, or providers per store.
- Subscriptions are fine-grained — components only re-render when the specific slice they use changes.
- Works outside React — can call `useAuthStore.getState()` in plain functions.
- Built-in persistence middleware for localStorage.

---

## Creating a Store

```ts
import { create } from "zustand";

export const useUIStore = create<UIState>()((set) => ({
  isLoading: false,
  isSidebarOpen: false,
  activeModal: null,

  setLoading: (loading: boolean) => set({ isLoading: loading }),
  toggleSidebar: () =>
    set((state) => ({ isSidebarOpen: !state.isSidebarOpen })),
  openModal: (modalId: string) => set({ activeModal: modalId }),
  closeModal: () => set({ activeModal: null }),
}));
```

`set` is the only primitive you need. Pass it an object (partial update) or a function that receives the previous state.

---

## Persistence Middleware

The `persist` middleware serialises state to localStorage automatically.

```ts
// store/auth.store.ts
export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,

      setAuth: (user: AuthUser, accessToken: string) =>
        set({ user, accessToken, isAuthenticated: true }),

      clearAuth: () =>
        set({ user: null, accessToken: null, isAuthenticated: false }),

      updateUser: (partial: Partial<AuthUser>) => {
        const current = get().user;
        if (!current) return;
        set({ user: { ...current, ...partial } });
      },
    }),
    {
      name: "food-delivery-auth", // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({
        // only persist these fields
        user: state.user,
        accessToken: state.accessToken,
        isAuthenticated: state.isAuthenticated,
      }),
    },
  ),
);
```

`partialize` lets you choose which fields are persisted. Actions (functions) are never persisted — only data.

---

## Fine-Grained Selector Hooks

Exporting named selector hooks from the store file keeps component code clean and avoids unnecessary re-renders.

```ts
// store/auth.store.ts — selectors at the bottom of the file
export const useUser = () => useAuthStore((state) => state.user);
export const useAccessToken = () => useAuthStore((state) => state.accessToken);
export const useIsAuthenticated = () =>
  useAuthStore((state) => state.isAuthenticated);
export const useUserRole = () => useAuthStore((state) => state.user?.role);

// Role-specific boolean selectors
export const useIsCustomer = () =>
  useAuthStore((state) => state.user?.role === ROLES.CUSTOMER);
export const useIsDriver = () =>
  useAuthStore((state) => state.user?.role === ROLES.DRIVER);
export const useIsAdmin = () =>
  useAuthStore((state) => state.user?.role === ROLES.ADMIN);
```

A component that calls `useIsCustomer()` only re-renders when `user.role` changes — not when any other part of the auth store changes.

---

## The Cart Store

The cart store shows a more complex example with derived calculations and cross-restaurant logic.

```ts
// store/cart.store.ts (key parts)
export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],
      restaurantId: null,

      addItem: (item: cartItem) => {
        const { items, restaurantId } = get();

        // If adding from a different restaurant, clear the cart first
        if (restaurantId && restaurantId !== item.restaurant_id) {
          set({
            items: [{ ...item, quantity: 1 }],
            restaurantId: item.restaurant_id,
          });
          return;
        }

        // Increment quantity if item already in cart
        const existing = items.find(
          (i) => i.menu_item_id === item.menu_item_id,
        );
        if (existing) {
          set({
            items: items.map((i) =>
              i.menu_item_id === item.menu_item_id
                ? { ...i, quantity: i.quantity + 1 }
                : i,
            ),
          });
          return;
        }

        // Otherwise add as new item
        set({
          items: [...items, { ...item, quantity: 1 }],
          restaurantId: item.restaurant_id,
        });
      },

      getTotalAmount: () =>
        get().items.reduce((acc, item) => acc + item.price * item.quantity, 0),
    }),
    {
      name: "food-delivery-cart",
      storage: createJSONStorage(() => localStorage),
    },
  ),
);

// Selectors
export const useCartItems = () => useCartStore((state) => state.items);
export const useCartTotal = () =>
  useCartStore((state) => state.getTotalAmount());
export const useCartItemCount = () =>
  useCartStore((state) => state.getTotalItems());
```

Key pattern: `get()` inside actions reads the current state synchronously. Use it when the new state depends on the previous state.

---

## Stores Overview

| Store        | Persisted          | Contents                                           |
| ------------ | ------------------ | -------------------------------------------------- |
| `auth.store` | Yes (localStorage) | user, accessToken, isAuthenticated, role selectors |
| `cart.store` | Yes (localStorage) | items, restaurantId, quantity logic, totals        |
| `ui.store`   | No                 | isLoading, isSidebarOpen, activeModal              |

UI state is not persisted because loading states and modals should not survive a page refresh.

---
