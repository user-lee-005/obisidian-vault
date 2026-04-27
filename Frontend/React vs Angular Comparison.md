---
tags:
  - frontend
  - react
  - angular
  - comparison
aliases:
  - React vs Angular
  - Angular vs React
---

# ⚔️ React vs Angular Comparison

> *"The best framework is the one that solves your problem. The best developer is the one who knows both."*

This is THE bridge note connecting the React and Angular worlds. If you already know one and want to learn the other — or you're starting fresh and want to understand both — this is where you begin.

---

## 📖 Phase 1: Overview

### What Are We Comparing?

| | React | Angular |
|--|-------|---------|
| **Type** | Library (UI only) | Framework (full-featured) |
| **Creator** | Meta (Facebook) | Google |
| **Released** | 2013 | 2016 (Angular 2+) |
| **Language** | JavaScript / TypeScript (optional) | TypeScript (required) |
| **Philosophy** | Flexible — choose your own stack | Opinionated — batteries included |
| **License** | MIT | MIT |

### 🏗️ The Spring Boot Analogy

> [!tip] For Spring Boot Developers
> **React** is like **Spring Core** — it gives you the foundation (component rendering), and you pick everything else: routing, state management, HTTP client, forms library.
>
> **Angular** is like **Spring Boot with all starters** — routing, forms, HTTP client, dependency injection, testing utilities — everything is pre-configured and follows conventions.
>
> If you love Spring Boot's "convention over configuration" philosophy, Angular will feel like home.
> If you prefer assembling your own stack with maximum flexibility, React is your playground.

### Market & Community

| Factor | React | Angular |
|--------|-------|---------|
| **npm weekly downloads** | ~25M+ | ~3M+ |
| **GitHub stars** | ~230k+ | ~97k+ |
| **Stack Overflow questions** | Very high | High |
| **Job market** | Largest frontend demand | Strong enterprise demand |
| **Used by** | Meta, Netflix, Airbnb, Uber | Google, Microsoft, Forbes, Samsung |
| **Community** | Massive, fragmented (many libraries) | Large, unified (official solutions) |
| **Learning resources** | Abundant (varying quality) | Abundant (more structured) |

> [!note] Key Insight
> React has a larger community overall, but Angular dominates in enterprise and government sectors. Both have excellent job prospects — your choice should depend on your target industry.

---

## 🏛️ Phase 2: Architecture Comparison

### High-Level Architecture

```
React App:                          Angular App:
┌──────────────────────┐            ┌──────────────────────┐
│  Components          │            │  Modules / Standalone │
│  (JSX / TSX)         │            │  ├── Components       │
│                      │            │  ├── Services          │
│  + Hooks             │            │  ├── Directives        │
│  + Context / Redux   │            │  ├── Pipes             │
│  + React Router      │            │  └── Guards            │
│  + React Hook Form   │            │                        │
│  + Axios / Fetch     │            │  Built-in:             │
│  + Zustand / Jotai   │            │  Router, Forms,        │
└──────────────────────┘            │  HttpClient, DI,       │
 (You assemble the stack)           │  Animations, i18n      │
                                    └──────────────────────┘
                                     (Everything included)
```

### The Assembly Analogy

```
Building a React App               Building an Angular App
═══════════════════════             ═══════════════════════

Step 1: npm create vite@latest      Step 1: ng new my-app
Step 2: Pick a router               (That's it. Everything
Step 3: Pick a state manager         is already configured.)
Step 4: Pick a form library
Step 5: Pick an HTTP client
Step 6: Pick a CSS approach
Step 7: Configure everything
```

> [!tip] Spring Boot Parallel
> This is exactly like choosing between a Spring Boot starter project (Angular) vs. building a Spring app from scratch with manually selected dependencies (React).

### Project Structure Comparison

