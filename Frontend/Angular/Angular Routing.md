---
title: Angular Routing
phase: fundamentals
topic: routing
stack: angular
difficulty: beginner-to-intermediate
tags:
  - angular
  - routing
  - spa
  - navigation
  - guards
  - lazy-loading
  - frontend
related:
  - "[[Angular Fundamentals]]"
  - "[[Components and Templates]]"
  - "[[React Routing]]"
  - "[[React vs Angular Comparison]]"
created: 2025-07-14
---

# Angular Routing

> [!quote] 💡
> *"Routes in Angular are like road signs for your SPA — they tell the browser which component to display without ever leaving the page."*

> [!info] 🎯 Why This Matters for a Java Developer
> If you've built REST APIs with Spring Boot, you already understand URL-to-handler mapping.
> Angular Routing is the **frontend equivalent** of `@RequestMapping` — except instead of mapping
> URLs to controller methods, you map URLs to **components**.

---

## Navigation — Where Are We Going?

| Phase | Topic | Spring Boot Parallel |
|-------|-------|---------------------|
| 1 | What is Routing? | `@RequestMapping` concept |
| 2 | Setting Up Routes | Route configuration |
| 3 | Router Outlet & Navigation | DispatcherServlet + links |
| 4 | Route Parameters | `@PathVariable` / `@RequestParam` |
| 5 | Nested Routes & Layouts | Nested controller mappings |
| 6 | Route Guards | Spring Security filters |
| 7 | Lazy Loading | On-demand module loading |
| 8 | Angular vs React Routing | Framework comparison |

---

## Phase 1: What is Routing?

> [!quote]
> *"Imagine a warehouse with many departments. Routing is the signage system that directs you
> to Shipping, Receiving, or Inventory — without making you leave the building and come back."*

### The Problem Routing Solves

In a traditional multi-page app (like a Spring MVC + Thymeleaf app), every page navigation
triggers a **full HTTP request** to the server, which returns a brand new HTML page:

```
┌──────────┐    GET /shipments     ┌──────────┐
│ Browser  │ ───────────────────►  │  Server  │
│          │ ◄───────────────────  │          │
│ Full     │   Full HTML page      │ Spring   │
│ Reload   │                       │ Boot     │
└──────────┘    GET /carriers      └──────────┘
│ Browser  │ ───────────────────►  │  Server  │
│          │ ◄───────────────────  │          │
│ Full     │   Full HTML page      │          │
│ Reload   │                       │          │
└──────────┘                       └──────────┘
```

In a **Single Page Application (SPA)**, the browser loads the app **once**. After that,
navigation happens **client-side** — Angular swaps out components without a full reload:

```
┌──────────────────────────────────────────────────┐
│                   BROWSER (SPA)                  │
│                                                  │
│  URL: /shipments                                 │
│  ┌──────────────────────────────────────┐        │
│  │         ShipmentListComponent        │        │
│  └──────────────────────────────────────┘        │
│                    │                             │
│          User clicks "Carriers"                  │
│                    ▼                             │
│  URL: /carriers  (NO full page reload!)          │
│  ┌──────────────────────────────────────┐        │
│  │         CarrierListComponent         │        │
│  └──────────────────────────────────────┘        │
│                                                  │
│  Only the component swaps. Header/footer stay.   │
└──────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description | Spring Boot Parallel |
|---------|-------------|---------------------|
| Route | A URL-to-component mapping | `@GetMapping("/path")` |
| Router | The service that manages navigation | `DispatcherServlet` |
| RouterOutlet | Where the routed component renders | View resolver target |
| RouterLink | Declarative navigation in templates | `<a href="...">` in Thymeleaf |
| Route Guard | Middleware that protects routes | `SecurityFilterChain` |

### Angular vs React — Routing is Built-in

- ✅ **Angular**: Router is built into `@angular/router` — ships with the framework
- ❌ **React**: No built-in router — you must install `react-router-dom` separately
- ✅ **Angular**: Route guards, lazy loading, resolvers are first-class features
- ❌ **React**: Guards must be hand-coded with wrapper components like `<ProtectedRoute>`

> [!tip] Spring Developer Mental Model
> Think of Angular's Router as a **frontend DispatcherServlet**.
> It intercepts URL changes, matches them against route definitions, and renders the
> appropriate component — just like `DispatcherServlet` dispatches to `@Controller` methods.

---

## Phase 2: Setting Up Routes

> [!quote]
> *"A route table is like a shipping manifest — it tells the system exactly where
> each package (URL) should be delivered (which component)."*

### The Route Configuration File

In a modern Angular app (v17+), routes are defined in a standalone `app.routes.ts` file:

```typescript
// app.routes.ts
// This is the Angular equivalent of your Spring @RequestMapping definitions.
// Each object maps a URL path to a component.

