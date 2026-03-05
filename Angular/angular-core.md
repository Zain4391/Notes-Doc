# Angular Core Concepts

## Table of Contents
- [Dependency Injection (DI)](#dependency-injection-di)
- [Services](#services)
- [Guards](#guards)
- [Interceptors](#interceptors)
- [Observables](#observables)
- [Signals](#signals)

---

## Dependency Injection (DI)

Angular's DI system provides dependencies to classes instead of having classes create them. Angular maintains an **injector tree** that mirrors the component tree.

### How It Works

When a class (component, service, etc.) declares a dependency in its constructor, Angular's injector resolves and provides the correct instance.

```typescript
// Without DI — tight coupling
export class UserComponent {
  private service = new UserService(); // bad: class creates its own dep
}

// With DI — Angular injects it
export class UserComponent {
  constructor(private userService: UserService) {} // Angular provides it
}
```

### `@Injectable` Decorator

Mark a class as injectable so Angular's DI system can instantiate it.

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root', // singleton across the entire app
})
export class LoggerService {
  log(message: string): void {
    console.log(`[LOG]: ${message}`);
  }
}
```

### Provider Scopes

| `providedIn` value | Scope |
|--------------------|-------|
| `'root'`           | App-wide singleton (tree-shakable) |
| `'platform'`       | Shared across multiple apps on the same page |
| `'any'`            | Separate instance per lazy-loaded module |
| A specific module  | Scoped to that module |

### Providing at Component Level

```typescript
@Component({
  selector: 'app-cart',
  templateUrl: './cart.component.html',
  providers: [CartService], // new instance for this component subtree
})
export class CartComponent {
  constructor(private cartService: CartService) {}
}
```

### Injection Tokens

Use `InjectionToken` for non-class values (config objects, primitives).

```typescript
import { InjectionToken } from '@angular/core';

export const API_URL = new InjectionToken<string>('api.url');

// provide it
providers: [
  { provide: API_URL, useValue: 'https://api.example.com' }
]

// inject it
import { Inject } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(API_URL) private apiUrl: string) {}
}
```

---

## Services

Services are classes with a focused purpose — data fetching, business logic, state management — that are shared across components via DI.

### Basic Service

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({
  providedIn: 'root',
})
export class UserService {
  private readonly baseUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getAll(): Observable<User[]> {
    return this.http.get<User[]>(this.baseUrl);
  }

  getById(id: number): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/${id}`);
  }

  create(user: Omit<User, 'id'>): Observable<User> {
    return this.http.post<User>(this.baseUrl, user);
  }

  update(id: number, user: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/${id}`, user);
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

### Using the Service in a Component

```typescript
// users.component.ts
import { Component, OnInit } from '@angular/core';
import { UserService, User } from './user.service';

@Component({
  selector: 'app-users',
  template: `
    <ul>
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `,
})
export class UsersComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    this.userService.getAll().subscribe((data) => (this.users = data));
  }
}
```

### State Service (with BehaviorSubject)

A lightweight way to share state between components without a full state-management library.

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

export interface AuthState {
  token: string | null;
  username: string | null;
}

@Injectable({
  providedIn: 'root',
})
export class AuthService {
  private state = new BehaviorSubject<AuthState>({ token: null, username: null });

  readonly state$: Observable<AuthState> = this.state.asObservable();

  get isLoggedIn(): boolean {
    return !!this.state.getValue().token;
  }

  login(token: string, username: string): void {
    this.state.next({ token, username });
  }

  logout(): void {
    this.state.next({ token: null, username: null });
  }
}
```

---

## Guards

Guards control whether a route can be **activated**, **deactivated**, **loaded**, or **matched**. In Angular 14+ the preferred style is **functional guards** (plain functions).

### Route Guard Types

| Guard function | Purpose |
|---|---|
| `canActivate` | Allow/deny navigation to a route |
| `canActivateChild` | Allow/deny navigation to child routes |
| `canDeactivate` | Prompt user before leaving a route |
| `canMatch` | Allow/deny route matching (replaces `canLoad`) |

### `canActivate` — Auth Guard

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn) {
    return true;
  }

  // Redirect to login, preserving the intended URL
  return router.createUrlTree(['/login']);
};
```

### `canActivate` — Role Guard

```typescript
// role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, ActivatedRouteSnapshot, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const roleGuard: CanActivateFn = (route: ActivatedRouteSnapshot) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  const requiredRole = route.data['role'] as string;

  if (auth.hasRole(requiredRole)) {
    return true;
  }

  return router.createUrlTree(['/forbidden']);
};
```

### Applying Guards to Routes

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './auth.guard';
import { roleGuard } from './role.guard';

export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard],
  },
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [authGuard, roleGuard],
    data: { role: 'admin' },
  },
];
```