```
React Project (typical):            Angular Project (ng new):
my-react-app/                       my-angular-app/
├── src/                            ├── src/
│   ├── components/                 │   ├── app/
│   │   ├── ShipmentCard.tsx        │   │   ├── app.component.ts
│   │   ├── ShipmentList.tsx        │   │   ├── app.component.html
│   │   └── Header.tsx              │   │   ├── app.component.css
│   ├── hooks/                      │   │   ├── app.component.spec.ts
│   │   └── useShipments.ts         │   │   ├── app.config.ts
│   ├── context/                    │   │   ├── app.routes.ts
│   │   └── ShipmentContext.tsx     │   │   ├── shipment/
│   ├── pages/                      │   │   │   ├── shipment-card/
│   │   ├── Home.tsx                │   │   │   │   ├── shipment-card.component.ts
│   │   └── Dashboard.tsx           │   │   │   │   ├── shipment-card.component.html
│   ├── services/                   │   │   │   │   ├── shipment-card.component.css
│   │   └── shipmentApi.ts          │   │   │   │   └── shipment-card.component.spec.ts
│   ├── App.tsx                     │   │   │   └── shipment.service.ts
│   └── main.tsx                    │   │   └── shared/
├── package.json                    │   ├── index.html
└── vite.config.ts                  │   └── main.ts
                                    ├── angular.json
                                    └── package.json

 (You decide the structure)          (CLI generates structure)
```

> [!info] Angular's Convention
> Angular enforces one component per folder with co-located `.ts`, `.html`, `.css`, and `.spec.ts` files. React has no enforced convention — you choose your own file organization.

---

## 🧩 Phase 3: Component Comparison

Components are the building blocks in **both** frameworks. But they look very different.

### The Same Shipment Card — Two Ways

**React Component:**

```tsx
// ShipmentCard.tsx — a single file
import { useState } from 'react';

interface ShipmentCardProps {
  shipmentId: string;
  origin: string;
  destination: string;
  status: string;
  onTrack: (id: string) => void;
}

export function ShipmentCard({ shipmentId, origin, destination, status, onTrack }: ShipmentCardProps) {
  const [isExpanded, setIsExpanded] = useState(false);

  return (
    <div className="shipment-card">
      <h3>Shipment #{shipmentId}</h3>
      <p>{origin} → {destination}</p>
      <span className={`status status-${status}`}>{status}</span>

      <button onClick={() => setIsExpanded(!isExpanded)}>
        {isExpanded ? 'Collapse' : 'Expand'}
      </button>

      {isExpanded && (
        <div className="details">
          <button onClick={() => onTrack(shipmentId)}>Track</button>
        </div>
      )}
    </div>
  );
}
```

**Angular Component:**

```typescript
// shipment-card.component.ts — logic
import { Component, input, output, signal } from '@angular/core';

@Component({
  selector: 'app-shipment-card',
  standalone: true,
  templateUrl: './shipment-card.component.html',
  styleUrl: './shipment-card.component.css'
})
export class ShipmentCardComponent {
  shipmentId = input.required<string>();
  origin = input.required<string>();
  destination = input.required<string>();
  status = input.required<string>();
  track = output<string>();

  isExpanded = signal(false);

  toggleExpand() {
    this.isExpanded.update(v => !v);
  }

  onTrack() {
    this.track.emit(this.shipmentId());
  }
}
```

```html
<!-- shipment-card.component.html — template -->
<div class="shipment-card">
  <h3>Shipment #{{ shipmentId() }}</h3>
  <p>{{ origin() }} → {{ destination() }}</p>
  <span class="status" [class]="'status-' + status()">{{ status() }}</span>

  <button (click)="toggleExpand()">
    @if (isExpanded()) { Collapse } @else { Expand }
  </button>

  @if (isExpanded()) {
    <div class="details">
      <button (click)="onTrack()">Track</button>
    </div>
  }
</div>
```

### Component Anatomy Table

| Concept | React | Angular |
|---------|-------|---------|
| **File structure** | Single `.tsx` file | Separate `.ts` + `.html` + `.css` files |
| **Component type** | Function | Class with `@Component` decorator |
| **Template** | JSX — HTML-in-JS | Separate HTML template file |
| **Passing data in** | `props` (function args) | `@Input()` / `input()` signals |
| **Emitting events** | Callback props | `@Output()` / `output()` + EventEmitter |
| **Local state** | `useState` hook | Signals / class properties |
| **Lifecycle** | `useEffect` hook | `ngOnInit`, `ngOnDestroy`, etc. |
| **Ref to DOM** | `useRef` hook | `@ViewChild` / `viewChild()` |

> [!tip] Spring Boot Parallel
> Angular's `@Input` and `@Output` decorators are like Spring's `@RequestParam` and `@ResponseBody` — annotations that declare the contract. React's props are more like passing arguments to a plain method.

For React component details, see [[Components and Props]].
For Angular component details, see [[Components and Templates]].

---

## 📝 Phase 4: Template / JSX Comparison

This is one of the biggest day-to-day differences. React uses **JSX** (JavaScript XML), while Angular uses **HTML templates** with special syntax.