import { Routes } from '@angular/router';
import { DashboardComponent } from './dashboard/dashboard.component';
import { ShipmentListComponent } from './shipments/shipment-list.component';
import { ShipmentDetailComponent } from './shipments/shipment-detail.component';
import { CarrierListComponent } from './carriers/carrier-list.component';
import { NotFoundComponent } from './shared/not-found.component';

export const routes: Routes = [
  // Root path — like @GetMapping("/")
  { path: '', component: DashboardComponent },

  // Static path — like @GetMapping("/shipments")
  { path: 'shipments', component: ShipmentListComponent },

  // Path with parameter — like @GetMapping("/shipments/{id}")
  { path: 'shipments/:id', component: ShipmentDetailComponent },

  // Another static path
  { path: 'carriers', component: CarrierListComponent },

  // Wildcard catch-all — like a @ControllerAdvice for 404s
  // MUST be last — Angular matches routes top-to-bottom (first match wins)
  { path: '**', component: NotFoundComponent }
];
```

### Registering Routes in the App

```typescript
// app.config.ts
// This is where you "boot" the router — like SpringApplication.run() for routing

import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes)  // Register all routes with the application
  ]
};
```

### Side-by-Side Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPRING BOOT (Backend)                        │
│                                                                 │
│  @RestController                                                │
│  @RequestMapping("/shipments")                                  │
│  public class ShipmentController {                              │
│                                                                 │
│      @GetMapping          → List all shipments                  │
│      @GetMapping("/{id}") → Get one shipment                   │
│  }                                                              │
├─────────────────────────────────────────────────────────────────┤
│                    ANGULAR (Frontend)                           │
│                                                                 │
│  export const routes: Routes = [                                │
│    { path: 'shipments',     component: ShipmentListComponent }, │
│    { path: 'shipments/:id', component: ShipmentDetailComponent }│
│  ];                                                             │
│                                                                 │
│  // :id in Angular = {id} in Spring                             │
│  // component = the "view" that handles this route              │
└─────────────────────────────────────────────────────────────────┘
```

> [!warning] Route Order Matters!
> Angular matches routes **top to bottom** (first match wins).
> Always put more specific routes before less specific ones.
> The wildcard `**` route must be **last** — it catches everything.
>
> This is similar to how Spring Security filter chains are evaluated in order.

---

## Phase 3: Router Outlet and Navigation

> [!quote]
> *"The `<router-outlet>` is like a loading dock — it's the designated spot where
> the right cargo (component) gets unloaded based on the delivery address (URL)."*

### Router Outlet — The Component Placeholder

The `<router-outlet>` directive is a placeholder in your template. Angular replaces it
with the component that matches the current URL:

```html
<!-- app.component.html -->
<!-- This is the main layout — like a Thymeleaf layout template -->

<header>
  <nav class="main-nav">
    <a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{exact: true}">
      Dashboard
    </a>
    <a routerLink="/shipments" routerLinkActive="active">
      Shipments
    </a>
    <a routerLink="/carriers" routerLinkActive="active">
      Carriers
    </a>
  </nav>
</header>

<main>
  <!-- This is where the routed component renders -->
  <!-- Think of it as a "slot" that gets filled based on the URL -->
  <router-outlet></router-outlet>
</main>

<footer>
  <p>&copy; 2025 Logistics Platform</p>
</footer>
```

### How It Works Visually

