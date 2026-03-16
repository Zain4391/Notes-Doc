# Zod + React Hook Form

This project uses Zod for schema-first validation and React Hook Form for form state management. They are connected via `@hookform/resolvers/zod`.

---

## Why This Combination

- Zod defines the shape and rules of data — one source of truth.
- React Hook Form handles form registration, submission, and error display — no re-renders on every keystroke.
- `z.infer<typeof schema>` generates the TypeScript type from the schema — no duplicate type definitions.

---

## Defining a Schema

```ts
// schemas/auth.schema.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z.email("Invalid email address"),
  password: z.string().min(1, "Password is required"),
});
```

For registration with a cross-field validation (password confirmation):

```ts
export const registerCustomerSchema = z
  .object({
    name: z.string().min(1, "Name is required"),
    email: z.email("Invalid email"),
    password: z.string().min(8, "Password must be at least 8 characters"),
    confirmPassword: z.string().min(1, "Please confirm your password"),
    address: z.string().min(1, "Address is required"),
    role: z.enum(ROLES).optional(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    error: "Passwords do not match",
    path: ["confirmPassword"],
  });
```

`.refine()` adds cross-field validation. The `path` field tells React Hook Form which input to attach the error to.

---

## Inferring Types from Schemas

```ts
// schemas/auth.schema.ts
export type LoginFormValues = z.infer<typeof loginSchema>;
export type RegisterCustomerFormValues = z.infer<typeof registerCustomerSchema>;
export type RegisterDriverFormValues = z.infer<typeof registerDriverSchema>;
```

These types are used in hooks and service files. If the schema changes, the types automatically update — no manual sync needed.

---

## Connecting to React Hook Form

```tsx
// app/login/page.tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { loginSchema, LoginFormValues } from "@/schemas/auth.schema";

export default function LoginPage() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema), // Zod handles all validation
  });

  const onSubmit = (data: LoginFormValues) => {
    // data is fully typed and already validated
    login(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} />
      {errors.email && <p>{errors.email.message}</p>}

      <input {...register("password")} type="password" />
      {errors.password && <p>{errors.password.message}</p>}

      <button type="submit">Sign In</button>
    </form>
  );
}
```

`handleSubmit` only calls `onSubmit` if Zod validation passes. `errors` contains per-field messages from the schema.

---

## Stripping UI-Only Fields Before Sending

`confirmPassword` exists in the form but should not be sent to the API. Destructure it out before passing the payload to the service:

```ts
// hooks/useRegister.ts
mutationFn: (values: RegisterCustomerFormValues) => {
  const { confirmPassword, ...payload } = values   // strip confirmPassword
  return authService.registerCustomer(payload)
},
```

---

## Driver Schema — Enum Validation

```ts
export const registerDriverSchema = z.object({
  // ...
  phone: z
    .string()
    .min(11, "Phone number must be 11 digits")
    .max(11, "Phone number must be 11 digits"),

  vehicle_type: z.enum(VEHICLE_TYPE, {
    message: "Invalid vehicle type", // shown if value is not 'car' or 'bike'
  }),
});
```

`z.enum()` validates against a TypeScript enum's values. The error message appears in `errors.vehicle_type.message`.

---

## Summary

```
Schema (Zod)
  → inferred type used in hooks and services
      → resolver passed to useForm()
          → handleSubmit only fires when schema passes
              → errors object maps field names to messages
```

One schema definition drives: validation rules, TypeScript types, and form error messages.

---