### Conditional Rendering

**React — ternary operator in JSX:**

```tsx
function ShipmentStatus({ status }: { status: string }) {
  return (
    <div>
      {status === 'delivered' ? (
        <span className="success">✅ Delivered</span>
      ) : status === 'in-transit' ? (
        <span className="warning">🚚 In Transit</span>
      ) : (
        <span className="danger">❌ Pending</span>
      )}
    </div>
  );
}
```

**Angular — `@if` control flow:**

```html
<div>
  @if (status() === 'delivered') {
    <span class="success">✅ Delivered</span>
  } @else if (status() === 'in-transit') {
    <span class="warning">🚚 In Transit</span>
  } @else {
    <span class="danger">❌ Pending</span>
  }
</div>
```

### List Rendering

**React — `.map()` with keys:**

```tsx
function ShipmentList({ shipments }: { shipments: Shipment[] }) {
  return (
    <ul>
      {shipments.map(s => (
        <li key={s.id}>
          {s.origin} → {s.destination}
        </li>
      ))}
    </ul>
  );
}
```

**Angular — `@for` control flow:**

```html
<ul>
  @for (s of shipments(); track s.id) {
    <li>{{ s.origin }} → {{ s.destination }}</li>
  }
</ul>
```

### Full Template Syntax Mapping

| Operation | React (JSX) | Angular (Template) |
|-----------|-------------|-------------------|
| **Text interpolation** | `{value}` | `{{ value }}` |
| **Conditional** | `{cond ? <A/> : <B/>}` | `@if (cond) { ... } @else { ... }` |
| **Loop** | `{items.map(i => <X key={i.id}/>)}` | `@for (i of items; track i.id) { ... }` |
| **CSS class** | `className="active"` | `class="active"` |
| **Dynamic class** | `className={isActive ? 'active' : ''}` | `[class.active]="isActive"` |
| **Inline style** | `style={{ color: 'red' }}` (object) | `[style.color]="'red'"` |
| **Click event** | `onClick={handleClick}` | `(click)="handleClick()"` |
| **Input binding** | `value={val} onChange={...}` | `[(ngModel)]="val"` (two-way) |
| **Ref** | `ref={myRef}` | `#myRef` (template ref) |
| **Render child** | `{children}` | `<ng-content />` |
| **Switch** | Multiple ternaries or `switch` | `@switch (val) { @case ... }` |
| **Empty state** | `{items.length === 0 && <Empty/>}` | `@empty { ... }` inside `@for` |

> [!note] JSX vs Templates
> **React's approach**: Everything is JavaScript. Conditions are ternaries, loops are `.map()`, and event handlers are function references. If you know JS, you know JSX.
>
> **Angular's approach**: Templates are enhanced HTML. Angular adds its own syntax (`@if`, `@for`, `(click)`, `[binding]`) on top of standard HTML. Separation of template and logic.

---

## 🗃️ Phase 5: State Management Comparison

State management is how your app tracks and reacts to changes in data. Both frameworks have a spectrum of solutions — from simple local state to complex global stores.

### The State Management Spectrum

```
Simple ◄──────────────────────────────────────────────► Complex

React:
  useState ──► useReducer ──► Context API ──► Zustand ──► Redux Toolkit

Angular:
  Signals ──► BehaviorSubject ──► Service State ──► Component Store ──► NgRx
```

### Comparison Table

| Approach | React | Angular |
|----------|-------|---------|
| **Local state** | `useState` / `useReducer` | Signals / class properties |
| **Shared state (simple)** | Context API | Services with `BehaviorSubject` / Signals |
| **Shared state (complex)** | Redux Toolkit | NgRx Store |
| **Lightweight store** | Zustand / Jotai | NgRx Component Store / Signal Store |
| **Server state** | TanStack Query | NgRx Effects / Angular HTTP + Signals |

### Local State — Side by Side

**React:**

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

**Angular:**

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p>Count: {{ count() }}</p>
    <button (click)="increment()">+</button>
  `
})
export class CounterComponent {
  count = signal(0);

  increment() {
    this.count.update(c => c + 1);
  }
}
```

### Shared State via Service / Context

**React — Context API:**