```
URL: /shipments

┌────────────────────────────────────────┐
│              <header>                  │
│   Dashboard | [Shipments] | Carriers   │  ← nav stays
├────────────────────────────────────────┤
│                                        │
│         <router-outlet>                │
│    ┌──────────────────────────┐        │
│    │  ShipmentListComponent   │        │  ← swapped in
│    │  (rendered here)         │        │
│    └──────────────────────────┘        │
│                                        │
├────────────────────────────────────────┤
│              <footer>                  │  ← footer stays
└────────────────────────────────────────┘
```

### Navigation Methods

#### 1. Declarative Navigation — `routerLink`

```html
<!-- Static link — like <a href="/shipments"> but without page reload -->
<a routerLink="/shipments">View Shipments</a>

<!-- Dynamic link with parameter -->
<a [routerLink]="['/shipments', shipment.id]">View Details</a>

<!-- With query parameters — like ?status=pending&page=1 -->
<a [routerLink]="['/shipments']"
   [queryParams]="{ status: 'pending', page: 1 }">
  Pending Shipments
</a>

<!-- Highlight active link -->
<a routerLink="/shipments" routerLinkActive="active-link">
  Shipments
</a>
```

#### 2. Programmatic Navigation — `Router.navigate()`

```typescript
// shipment-list.component.ts
// Programmatic navigation — like using RedirectView in Spring

import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-shipment-list',
  template: `
    <ul>
      <li *ngFor="let s of shipments" (click)="viewDetail(s.id)">
        {{ s.trackingId }} — {{ s.origin }} → {{ s.destination }}
      </li>
    </ul>
  `
})
export class ShipmentListComponent {
  private router = inject(Router);
  shipments = [/* ... */];

  viewDetail(id: string) {
    // Navigate to /shipments/SHP-001
    // Similar to: return "redirect:/shipments/" + id; in Spring MVC
    this.router.navigate(['/shipments', id]);
  }

  searchWithFilters(status: string) {
    // Navigate with query params: /shipments?status=pending
    this.router.navigate(['/shipments'], {
      queryParams: { status: status }
    });
  }
}
```

> [!tip] `routerLink` vs `Router.navigate()`
> - Use `routerLink` in templates for **static links** (user clicks a link)
> - Use `Router.navigate()` in TypeScript for **programmatic navigation** (after form submit, etc.)
> - This is like `<a th:href="...">` vs `return "redirect:..."` in Spring MVC

---

## Phase 4: Route Parameters

> [!quote]
> *"Route parameters are like the tracking number on a parcel —
> they identify exactly which shipment you're looking at."*

### Path Parameters — `:id`

Path parameters are embedded in the URL path itself. In Spring Boot, you'd use
`@PathVariable`. In Angular, you use `ActivatedRoute`:

```typescript
// shipment-detail.component.ts
// Reading path params — equivalent of @PathVariable in Spring

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { ShipmentService } from '../services/shipment.service';
import { Shipment } from '../models/shipment.model';

@Component({
  selector: 'app-shipment-detail',
  template: `
    <div *ngIf="shipment">
      <h2>{{ shipment.trackingId }}</h2>
      <p>Origin: {{ shipment.origin }}</p>
      <p>Destination: {{ shipment.destination }}</p>
      <p>Status: {{ shipment.status }}</p>
    </div>
    <div *ngIf="!shipment">
      <p>Loading shipment details...</p>
    </div>
  `
})
export class ShipmentDetailComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private shipmentService = inject(ShipmentService);
  shipment: Shipment | null = null;

  ngOnInit() {
    // Option 1: Observable (reacts to param changes without remounting)
    // Use this when the component stays mounted but the :id changes
    this.route.params.subscribe(params => {
      const id = params['id'];  // Extract :id from URL
      this.shipmentService.getById(id).subscribe(data => {
        this.shipment = data;
      });
    });

    // Option 2: Snapshot (one-time read)
    // Use this when the component is always destroyed and recreated on navigation
    // const id = this.route.snapshot.params['id'];
  }
}
```

### Query Parameters — `?key=value`

Query parameters are appended after `?` in the URL. In Spring Boot, you'd use
`@RequestParam`. In Angular, you use `ActivatedRoute.queryParams`:

