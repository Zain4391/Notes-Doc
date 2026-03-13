# Angular Routing

Angular's `RouterModule` provides client-side navigation without full page reloads.

---

## Table of Contents

1. [Setup](#setup)
2. [Defining Routes](#defining-routes)
3. [RouterLink & RouterLinkActive](#routerlink--routerlinkactive)
4. [Route Parameters](#route-parameters)
5. [Query Parameters](#query-parameters)
6. [Fragment (Hash)](#fragment-hash)
7. [Nested / Child Routes](#nested--child-routes)
8. [Lazy Loading](#lazy-loading)
9. [Route Guards](#route-guards)
10. [Resolvers](#resolvers)
11. [Wildcard & Redirect Routes](#wildcard--redirect-routes)
12. [Programmatic Navigation](#programmatic-navigation)
13. [Router Events](#router-events)

---

## Setup

```bash
ng new my-app --routing   # generates AppRoutingModule automatically
```

Import `RouterModule` with routes in `AppRoutingModule`:

```ts
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

Add the router outlet in `app.component.html`:

```html
<router-outlet></router-outlet>
```

---

## Defining Routes

| Property | Purpose |
|---|---|
| `path` | URL segment (no leading `/`) |
| `component` | Component to render |
| `redirectTo` | Redirect to another path |
| `pathMatch` | `'full'` or `'prefix'` matching |
| `children` | Nested child routes |
| `loadChildren` | Lazy-loaded module/route |
| `canActivate` | Guard array |
| `resolve` | Resolver map |
| `data` | Static data attached to route |
| `title` | Page title (Angular 14+) |

---

## RouterLink & RouterLinkActive

```html
<!-- Static link -->
<a routerLink="/about">About</a>

<!-- Dynamic link using array syntax -->
<a [routerLink]="['/users', userId]">Profile</a>

<!-- Active class -->
<a routerLink="/home" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">
  Home
</a>
```

---

## Route Parameters

Route params are **required** segments embedded in the path.

### Define

```ts
{ path: 'users/:id', component: UserDetailComponent }
```

### Read (Snapshot — static)

```ts
import { ActivatedRoute } from '@angular/router';

constructor(private route: ActivatedRoute) {}

ngOnInit() {
  const id = this.route.snapshot.paramMap.get('id'); // string | null
}
```

### Read (Observable — reacts to param changes without re-creating the component)

```ts
ngOnInit() {
  this.route.paramMap.subscribe(params => {
    const id = params.get('id');
  });
}
```

### Multiple Params

```ts
{ path: 'categories/:catId/products/:prodId', component: ProductComponent }
```

```ts
const catId  = this.route.snapshot.paramMap.get('catId');
const prodId = this.route.snapshot.paramMap.get('prodId');
```

---

## Query Parameters

Query params are **optional** key-value pairs appended after `?`.

### Navigate with query params

```ts
// Programmatically
this.router.navigate(['/search'], { queryParams: { q: 'angular', page: 2 } });
// → /search?q=angular&page=2
```

```html
<!-- Template -->
<a [routerLink]="['/search']" [queryParams]="{ q: 'angular', page: 2 }">Search</a>
```

### Preserve / Merge query params across navigation

```ts
this.router.navigate(['/search'], {
  queryParams:         { sort: 'asc' },
  queryParamsHandling: 'merge',   // 'preserve' | 'merge' | ''
});
```

### Read query params

```ts
// Snapshot
const q = this.route.snapshot.queryParamMap.get('q');

// Observable
this.route.queryParamMap.subscribe(params => {
  const q    = params.get('q');
  const page = params.get('page');
});
```

---

## Fragment (Hash)

```ts
this.router.navigate(['/docs'], { fragment: 'installation' });
// → /docs#installation
```

```ts
this.route.fragment.subscribe(f => console.log(f)); // 'installation'
```

---

## Nested / Child Routes

```ts
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: '',         component: DashboardHomeComponent },
      { path: 'stats',    component: StatsComponent },
      { path: 'settings', component: SettingsComponent },
    ],
  },
];
```

Add a `<router-outlet>` inside `DashboardComponent`'s template for children to render into.

---

## Lazy Loading

Lazy loading splits the bundle, loading a module only when its route is visited.

```ts
// app-routing.module.ts
{
  path: 'admin',
  loadChildren: () =>
    import('./admin/admin.module').then(m => m.AdminModule),
}
```

```ts
// admin-routing.module.ts  (uses forChild)
RouterModule.forChild(adminRoutes)
```

### Standalone API (Angular 15+)

```ts
{
  path: 'admin',
  loadComponent: () =>
    import('./admin/admin.component').then(m => m.AdminComponent),
}
```

---

## Route Guards

Guards decide whether to allow or block navigation.

| Guard | Interface | Use case |
|---|---|---|
| `canActivate` | `CanActivateFn` | Protect route from access |
| `canActivateChild` | `CanActivateChildFn` | Protect child routes |
| `canDeactivate` | `CanDeactivateFn` | Warn on unsaved changes |
| `canMatch` | `CanMatchFn` | Lazy-load conditionally |

### Functional Guard (Angular 15+, preferred)

```ts
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth   = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.createUrlTree(['/login']);
};
```

```ts
{ path: 'profile', component: ProfileComponent, canActivate: [authGuard] }
```

### CanDeactivate example

```ts
export const unsavedChangesGuard: CanDeactivateFn<EditComponent> =
  (component) => component.hasUnsavedChanges()
    ? confirm('Discard unsaved changes?')
    : true;
```

---

## Resolvers

Resolvers pre-fetch data **before** the route activates.

```ts
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from './user.service';
import { User } from './user.model';

export const userResolver: ResolveFn<User> = (route) => {
  return inject(UserService).getUser(route.paramMap.get('id')!);
};
```

```ts
{
  path: 'users/:id',
  component: UserDetailComponent,
  resolve: { user: userResolver },
}
```

```ts
// Inside UserDetailComponent
ngOnInit() {
  const user = this.route.snapshot.data['user'];
}
```

---

## Wildcard & Redirect Routes

```ts
const routes: Routes = [
  { path: '',       redirectTo: '/home', pathMatch: 'full' },
  { path: 'home',   component: HomeComponent },
  { path: '**',     component: NotFoundComponent },  // must be last
];
```

> **Tip:** `pathMatch: 'full'` is required on empty-string redirects to avoid matching every route.

---

## Programmatic Navigation

```ts
import { Router } from '@angular/router';

constructor(private router: Router) {}

// Basic
this.router.navigate(['/home']);

// With route param
this.router.navigate(['/users', 42]);

// With query params & fragment
this.router.navigate(['/users', 42], {
  queryParams: { tab: 'info' },
  fragment:    'section1',
});

// Relative navigation
this.router.navigate(['../sibling'], { relativeTo: this.route });

// Navigate by URL string
this.router.navigateByUrl('/home?ref=email#top');
```

---

## Router Events

Tap into the router's lifecycle to show loading spinners, track analytics, etc.

```ts
import { Router, NavigationStart, NavigationEnd } from '@angular/router';

constructor(private router: Router) {
  this.router.events.subscribe(event => {
    if (event instanceof NavigationStart) {
      // show spinner
    }
    if (event instanceof NavigationEnd) {
      // hide spinner, log page view
    }
  });
}
```

Common events (in order): `NavigationStart` → `RoutesRecognized` → `GuardsCheckStart` → `GuardsCheckEnd` → `ResolveStart` → `ResolveEnd` → `NavigationEnd`.

---

## Quick Reference

```ts
// Read snapshot values
this.route.snapshot.paramMap.get('id')
this.route.snapshot.queryParamMap.get('q')
this.route.snapshot.data['key']
this.route.snapshot.fragment

// Observable versions
this.route.paramMap.subscribe(...)
this.route.queryParamMap.subscribe(...)
this.route.data.subscribe(...)
this.route.fragment.subscribe(...)
```