```tsx
// ShipmentContext.tsx
const ShipmentContext = createContext<ShipmentState | null>(null);

export function ShipmentProvider({ children }: { children: React.ReactNode }) {
  const [shipments, setShipments] = useState<Shipment[]>([]);

  return (
    <ShipmentContext.Provider value={{ shipments, setShipments }}>
      {children}
    </ShipmentContext.Provider>
  );
}

export function useShipments() {
  const ctx = useContext(ShipmentContext);
  if (!ctx) throw new Error('Must be inside ShipmentProvider');
  return ctx;
}
```

**Angular — Injectable Service:**

```typescript
// shipment.service.ts
@Injectable({ providedIn: 'root' })
export class ShipmentService {
  private shipments = signal<Shipment[]>([]);

  readonly allShipments = this.shipments.asReadonly();

  addShipment(shipment: Shipment) {
    this.shipments.update(list => [...list, shipment]);
  }
}
```

> [!tip] Spring Boot Parallel
> Angular Services with `@Injectable({ providedIn: 'root' })` are singleton beans — just like Spring's `@Service` with default singleton scope. React Context is more like passing a request-scoped attribute through middleware.

For React state details, see [[React State Management]].
For Angular state details, see [[Angular State Management]].

---

## 🗺️ Phase 6: Routing Comparison

Both frameworks support client-side routing, but React requires a third-party library while Angular includes a router.

| Aspect | React (react-router-dom) | Angular (@angular/router) |
|--------|--------------------------|---------------------------|
| **Included?** | No — install separately | Yes — built-in |
| **Config style** | JSX or object-based | Array of route objects |
| **Lazy loading** | `React.lazy()` + `Suspense` | `loadComponent` / `loadChildren` |
| **Guards** | Loader functions / manual | `canActivate`, `canDeactivate` guards |
| **Params** | `useParams()` hook | `ActivatedRoute` service |
| **Navigation** | `useNavigate()` hook | `Router.navigate()` service |

### Route Definition — Side by Side

**React Router:**

```tsx
// App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const ShipmentDetail = lazy(() => import('./pages/ShipmentDetail'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<p>Loading...</p>}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/shipments/:id" element={<ShipmentDetail />} />
          <Route path="*" element={<p>404 Not Found</p>} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**Angular Router:**

```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'shipments/:id',
    loadComponent: () => import('./shipment-detail/shipment-detail.component')
      .then(m => m.ShipmentDetailComponent)
  },
  { path: '**', redirectTo: '' }
];
```

### Route Guards

**React — Manual guard via loader:**

```tsx
// No built-in guard. Use a wrapper component or route loader.
function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" />;
  return children;
}
```

**Angular — Built-in functional guard:**

```typescript
// auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);
  return authService.isAuthenticated() ? true : router.createUrlTree(['/login']);
};

// In routes:
{ path: 'dashboard', component: DashboardComponent, canActivate: [authGuard] }
```

> [!tip] Spring Boot Parallel
> Angular's route guards are like Spring Security's `@PreAuthorize` or `SecurityFilterChain` — declarative access control at the route level. React's approach is more like manually checking authentication in a controller method.

For React routing details, see [[React Routing]].
For Angular routing details, see [[Angular Routing]].

---

## 📋 Phase 7: Forms Comparison

Forms are where Angular **really** shines compared to React. Angular has a powerful built-in forms system; React relies on third-party libraries.

| Aspect | React | Angular |
|--------|-------|---------|
| **Built-in?** | No — use `react-hook-form` or `formik` | Yes — `ReactiveFormsModule` |
| **Approach** | Uncontrolled by default | Reactive forms (model-driven) |
| **Validation** | Manual or library-based | Built-in validators + custom validators |
| **Form groups** | Nested objects (library) | `FormGroup` / `FormArray` |
| **Two-way binding** | Manual: `value` + `onChange` | `[(ngModel)]` or `formControlName` |

### Shipment Creation Form — Side by Side

**React (with react-hook-form):**

```tsx
import { useForm } from 'react-hook-form';

interface ShipmentForm {
  origin: string;
  destination: string;
  weight: number;
}

function CreateShipment() {
  const { register, handleSubmit, formState: { errors } } = useForm<ShipmentForm>();

  const onSubmit = (data: ShipmentForm) => {
    console.log('Creating shipment:', data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('origin', { required: 'Origin is required' })} placeholder="Origin" />
      {errors.origin && <span>{errors.origin.message}</span>}

      <input {...register('destination', { required: 'Destination is required' })} placeholder="Destination" />
      {errors.destination && <span>{errors.destination.message}</span>}

      <input type="number" {...register('weight', { min: { value: 0.1, message: 'Min 0.1 kg' } })} placeholder="Weight (kg)" />
      {errors.weight && <span>{errors.weight.message}</span>}

      <button type="submit">Create Shipment</button>
    </form>
  );
}
```

**Angular (Reactive Forms):**

```typescript
// create-shipment.component.ts
import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';