### `canDeactivate` — Unsaved Changes Guard

```typescript
// unsaved-changes.guard.ts
import { CanDeactivateFn } from '@angular/router';

export interface CanComponentDeactivate {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<CanComponentDeactivate> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('You have unsaved changes. Leave anyway?');
  }
  return true;
};
```

```typescript
// edit-profile.component.ts
import { CanComponentDeactivate } from './unsaved-changes.guard';

@Component({ selector: 'app-edit-profile', ... })
export class EditProfileComponent implements CanComponentDeactivate {
  isDirty = false;

  hasUnsavedChanges(): boolean {
    return this.isDirty;
  }
}
```

---

## Interceptors

HTTP interceptors sit in the `HttpClient` pipeline and let you transform requests and responses (add headers, handle errors, show loaders, cache, etc.).

In Angular 15+ the preferred style is **functional interceptors**.

### `HttpInterceptorFn` Signature

```typescript
type HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
) => Observable<HttpEvent<unknown>>;
```

### Auth Token Interceptor

Attaches a Bearer token to every outgoing request.

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.getToken();

  if (!token) {
    return next(req);
  }

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });

  return next(authReq);
};
```

### Error Handling Interceptor

Catches HTTP errors globally and redirects on 401.

```typescript
// error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (error.status) {
        case 401:
          router.navigate(['/login']);
          break;
        case 403:
          router.navigate(['/forbidden']);
          break;
        case 500:
          console.error('Server error', error.message);
          break;
      }
      return throwError(() => error);
    })
  );
};
```

### Loading Spinner Interceptor

Tracks in-flight requests to show/hide a global spinner.

```typescript
// loading.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs';
import { LoadingService } from './loading.service';

export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(LoadingService);
  loading.show();

  return next(req).pipe(finalize(() => loading.hide()));
};
```

```typescript
// loading.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoadingService {
  readonly isLoading = signal(false);

  show(): void { this.isLoading.set(true); }
  hide(): void { this.isLoading.set(false); }
}
```

### Registering Interceptors

```typescript
// app.config.ts  (standalone bootstrap)
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './auth.interceptor';
import { errorInterceptor } from './error.interceptor';
import { loadingInterceptor } from './loading.interceptor';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor, loadingInterceptor])
    ),
  ],
};
```

> Interceptors run in the order they are listed. Responses pass through in **reverse** order.

---

## Observables

Angular uses **RxJS Observables** as the primary async primitive for HTTP calls, router events, form value changes, and more. An Observable does nothing until subscribed to — it is *lazy* by nature.

### Creating Observables

```typescript
import { Observable, of, from, interval } from 'rxjs';

// Emit a static set of values
const numbers$ = of(1, 2, 3);

// Wrap a Promise
const fromPromise$ = from(fetch('/api/data'));

// Emit every second
const timer$ = interval(1000);

// Custom observable
const custom$ = new Observable<number>((subscriber) => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});
```

### Common Operators

```typescript
import { map, filter, switchMap, catchError, tap, debounceTime, distinctUntilChanged } from 'rxjs/operators';
import { of } from 'rxjs';

this.searchInput.valueChanges.pipe(
  debounceTime(300),              // wait 300 ms after last keystroke
  distinctUntilChanged(),         // skip if value didn't change
  filter((term) => term.length > 1),
  switchMap((term) =>             // cancel previous request, start new one
    this.searchService.search(term).pipe(
      catchError(() => of([]))    // swallow errors, return empty array
    )
  ),
  tap((results) => console.log('results', results)),
  map((results) => results.slice(0, 10))
);
```

### `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`

| Operator | Behaviour |
|---|---|
| `switchMap` | Cancels the previous inner observable when a new value arrives. Best for search/autocomplete. |
| `mergeMap` | Runs all inner observables concurrently. Best for parallel independent requests. |
| `concatMap` | Queues inner observables one after another. Best for ordered sequential tasks. |
| `exhaustMap` | Ignores new values while an inner observable is active. Best for submit buttons. |

### Subscribing and Unsubscribing

Memory leaks occur when subscriptions are not cleaned up. The two main patterns:

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({ selector: 'app-example', template: '' })
export class ExampleComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  constructor(private userService: UserService) {}

  ngOnInit(): void {
    // Pattern 1: takeUntil
    this.userService.getAll().pipe(
      takeUntil(this.destroy$)
    ).subscribe((users) => console.log(users));
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

```html
<!-- Pattern 2: async pipe (preferred in templates — auto-unsubscribes) -->
<ul>
  <li *ngFor="let user of users$ | async">{{ user.name }}</li>
