# Angular Services & HttpClient

## Table of Contents
- [What is a Service and Why Use One?](#what-is-a-service-and-why-use-one)
- [Setting Up HttpClient](#setting-up-httpclient)
- [Defining a Typed Model](#defining-a-typed-model)
- [Creating an HTTP Service](#creating-an-http-service)
- [HTTP Methods](#http-methods)
- [Using the Service in a Component](#using-the-service-in-a-component)
- [Handling Errors](#handling-errors)
- [Query Params and Headers](#query-params-and-headers)
- [Key Patterns to Internalize](#key-patterns-to-internalize)
- [Role-Based UI with Signals](#role-based-ui-with-signals)

---

## What is a Service and Why Use One?

In Angular, **services hold your data-fetching and business logic**. Components should never call HTTP directly — they delegate to a service. This keeps components focused on display only and makes the logic reusable and testable.

```
Component  →  calls method on Service  →  Service calls HttpClient  →  returns Observable
Component  ←  subscribes / uses async pipe  ←  data arrives
```

---

## Setting Up HttpClient

In a standalone Angular app (`app.config.ts`):

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(),
  ],
};
```

> If you are using interceptors, add `withInterceptors([...])` inside `provideHttpClient()`.

---

## Defining a Typed Model

Always type your API responses to get full IDE support and catch shape mismatches at compile time.

```typescript
// models/post.model.ts
export interface Post {
  id: number;
  title: string;
  body: string;
  userId: number;
}

export interface CreatePostDto {
  title: string;
  body: string;
  userId: number;
}
```

---

## Creating an HTTP Service

```typescript
// services/post.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Post, CreatePostDto } from '../models/post.model';

@Injectable({
  providedIn: 'root',
})
export class PostService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = 'https://jsonplaceholder.typicode.com/posts';

  getAll(): Observable<Post[]> {
    return this.http.get<Post[]>(this.baseUrl);
  }

  getById(id: number): Observable<Post> {
    return this.http.get<Post>(`${this.baseUrl}/${id}`);
  }

  getByUser(userId: number): Observable<Post[]> {
    const params = new HttpParams().set('userId', userId);
    return this.http.get<Post[]>(this.baseUrl, { params });
  }

  create(dto: CreatePostDto): Observable<Post> {
    return this.http.post<Post>(this.baseUrl, dto);
  }

  update(id: number, dto: Partial<CreatePostDto>): Observable<Post> {
    return this.http.patch<Post>(`${this.baseUrl}/${id}`, dto);
  }

  replace(id: number, dto: CreatePostDto): Observable<Post> {
    return this.http.put<Post>(`${this.baseUrl}/${id}`, dto);
  }

  delete(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

---

## HTTP Methods

Every `http.get()`, `http.post()`, etc. returns an **Observable** — not the data itself, but a stream that emits the data when the response arrives. You subscribe to it to get the value.

| Method | Purpose | Body |
|---|---|---|
| `GET` | Fetch one or many resources | None |
| `POST` | Create a new resource | Required |
| `PUT` | Fully replace a resource | Required |
| `PATCH` | Partially update a resource | Required |
| `DELETE` | Remove a resource | None |

### GET — Fetch a list

```typescript
this.http.get<Post[]>('https://api.example.com/posts');
```

### GET — Fetch a single item

```typescript
this.http.get<Post>('https://api.example.com/posts/1');
```

### POST — Create

```typescript
const newPost: CreatePostDto = { title: 'Hello', body: 'World', userId: 1 };
this.http.post<Post>('https://api.example.com/posts', newPost);
```

### PATCH — Partial Update

```typescript
this.http.patch<Post>('https://api.example.com/posts/1', { title: 'Updated' });
```

### PUT — Full Replace

```typescript
this.http.put<Post>('https://api.example.com/posts/1', { title: 'New', body: 'Body', userId: 1 });
```

### DELETE

```typescript
this.http.delete<void>('https://api.example.com/posts/1');
```

---

## Using the Service in a Component

The modern Angular pattern uses `inject()` instead of constructor injection and `signal()` for reactive local state.

```typescript
// posts.component.ts
import { Component, inject, signal, OnInit } from '@angular/core';
import { PostService } from '../services/post.service';
import { Post } from '../models/post.model';

@Component({
  selector: 'app-posts',
  templateUrl: './posts.component.html',
})
export class PostsComponent implements OnInit {
  private postService = inject(PostService);

  // signal<Type>(initialValue) — reactive state, setting it re-renders the template
  posts    = signal<Post[]>([]);
  selected = signal<Post | null>(null);
  loading  = signal(false);
  error    = signal<string | null>(null);

  ngOnInit(): void {
    this.loadPosts();
  }

  loadPosts(): void {
    this.loading.set(true);
    this.error.set(null);

    // .subscribe({ next, error }) — unwrap the Observable, handle success and failure
    this.postService.getAll().subscribe({
      next:  (data)  => { this.posts.set(data); this.loading.set(false); },
      error: (err)   => { this.error.set(err.message); this.loading.set(false); },
    });
  }

  selectPost(id: number): void {
    this.postService.getById(id).subscribe({
      next: (post) => this.selected.set(post),
    });
  }

  createPost(): void {
    const dto = { title: 'New Post', body: 'Content here', userId: 1 };
    this.postService.create(dto).subscribe({
      next: (post) => this.posts.update((list) => [...list, post]),
    });
  }

  updatePost(id: number): void {
    this.postService.update(id, { title: 'Updated Title' }).subscribe({
      next: (updated) =>
        this.posts.update((list) =>
          list.map((p) => (p.id === updated.id ? updated : p))
        ),
    });
  }

  deletePost(id: number): void {
    this.postService.delete(id).subscribe({
      next: () => this.posts.update((list) => list.filter((p) => p.id !== id)),
    });
  }
}
```

### Template with Modern Control Flow

```html
<!-- posts.component.html -->
@if (loading()) {
  <p>Loading…</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else {
  <!-- @for requires track — always use a unique identifier -->
  @for (post of posts(); track post.id) {
    <div class="post-card">
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
      <button (click)="updatePost(post.id)">Edit</button>
      <button (click)="deletePost(post.id)">Delete</button>
    </div>
  } @empty {
    <p>No posts found.</p>
  }
}
```

> `@if` / `@else` / `@empty` are the modern control flow syntax — no `*ngIf` or `*ngFor` needed.

---

## Handling Errors

### In the component with `subscribe`

```typescript
this.postService.getAll().subscribe({
  next:     (data) => this.posts.set(data),
  error:    (err: HttpErrorResponse) => this.error.set(`Error ${err.status}: ${err.message}`),
  complete: ()     => this.loading.set(false),
});
```

### In the service with `catchError`

Handle the error once in the service so every consumer gets a clean fallback.

```typescript
import { catchError, throwError } from 'rxjs';
import { HttpErrorResponse } from '@angular/common/http';

getAll(): Observable<Post[]> {
  return this.http.get<Post[]>(this.baseUrl).pipe(
    catchError((err: HttpErrorResponse) => {
      console.error('PostService.getAll failed', err);
      return throwError(() => new Error(`Failed to load posts (${err.status})`));
    })
  );
}
```

### Global error handling with an interceptor

For app-wide 401/403/500 handling see the Interceptors section in `angular-core.md`.

---

## Query Params and Headers

### Query Parameters

```typescript
import { HttpParams } from '@angular/common/http';

// Builds ?page=2&limit=20&sort=title
const params = new HttpParams()
  .set('page', 2)
  .set('limit', 20)
  .set('sort', 'title');

this.http.get<Post[]>(this.baseUrl, { params });
```

### Custom Headers

```typescript
import { HttpHeaders } from '@angular/common/http';

const headers = new HttpHeaders({
  'Content-Type': 'application/json',
  'X-Custom-Header': 'my-value',
});

this.http.get<Post[]>(this.baseUrl, { headers });
```

### Full Response (status code, headers)

```typescript
this.http.get<Post[]>(this.baseUrl, { observe: 'response' }).subscribe((resp) => {
  console.log(resp.status);       // 200
  console.log(resp.headers.get('X-Total-Count'));
  console.log(resp.body);         // Post[]
});
```

---

## Key Patterns to Internalize

These are the patterns you will repeat in almost every Angular feature:

| Pattern | What it does |
|---|---|
| `inject(SomeService)` | Gets the service — no constructor needed |
| `signal<Type[]>([])` | Reactive state — setting it automatically re-renders the template |
| `.subscribe({ next, error })` | Unwraps the Observable, handles success and failure |
| `@for (item of items(); track item.id)` | Loops over a signal's value — `track` is always required |
| `@if` / `@else` / `@empty` | Modern control flow, replaces `*ngIf` / `*ngFor` |
| `pipe(catchError(...))` | Handles errors inside the service before they reach the component |
| `signal.update(fn)` | Immutably derives the next state from the previous value |

---

## Role-Based UI with Signals

Role-based UI is just a `computed()` reading from `AuthService` — `isAdmin()`, `isTeacher()` — wrapping sections in `@if`.

### AuthService

```typescript
// services/auth.service.ts
import { Injectable, signal, computed } from '@angular/core';

export type Role = 'admin' | 'teacher' | 'student' | null;

@Injectable({ providedIn: 'root' })
export class AuthService {
  private role = signal<Role>(null);

  readonly isAdmin    = computed(() => this.role() === 'admin');
  readonly isTeacher  = computed(() => this.role() === 'teacher');
  readonly isLoggedIn = computed(() => this.role() !== null);

  login(role: Role): void  { this.role.set(role); }
  logout(): void           { this.role.set(null); }
}
```

### Component reading roles

```typescript
// dashboard.component.ts
import { Component, inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

@Component({
  selector: 'app-dashboard',
  templateUrl: './dashboard.component.html',
})
export class DashboardComponent {
  auth = inject(AuthService);
}
```

### Template with conditional sections

```html
<!-- dashboard.component.html -->
@if (auth.isLoggedIn()) {
  <nav>
    <a routerLink="/home">Home</a>

    @if (auth.isAdmin()) {
      <a routerLink="/admin">Admin Panel</a>
      <a routerLink="/users">Manage Users</a>
    }

    @if (auth.isTeacher()) {
      <a routerLink="/courses">My Courses</a>
      <a routerLink="/grades">Grade Book</a>
    }
  </nav>
} @else {
  <a routerLink="/login">Please log in</a>
}
```