@Component({
  selector: 'app-create-shipment',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './create-shipment.component.html'
})
export class CreateShipmentComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    origin:      ['', Validators.required],
    destination: ['', Validators.required],
    weight:      [0, [Validators.required, Validators.min(0.1)]]
  });

  onSubmit() {
    if (this.form.valid) {
      console.log('Creating shipment:', this.form.value);
    }
  }
}
```

```html
<!-- create-shipment.component.html -->
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <input formControlName="origin" placeholder="Origin" />
  @if (form.get('origin')?.hasError('required') && form.get('origin')?.touched) {
    <span>Origin is required</span>
  }

  <input formControlName="destination" placeholder="Destination" />
  @if (form.get('destination')?.hasError('required') && form.get('destination')?.touched) {
    <span>Destination is required</span>
  }

  <input type="number" formControlName="weight" placeholder="Weight (kg)" />
  @if (form.get('weight')?.hasError('min') && form.get('weight')?.touched) {
    <span>Min 0.1 kg</span>
  }

  <button type="submit" [disabled]="form.invalid">Create Shipment</button>
</form>
```

> [!tip] Spring Boot Parallel
> Angular's Reactive Forms are like Spring's `@Valid` + `BindingResult` — a declarative validation system built into the framework. React's approach is more like wiring up a third-party validation library (e.g., Hibernate Validator manually).

For React form details, see [[Event Handling and Forms]].
For Angular form details, see [[Angular Forms]].

---

## 🌐 Phase 8: HTTP / API Calls

How does each framework communicate with your Spring Boot backend?

| Aspect | React | Angular |
|--------|-------|---------|
| **Built-in HTTP?** | No | Yes — `HttpClient` |
| **Common choice** | `fetch()` or `axios` | `HttpClient` (returns Observables) |
| **Interceptors** | Manual (axios interceptors) | Built-in `HttpInterceptorFn` |
| **Typed responses** | Manual typing | Generic typing on `HttpClient` |
| **Error handling** | try/catch or `.catch()` | `catchError` RxJS operator |

### Fetching Shipments — Side by Side

**React (with fetch):**

```tsx
import { useState, useEffect } from 'react';

function useShipments() {
  const [shipments, setShipments] = useState<Shipment[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('/api/shipments')
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => setShipments(data))
      .catch(err => setError(err.message))
      .finally(() => setLoading(false));
  }, []);

  return { shipments, loading, error };
}
```

**Angular (with HttpClient):**

```typescript
// shipment.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class ShipmentService {
  private http = inject(HttpClient);

  getShipments() {
    return this.http.get<Shipment[]>('/api/shipments');
  }
}

// shipment-list.component.ts
@Component({ ... })
export class ShipmentListComponent {
  private shipmentService = inject(ShipmentService);

  shipments = toSignal(this.shipmentService.getShipments(), { initialValue: [] });
}
```

### Interceptors — Adding Auth Token

**React (axios interceptor):**

```tsx
import axios from 'axios';

const api = axios.create({ baseURL: '/api' });

api.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

**Angular (functional interceptor):**

```typescript
// auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');
  if (token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }
  return next(req);
};

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor]))
  ]
};
```

> [!tip] Spring Boot Parallel
> Angular's `HttpInterceptorFn` is like Spring's `ClientHttpRequestInterceptor` or a `WebClient` filter — middleware that wraps every HTTP call. React has no built-in equivalent; you build it with `axios` interceptors or custom `fetch` wrappers.

---

## 💉 Phase 9: Dependency Injection

This is where Angular truly mirrors Spring Boot — and where React is fundamentally different.

| Aspect | React | Angular |
|--------|-------|---------|
| **DI system?** | ❌ None built-in | ✅ Full hierarchical DI |
| **Service pattern** | Custom hooks or Context | `@Injectable` classes |
| **Singleton services** | Manual (module-level vars) | `providedIn: 'root'` |
| **Scoped services** | Context providers at tree level | Component-level `providers` |
| **Constructor injection** | N/A | `inject()` function or constructor |
| **Testing** | Mock modules or custom providers | `TestBed.configureTestingModule` |

### The DI Mapping