```typescript
// shipment-list.component.ts
// Reading query params — equivalent of @RequestParam in Spring

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { ShipmentService } from '../services/shipment.service';

@Component({
  selector: 'app-shipment-list',
  template: `
    <h2>Shipments</h2>
    <div class="filters">
      <button (click)="filterByStatus('all')">All</button>
      <button (click)="filterByStatus('pending')">Pending</button>
      <button (click)="filterByStatus('delivered')">Delivered</button>
    </div>
    <ul>
      <li *ngFor="let s of shipments">{{ s.trackingId }} — {{ s.status }}</li>
    </ul>
  `
})
export class ShipmentListComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private shipmentService = inject(ShipmentService);
  shipments: any[] = [];

  ngOnInit() {
    // Subscribe to query param changes
    // URL: /shipments?status=pending&page=1
    this.route.queryParams.subscribe(params => {
      const status = params['status'] || 'all';
      const page = params['page'] || 1;
      this.loadShipments(status, +page);
    });
  }

  loadShipments(status: string, page: number) {
    this.shipmentService.list(status, page).subscribe(data => {
      this.shipments = data;
    });
  }

  filterByStatus(status: string) {
    // Programmatically update query params
    inject(Router).navigate([], {
      queryParams: { status },
      queryParamsHandling: 'merge'  // Keep other query params
    });
  }
}
```

### Comparison Table

| Concept | Spring Boot | Angular |
|---------|------------|---------|
| Path param definition | `@GetMapping("/{id}")` | `{ path: ':id' }` |
| Path param reading | `@PathVariable String id` | `route.params['id']` |
| Query param reading | `@RequestParam String status` | `route.queryParams['status']` |
| Optional query param | `@RequestParam(required=false)` | Params are always optional |
| Reactive param access | N/A (request-scoped) | `route.params.subscribe()` |
| One-time param read | Default behavior | `route.snapshot.params['id']` |

---

## Phase 5: Nested Routes and Layouts

> [!quote]
> *"Nested routes are like a warehouse with departments that have sub-sections.
> The Shipping department has Outbound, Inbound, and Returns — each with its own space."*

### Why Nested Routes?

Sometimes a page has its own internal navigation. For example, a shipment detail page
might have tabs: Overview, Tracking, Documents.

```
URL: /shipments/SHP-001/tracking

┌─────────────────────────────────────────────────┐
│                  Main Nav                       │
│   Dashboard | [Shipments] | Carriers            │
├─────────────────────────────────────────────────┤
│                                                 │
│  Shipment: SHP-001                              │
│  ┌───────────┬───────────┬───────────┐          │
│  │ Overview  │ [Tracking]│ Documents │  ← tabs  │
│  ├───────────┴───────────┴───────────┤          │
│  │                                   │          │
│  │   Nested <router-outlet>          │          │
│  │   ┌───────────────────────────┐   │          │
│  │   │ TrackingHistoryComponent  │   │          │
│  │   │ (rendered here)           │   │          │
│  │   └───────────────────────────┘   │          │
│  │                                   │          │
│  └───────────────────────────────────┘          │
│                                                 │
├─────────────────────────────────────────────────┤
│                  Footer                         │
└─────────────────────────────────────────────────┘
```

### Route Configuration with Children

```typescript
// app.routes.ts — nested route configuration

export const routes: Routes = [
  { path: '', component: DashboardComponent },
  { path: 'shipments', component: ShipmentListComponent },
  {
    path: 'shipments/:id',
    component: ShipmentLayoutComponent,  // Parent layout with tabs
    children: [
      // Default child — shown when URL is /shipments/:id
      { path: '', component: ShipmentOverviewComponent },
      // Tab routes — /shipments/:id/tracking
      { path: 'tracking', component: TrackingHistoryComponent },
      // /shipments/:id/documents
      { path: 'documents', component: DocumentsComponent }
    ]
  },
  { path: '**', component: NotFoundComponent }
];
```

### Parent Layout Component