</ul>
```

### Subject Types

| Type | Behaviour |
|---|---|
| `Subject` | No initial value. Only emits to current subscribers. |
| `BehaviorSubject(value)` | Holds the latest value. New subscribers receive it immediately. |
| `ReplaySubject(n)` | Replays the last *n* values to new subscribers. |
| `AsyncSubject` | Emits only the last value, and only on `complete()`. |

```typescript
import { BehaviorSubject } from 'rxjs';

const count$ = new BehaviorSubject<number>(0);

count$.subscribe((v) => console.log('A:', v)); // A: 0
count$.next(1);                               // A: 1
count$.subscribe((v) => console.log('B:', v)); // B: 1  <- gets latest immediately
count$.next(2);                               // A: 2, B: 2
```

---

## Signals

Signals (stable since Angular 17) are a **synchronous, fine-grained reactivity** primitive. Unlike Observables, they are always synchronous and always hold a current value — no subscription required.

### Creating and Reading Signals

```typescript
import { signal, computed, effect } from '@angular/core';

// Writable signal
const count = signal(0);

// Read the current value by calling it like a function
console.log(count()); // 0

// Update
count.set(5);
count.update((prev) => prev + 1); // 6

// Mutate objects/arrays in-place
const items = signal<string[]>([]);
items.mutate((list) => list.push('Angular'));
```

### `computed` — Derived Signals

Derived signals re-evaluate automatically when their dependencies change.

```typescript
const price = signal(100);
const quantity = signal(3);

const total = computed(() => price() * quantity());

console.log(total()); // 300
price.set(120);
console.log(total()); // 360 — automatically updated
```

### `effect` — Side Effects

Runs a callback whenever the signals it reads change. Use for DOM manipulation, logging, or syncing to `localStorage`.

```typescript
import { effect, signal } from '@angular/core';

const theme = signal<'light' | 'dark'>('light');

effect(() => {
  // Runs immediately, then again every time `theme` changes
  document.body.setAttribute('data-theme', theme());
});

theme.set('dark'); // effect re-runs automatically
```

> `effect()` must be called inside an injection context (constructor, `inject()` call, or `runInInjectionContext`).

### Signals in Components

```typescript
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <p>Count: {{ count() }}</p>
    <p>Double: {{ double() }}</p>
    <button (click)="increment()">+1</button>
    <button (click)="reset()">Reset</button>
  `,
})
export class CounterComponent {
  count = signal(0);
  double = computed(() => this.count() * 2);

  increment(): void { this.count.update((v) => v + 1); }
  reset(): void     { this.count.set(0); }
}
```

### `input()`, `output()`, and `model()` — Signal-Based Component API

Available from Angular 17.1+.

```typescript
import { Component, input, output, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  template: `
    <span *ngFor="let star of stars">
      <button (click)="select(star)">{{ star <= value() ? '★' : '☆' }}</button>
    </span>
  `,
})
export class RatingComponent {
  // Required input signal
  max = input.required<number>();

  // Two-way bindable model signal (replaces @Input + @Output pair)
  value = model<number>(0);

  // Output signal (replaces EventEmitter)
  ratingChange = output<number>();

  get stars(): number[] {
    return Array.from({ length: this.max() }, (_, i) => i + 1);
  }

  select(star: number): void {
    this.value.set(star);
    this.ratingChange.emit(star);
  }
}
```

### Interop: Observable ↔ Signal

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

// Observable → Signal (auto-unsubscribes with the component)
const tick = toSignal(interval(1000), { initialValue: 0 });
console.log(tick()); // current tick value

// Signal → Observable
const count = signal(0);
const count$ = toObservable(count);
count$.subscribe((v) => console.log(v));
```

### Observables vs Signals — Quick Comparison

| | Observable | Signal |
|---|---|---|
| **Nature** | Lazy stream (0…∞ values) | Eager holder of a single current value |
| **Sync/Async** | Both | Always synchronous |
| **Subscription** | Required (`subscribe` / `async` pipe) | None — just call `value()` |
| **Change Detection** | Zone.js or `markForCheck` | Fully integrated, zone-optional |
| **Best for** | HTTP, events, complex async flows | Local UI state, derived values, component I/O |
| **Interop** | `toSignal()` / `toObservable()` | same |