```
Spring Boot                    Angular                     React
═══════════════════           ═══════════════════         ═══════════════════
@Service                      @Injectable()               Custom hook
@Autowired                    inject() / constructor      useContext()
@Scope("singleton")           providedIn: 'root'          Module-level variable
@Scope("prototype")           providers: [MyService]      New instance in hook
@Component                    @Component                  Function component
ApplicationContext             Injector                    React Context
```

> [!important] For Spring Boot Developers
> **Angular's DI is the closest thing to Spring DI in the frontend world.** If you understand `@Service`, `@Autowired`, and Spring's `ApplicationContext`, Angular's DI will feel instantly familiar.
>
> React simply doesn't have DI. It uses **composition** (passing things via props/context) instead of **injection** (framework resolving dependencies automatically).

### Example: Injecting a Logger Service

**Angular:**

```typescript
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(message: string) { console.log(`[LOG] ${message}`); }
}

@Component({ ... })
export class ShipmentComponent {
  private logger = inject(LoggerService); // ≈ @Autowired

  ngOnInit() {
    this.logger.log('ShipmentComponent initialized');
  }
}
```

**React (closest equivalent — custom hook):**

```tsx
// No DI — just import and use
function useLogger() {
  return {
    log: (message: string) => console.log(`[LOG] ${message}`)
  };
}

function ShipmentComponent() {
  const logger = useLogger(); // Just a function call, not injection

  useEffect(() => {
    logger.log('ShipmentComponent mounted');
  }, []);

  return <div>...</div>;
}
```

For Angular DI details, see [[Services and Dependency Injection]].

---

## 🎨 Phase 10: Styling

| Aspect | React | Angular |
|--------|-------|---------|
| **Default approach** | Global CSS (no scoping) | Component-scoped CSS (ViewEncapsulation) |
| **CSS Modules** | ✅ Supported (`.module.css`) | Not common (use component CSS) |
| **CSS-in-JS** | styled-components, Emotion | Not common |
| **Tailwind CSS** | Very popular | Supported |
| **Scoped by default?** | ❌ No | ✅ Yes |

### How Scoping Works

**React — CSS Modules:**

```tsx
// ShipmentCard.module.css
// .card { background: #f0f0f0; }

import styles from './ShipmentCard.module.css';

function ShipmentCard() {
  return <div className={styles.card}>...</div>;
}
```

**Angular — Component CSS (auto-scoped):**

```typescript
@Component({
  selector: 'app-shipment-card',
  templateUrl: './shipment-card.component.html',
  styleUrl: './shipment-card.component.css'  // Scoped automatically!
})
export class ShipmentCardComponent { }
```

```css
/* shipment-card.component.css — only affects THIS component */
.card { background: #f0f0f0; }
```

> [!note] Key Difference
> Angular scopes CSS to each component by default using emulated Shadow DOM (attribute selectors). In React, you must opt-in to scoping via CSS Modules, CSS-in-JS, or Tailwind utility classes.

For React styling details, see [[React Styling]].

---

## 🧪 Phase 11: Testing

| Aspect | React | Angular |
|--------|-------|---------|
| **Test runner** | Jest / Vitest | Jasmine + Karma (or Jest) |
| **Component testing** | React Testing Library (RTL) | TestBed + ComponentFixture |
| **Philosophy** | Test behavior, not implementation | Test behavior via TestBed |
| **Mocking services** | Jest mocks / manual | TestBed providers with mocks |
| **E2E** | Playwright / Cypress | Playwright / Cypress |

### Testing the Same Counter Component

**React (Jest + RTL):**

```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

test('increments counter on click', () => {
  render(<Counter />);

  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  fireEvent.click(screen.getByText('+'));

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

**Angular (Jasmine + TestBed):**

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    fixture.detectChanges();
  });

  it('should increment counter on click', () => {
    const compiled = fixture.nativeElement as HTMLElement;

    expect(compiled.textContent).toContain('Count: 0');

    compiled.querySelector('button')!.click();
    fixture.detectChanges();

    expect(compiled.textContent).toContain('Count: 1');
  });
});
```

> [!note] Key Difference
> React Testing Library is minimalist — render, query, assert. Angular's TestBed is more involved but also more powerful, especially for testing services and dependency injection.

For React testing details, see [[React Testing]].
For Angular testing details, see [[Angular Testing]].

---

## ⚡ Phase 12: Performance