```typescript
// shipment-layout.component.ts
// The parent component provides the layout shell (tabs) and a nested <router-outlet>

@Component({
  selector: 'app-shipment-layout',
  template: `
    <div class="shipment-header">
      <h2>Shipment: {{ shipmentId }}</h2>
      <nav class="tabs">
        <a [routerLink]="['/shipments', shipmentId]"
           routerLinkActive="active"
           [routerLinkActiveOptions]="{exact: true}">Overview</a>
        <a [routerLink]="['/shipments', shipmentId, 'tracking']"
           routerLinkActive="active">Tracking</a>
        <a [routerLink]="['/shipments', shipmentId, 'documents']"
           routerLinkActive="active">Documents</a>
      </nav>
    </div>

    <!-- Nested router outlet — child routes render here -->
    <router-outlet></router-outlet>
  `
})
export class ShipmentLayoutComponent {
  private route = inject(ActivatedRoute);
  shipmentId = '';

  ngOnInit() {
    this.route.params.subscribe(p => this.shipmentId = p['id']);
  }
}
```

> [!tip] Spring Parallel
> Nested routes are similar to having a parent `@RequestMapping("/shipments/{id}")`
> on a controller class, with child `@GetMapping("/tracking")` and
> `@GetMapping("/documents")` on individual methods.
> The parent provides the layout, children provide the content.

---

## Phase 6: Route Guards

> [!quote]
> *"Route guards are the security checkpoint at the warehouse gate —
> they check your badge before letting you into restricted areas."*

### What Are Route Guards?

Route guards are functions that run **before** a route is activated (or deactivated).
They decide whether navigation should proceed or be blocked.

**Spring Boot parallel**: Route guards are like **Spring Security filters** or `@PreAuthorize`.

### Types of Guards

| Guard Type | When It Runs | Use Case | Spring Equivalent |
|-----------|-------------|----------|-------------------|
| `canActivate` | Before entering a route | Auth checks | `@PreAuthorize` / `SecurityFilterChain` |
| `canDeactivate` | Before leaving a route | Unsaved changes warning | No direct equivalent |
| `canMatch` | Before matching a route | Feature flags | Conditional `@Profile` beans |
| `resolve` | Before rendering | Pre-fetch data | `@ModelAttribute` pre-loading |

### Functional Guards (Modern Angular — v15+)

```typescript
// guards/auth.guard.ts
// A functional guard — the modern, recommended approach
// Like a Spring Security filter that checks authentication

import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;  // Allow navigation
  }

  // Redirect to login — like Spring Security's loginPage("/login")
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }  // Remember where user wanted to go
  });
};
```

### Role-Based Guard

```typescript
// guards/role.guard.ts
// Like @PreAuthorize("hasRole('ADMIN')") in Spring Security

export const roleGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const requiredRoles = route.data['roles'] as string[];  // Roles from route config

  if (authService.hasAnyRole(requiredRoles)) {
    return true;
  }

  return router.createUrlTree(['/unauthorized']);
};
```

### Unsaved Changes Guard

```typescript
// guards/unsaved-changes.guard.ts
// Prevents navigating away from a form with unsaved changes
// No direct Spring equivalent — this is a frontend UX concern

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('You have unsaved changes. Are you sure you want to leave?');
  }
  return true;
};
```

### Applying Guards to Routes

```typescript
// app.routes.ts — using guards

export const routes: Routes = [
  { path: 'login', component: LoginComponent },

  // Protected route — requires authentication
  {
    path: 'shipments',
    component: ShipmentListComponent,
    canActivate: [authGuard]
  },

  // Role-based protection — requires ADMIN role
  {
    path: 'admin',
    component: AdminDashboardComponent,
    canActivate: [authGuard, roleGuard],
    data: { roles: ['ADMIN', 'MANAGER'] }  // Passed to the guard via route.data
  },

  // Unsaved changes protection
  {
    path: 'shipments/new',
    component: CreateShipmentComponent,
    canActivate: [authGuard],
    canDeactivate: [unsavedChangesGuard]
  },

  { path: '**', component: NotFoundComponent }
];
```

### Route Guard vs Spring Security — Comparison