| Aspect | React | Angular |
|--------|-------|---------|
| **Rendering strategy** | Virtual DOM diffing | Change detection (Zone.js) |
| **Optimization** | `React.memo`, `useMemo`, `useCallback` | OnPush strategy, Signals |
| **Lazy loading** | `React.lazy()` + `Suspense` | `loadComponent` / `loadChildren` |
| **Bundle size (base)** | ~40-45 KB (gzipped) | ~60-65 KB (gzipped) |
| **Tree shaking** | Excellent (Vite/Webpack) | Excellent (Angular CLI / esbuild) |
| **SSR** | Next.js | Angular SSR (built-in since v17) |
| **Future direction** | React Compiler (auto-memoization) | Signals (Zone.js removal) |

### How Rendering Works

```
React — Virtual DOM:
────────────────────
State changes → Re-render component → Virtual DOM diff → Patch real DOM
                                       ▲
                                       │
                               React.memo / useMemo
                               skip if props unchanged

Angular — Change Detection:
───────────────────────────
Event/Timer/HTTP → Zone.js detects → Check all components → Update DOM
                                      ▲
                                      │
                              OnPush strategy: only check
                              when @Input ref changes or
                              signals update
```

> [!tip] Performance Tip
> Both frameworks are fast enough for almost any application. Performance issues are almost always caused by **bad code patterns** (unnecessary re-renders, large lists without virtualization) — not the framework itself.

For React performance details, see [[React Performance Optimization]].

---

## 📈 Phase 13: Learning Curve

### The Learning Journey

```
Weeks:  1    2    3    4    5    6    7    8    9    10

React:  ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
        ▲ Easy start          ▲ Overwhelmed by choices
        Basic components      (Which router? Which state lib?
        are simple             How to structure? Best practices?)

Angular: ░░░░░░░░░████████████████████████████████████
              ▲ Steep initial       ▲ Clear path forward
              TypeScript, DI,       (Everything is built-in,
              decorators, RxJS      patterns are established)
              modules...
```

### For Spring Boot Developers Specifically

| Familiar Concept | Angular Equivalent | React Equivalent |
|------------------|--------------------|------------------|
| `@Service` | `@Injectable` | Custom hook |
| `@Autowired` | `inject()` | `useContext()` |
| `@Controller` mapping | Route config | Route config |
| `@Valid` + BindingResult | Reactive Forms validators | react-hook-form |
| Interceptor / Filter | `HttpInterceptorFn` | axios interceptor |
| `application.properties` | `environment.ts` | `.env` files |
| Maven / Gradle | Angular CLI | Vite / CRA |
| JUnit + Mockito | Jasmine + TestBed | Jest + RTL |

> [!important] Recommendation for Spring Boot Developers
> **If you want the easiest transition**: Choose Angular. The DI system, service pattern, TypeScript-first approach, and opinionated structure will feel familiar.
>
> **If you want to broaden your perspective**: Choose React. It will teach you a completely different way of thinking about UI — functional composition over OOP patterns.
>
> **Best of all**: Learn both. Start with whichever interests you more, then learn the other.

---

## 🤔 Phase 14: Which Should You Choose?

### Decision Matrix

| Factor | Choose React | Choose Angular |
|--------|-------------|----------------|
| **Team background** | JS-heavy, startup culture | Java/C#, enterprise culture |
| **Project size** | Small to medium apps | Medium to large enterprise apps |
| **Flexibility** | Want to choose every library | Want opinionated, pre-configured setup |
| **Time to prototype** | Quick MVP / proof of concept | Large-scale production app |
| **Existing ecosystem** | Already using React Native | Already using Angular Material |
| **Learning preference** | Learn by building, explore options | Learn structured patterns, follow docs |
| **Spring Boot dev?** | Want something different | Want familiar patterns (DI, services) |
| **Mobile apps** | React Native (shared code) | Ionic / NativeScript |
| **SSR needed** | Next.js (mature ecosystem) | Angular SSR (built-in) |
| **Corporate standard** | Check your company's preference | Check your company's preference |

### When React Wins

- You need a lightweight UI for a microservice dashboard
- You're building a prototype and want maximum speed
- Your team already knows JavaScript deeply
- You want to use React Native for mobile later
- You prefer functional programming patterns

### When Angular Wins

- You're building a large enterprise application with many developers
- You need built-in solutions for routing, forms, HTTP, and DI
- Your team comes from Java/C#/Spring Boot backgrounds
- You want consistent patterns across the entire codebase
- You need a full framework with official upgrade paths (`ng update`)

### The Honest Answer

> [!quote] The Real Answer
> The best framework is the one your team can be productive with. Both React and Angular can build excellent applications. The choice often comes down to team experience, project requirements, and organizational preferences — not technical superiority.

---

## 📑 Phase 15: Quick Reference Cheat Sheet

### The Giant Mapping Table

This is the table you'll come back to again and again. Bookmark it.

| Concept | React | Angular |
|---------|-------|---------|
| **Language** | JSX / TSX | TypeScript |
| **Component** | Function + JSX | Class + Template + Style |
| **Props (data in)** | `props` (function args) | `@Input()` / `input()` |
| **Events (data out)** | Callback props | `@Output()` / `output()` + EventEmitter |
| **State (local)** | `useState` | Signals / class properties |
| **State (shared)** | Context API / Zustand | Services with Signals / BehaviorSubject |
| **State (complex)** | Redux Toolkit | NgRx Store |
| **Side effects** | `useEffect` | `ngOnInit` / `effect()` |
| **Computed values** | `useMemo` | `computed()` signal |
| **DI** | N/A (Context / hooks) | `@Injectable` + `inject()` |
| **Routing** | `react-router-dom` (3rd party) | `@angular/router` (built-in) |
| **Route guards** | Manual wrapper components | `canActivate` / `canDeactivate` |
| **Forms** | `react-hook-form` (3rd party) | `ReactiveFormsModule` (built-in) |
| **HTTP** | `fetch` / `axios` | `HttpClient` (built-in) |
| **Interceptors** | `axios` interceptors | `HttpInterceptorFn` (built-in) |
| **Pipes / Transforms** | Regular functions | `@Pipe` + `| pipeName` |
| **Directives** | N/A (JSX logic) | `@Directive` + structural/attribute |
| **Conditional render** | Ternary `{cond ? ... : ...}` | `@if (cond) { ... }` |
| **List render** | `.map()` with `key` | `@for (item of items; track ...)` |
| **Two-way binding** | Manual: `value` + `onChange` | `[(ngModel)]` |
| **Content projection** | `{children}` | `<ng-content />` |
| **Ref to DOM** | `useRef` | `@ViewChild` / `viewChild()` |
| **Lifecycle** | `useEffect` (mount/unmount) | `ngOnInit`, `ngOnDestroy`, etc. |
| **Testing** | Jest + React Testing Library | Jasmine + TestBed |
| **E2E Testing** | Playwright / Cypress | Playwright / Cypress |
| **CLI** | Vite (`npm create vite`) | Angular CLI (`ng new`) |
| **Dev server** | `npm run dev` (Vite) | `ng serve` |
| **Build** | `npm run build` (Vite) | `ng build` |
| **Styling** | CSS Modules / Tailwind / styled-components | Component-scoped CSS |
| **Animations** | `framer-motion` (3rd party) | `@angular/animations` (built-in) |
| **i18n** | `react-i18next` (3rd party) | `@angular/localize` (built-in) |
| **SSR** | Next.js | Angular SSR (built-in) |
| **Mobile** | React Native | Ionic / NativeScript |
| **Bundle analyzer** | `rollup-plugin-visualizer` | `source-map-explorer` |

---

## 🔗 Related Notes

### React Deep Dives

- [[Components and Props]] — React component fundamentals
- [[React State Management]] — useState, useReducer, Context, Redux
- [[React Routing]] — React Router setup and navigation
- [[Event Handling and Forms]] — Forms and event handling in React
- [[React Styling]] — CSS Modules, styled-components, Tailwind
- [[React Testing]] — Jest and React Testing Library
- [[React Performance Optimization]] — Memoization, lazy loading, profiling

### Angular Deep Dives

- [[Components and Templates]] — Angular component fundamentals
- [[Angular State Management]] — Signals, Services, NgRx
- [[Angular Routing]] — Router setup, guards, lazy loading
- [[Angular Forms]] — Template-driven and Reactive forms
- [[Services and Dependency Injection]] — @Injectable, providers, inject()
- [[Angular Testing]] — Jasmine, TestBed, component testing

### Cross-Cutting Concepts

- [[Cloud Fundamentals]] — Deploying your frontend apps
- [[REST]] — The API style your frontend consumes
- [[API Design]] — Designing the API your frontend calls

---

> ⚔️ This is a living comparison. As React and Angular evolve, this note evolves with them. When in doubt, build the same small feature in both and see which feels more natural to you.