| Feature | Angular Guard | Spring Security |
|---------|--------------|-----------------|
| Check authentication | `canActivate` returns `true/false` | `SecurityFilterChain.authorizeHttpRequests()` |
| Check roles | Read from `route.data['roles']` | `@PreAuthorize("hasRole('ADMIN')")` |
| Redirect on failure | `router.createUrlTree(['/login'])` | `.loginPage("/login")` |
| Protect all routes | Apply guard to parent route | `.anyRequest().authenticated()` |
| Unsaved changes | `canDeactivate` | No backend equivalent |
| Pre-fetch data | `resolve` | `@ModelAttribute` method |

> [!warning] Guards Are Client-Side Only!
> Angular guards protect the **UI** — they prevent components from rendering.
> They do **NOT** protect your API. You still need Spring Security on the backend.
> A malicious user can bypass frontend guards by calling your API directly.
>
> **Always secure both frontend AND backend.**

---

## Phase 7: Lazy Loading

> [!quote]
> *"Lazy loading is like a just-in-time delivery system — you don't ship all the inventory
> to every warehouse upfront. You only deliver what's needed, when it's needed."*

### The Problem: Large Bundle Size

Without lazy loading, Angular bundles **all** your components into one large JavaScript file.
Users download everything upfront — even features they may never use:

```
WITHOUT Lazy Loading:
┌─────────────────────────────────────────┐
│            main.js (2.5 MB)             │
│                                         │
│  Dashboard + Shipments + Carriers +     │
│  Reports + Admin + Settings + ...       │
│                                         │
│  User downloads ALL code on first load  │
└─────────────────────────────────────────┘

WITH Lazy Loading:
┌───────────────┐
│ main.js (500K)│  ← Only core app + dashboard
└───────────────┘
      │
      │  User navigates to /reports
      ▼
┌───────────────┐
│ reports.js    │  ← Downloaded on demand
│ (200K)        │
└───────────────┘
      │
      │  User navigates to /admin
      ▼
┌───────────────┐
│ admin.js      │  ← Downloaded on demand
│ (150K)        │
└───────────────┘
```

### Lazy Loading with `loadComponent` (Standalone Components)

```typescript
// app.routes.ts — lazy loading standalone components

export const routes: Routes = [
  // Eagerly loaded — always in the main bundle
  { path: '', component: DashboardComponent },
  { path: 'shipments', component: ShipmentListComponent },

  // Lazy loaded — downloaded only when user navigates to /reports
  {
    path: 'reports',
    loadComponent: () =>
      import('./reports/reports.component').then(m => m.ReportsComponent)
  },

  // Lazy loaded with children
  {
    path: 'admin',
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.adminRoutes),
    canActivate: [authGuard, roleGuard],
    data: { roles: ['ADMIN'] }
  },

  { path: '**', component: NotFoundComponent }
];
```

### Lazy Loading a Feature's Child Routes

```typescript
// admin/admin.routes.ts — a separate route file loaded on demand

import { Routes } from '@angular/router';

export const adminRoutes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./admin-layout.component').then(m => m.AdminLayoutComponent),
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./admin-dashboard.component').then(m => m.AdminDashboardComponent)
      },
      {
        path: 'users',
        loadComponent: () =>
          import('./user-management.component').then(m => m.UserManagementComponent)
      },
      {
        path: 'settings',
        loadComponent: () =>
          import('./settings.component').then(m => m.SettingsComponent)
      }
    ]
  }
];
```

### Benefits of Lazy Loading

| Benefit | Description |
|---------|-------------|
| ✅ Faster initial load | Users only download what they need immediately |
| ✅ Better performance | Smaller JavaScript bundles parse faster |
| ✅ Better caching | Feature bundles are cached independently |
| ✅ Scalable architecture | New features don't bloat the main bundle |

> [!tip] When to Lazy Load
> - ✅ Feature sections users may not visit (admin, reports, settings)
> - ✅ Large components with heavy dependencies (chart libraries, editors)
> - ❌ Core components every user sees (dashboard, navigation, login)

---

## Phase 8: Angular Router vs React Router

> [!quote]
> *"Angular gives you the whole shipping fleet out of the box.
> React gives you the ocean — you pick your own ships."*

### Feature Comparison

| Feature | Angular Router | React Router (v6) |
|---------|---------------|-------------------|
| **Installation** | ✅ Built-in (`@angular/router`) | ❌ Separate package (`react-router-dom`) |
| **Route definition** | TypeScript array of `Route` objects | JSX `<Route>` components or `createBrowserRouter` |
| **Route outlet** | `<router-outlet>` directive | `<Outlet />` component |
| **Navigation links** | `routerLink` directive | `<Link to="...">` component |
| **Programmatic nav** | `Router.navigate()` | `useNavigate()` hook |
| **Route params** | `ActivatedRoute.params` Observable | `useParams()` hook |
| **Query params** | `ActivatedRoute.queryParams` Observable | `useSearchParams()` hook |
| **Guards** | ✅ `canActivate`, `canDeactivate`, etc. | ❌ Must build manually (wrapper components) |
| **Lazy loading** | `loadComponent` / `loadChildren` | `React.lazy()` + `<Suspense>` |
| **Nested routes** | `children` array + nested `<router-outlet>` | Nested `<Route>` + `<Outlet>` |
| **Active link styling** | `routerLinkActive` directive | `<NavLink>` with `className` callback |
| **Resolvers** | ✅ Built-in `resolve` guard | ❌ `loader` in React Router v6.4+ (data APIs) |

### Lazy Loading Syntax Comparison

```typescript
// Angular — lazy loading a route
{
  path: 'reports',
  loadComponent: () =>
    import('./reports/reports.component').then(m => m.ReportsComponent)
}
```

```jsx
// React — lazy loading a route
const Reports = React.lazy(() => import('./Reports'));

<Route path="/reports" element={
  <Suspense fallback={<Loading />}>
    <Reports />
  </Suspense>
} />
```

### Guard vs ProtectedRoute Pattern

```typescript
// Angular — route guard (declarative, in route config)
{
  path: 'admin',
  component: AdminComponent,
  canActivate: [authGuard]  // Clean and declarative
}
```

```jsx
// React — protected route wrapper (manual pattern)
function ProtectedRoute({ children }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" />;
  return children;
}

<Route path="/admin" element={
  <ProtectedRoute>
    <AdminComponent />
  </ProtectedRoute>
} />
```

> [!success] Angular Advantage for Java Developers
> Angular's routing feels more natural to Spring Boot developers because:
> - Routes are configured as **data structures** (like Spring's `SecurityFilterChain`)
> - Guards are like **middleware/filters** (not wrapper components)
> - The router is a **service** you inject (familiar DI pattern)
> - Route params are **Observables** (reactive, like Spring WebFlux)

---

## Quick Reference Cheat Sheet

```typescript
// ─── Route Setup ───────────────────────────────────────
{ path: 'users', component: UsersComponent }         // static
{ path: 'users/:id', component: UserDetailComponent } // param
{ path: '**', component: NotFoundComponent }          // 404

// ─── Template Navigation ──────────────────────────────
<a routerLink="/users">Users</a>                      // static link
<a [routerLink]="['/users', user.id]">Detail</a>      // dynamic link
routerLinkActive="active"                              // active class

// ─── Programmatic Navigation ──────────────────────────
this.router.navigate(['/users', id]);                  // path nav
this.router.navigate(['/users'], {                     // with query params
  queryParams: { role: 'admin' }
});

// ─── Reading Route Params ─────────────────────────────
this.route.params.subscribe(p => p['id']);              // path param
this.route.queryParams.subscribe(q => q['role']);       // query param
this.route.snapshot.params['id'];                       // one-time read

// ─── Guards ───────────────────────────────────────────
canActivate: [authGuard]                               // protect entry
canDeactivate: [unsavedChangesGuard]                   // protect exit

// ─── Lazy Loading ─────────────────────────────────────
loadComponent: () => import('./x.component').then(m => m.XComponent)
loadChildren: () => import('./x.routes').then(m => m.xRoutes)
```

---

## Cross-References

- [[Angular Fundamentals]] — Core concepts, project structure, modules
- [[Components and Templates]] — Building the components that routes point to
- [[Services and Dependency Injection]] — Services used in guards and resolvers
- [[RxJS and Reactive Programming]] — Observables used in route params
- [[React Routing]] — Compare Angular routing with React Router
- [[React vs Angular Comparison]] — Full framework comparison
