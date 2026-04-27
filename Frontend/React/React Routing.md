---
tags:
  - react
  - routing
  - frontend
  - phase3
created: 2025-07-18
---

# React Routing

> *"A shipment without a route is just cargo sitting in a warehouse — and a web app without routing is just a single page going nowhere."*

---

## Phase 1 — What Is Client-Side Routing?

In your Spring Boot world, every URL change triggers a **full HTTP request** to the server.
The server picks a controller via `@RequestMapping`, renders a view, and sends back an entire HTML page.

**Client-side routing** flips that model. The browser loads the app **once**, and then JavaScript
intercepts URL changes to swap components in-place — no round-trip to the server.

> [!info] Logistics Analogy
> Think of **server-side routing** as a freight hub where every package must return to the
> central warehouse before being redirected. **Client-side routing** is like a local
> distribution centre — packages move between nearby docks without ever leaving the facility.

```
┌─────────────────────────────────────────────────────────┐
│              SERVER-SIDE (Spring MVC)                    │
│                                                         │
│  Browser ──GET /shipments──▶ Server ──HTML──▶ Browser   │
│  Browser ──GET /tracking───▶ Server ──HTML──▶ Browser   │
│          (full page reload each time)                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              CLIENT-SIDE (React Router)                  │
│                                                         │
│  Browser loads app ONCE                                 │
│  /shipments  ──▶ swap to <ShipmentList />               │
│  /tracking   ──▶ swap to <TrackingPage />               │
│          (no server round-trip for navigation)          │
└─────────────────────────────────────────────────────────┘
```

| Aspect             | Spring MVC (`@RequestMapping`) | React Router (client-side)     |
| ------------------- | ------------------------------ | ------------------------------ |
| Page reload         | ✅ Yes, every navigation        | ❌ No, SPA stays loaded         |
| Server involvement  | ✅ Server renders HTML           | ❌ Server only serves API/JSON  |
| Speed               | Slower (network round-trip)    | Faster (instant swap)          |
| SEO (out of the box)| ✅ Good                          | ❌ Needs SSR/pre-rendering      |
| State preservation  | ❌ Lost on reload                | ✅ Preserved across navigations |

---

## Phase 2 — React Router v6 Setup

React Router is the de-facto routing library for React. Install it:

```bash
npm install react-router-dom
```

> [!tip] Version Note
> We use **React Router v6** throughout this guide. v6 introduced `<Routes>` (replacing
> `<Switch>`), relative routes, and the `<Outlet>` pattern. If you see `<Switch>` in
> older tutorials, that's v5 syntax.

### Wrap Your App

```tsx
// main.tsx (or index.tsx)
import { BrowserRouter } from "react-router-dom";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

> [!info] Logistics Analogy
> `BrowserRouter` is the **dispatch system** for your entire logistics network. Without it
> installed, no shipment (page) can be routed anywhere.

---

## Phase 3 — Core Components

### 3.1 `BrowserRouter`

The top-level provider that enables routing. Uses the HTML5 History API.
You wrap your entire app in it — like `@SpringBootApplication` wraps your whole Spring context.

### 3.2 `Routes` and `Route`

`Routes` is the container; each `Route` maps a URL path to a component.

```tsx
// App.tsx
import { Routes, Route } from "react-router-dom";
import ShipmentList from "./pages/ShipmentList";
import TrackingPage from "./pages/TrackingPage";
import Dashboard from "./pages/Dashboard";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Dashboard />} />
      <Route path="/shipments" element={<ShipmentList />} />
      <Route path="/tracking" element={<TrackingPage />} />
    </Routes>
  );
}
```

**Spring equivalent:**

```java
@Controller
public class ShipmentController {
    @GetMapping("/")            // → <Route path="/" ...>
    public String dashboard() { return "dashboard"; }

    @GetMapping("/shipments")   // → <Route path="/shipments" ...>
    public String list() { return "shipment-list"; }
}
```

### 3.3 `Link`

A declarative anchor tag that navigates **without** a full page reload.

```tsx
import { Link } from "react-router-dom";

function Navbar() {
  return (
    <nav>
      <Link to="/">Dashboard</Link>
      <Link to="/shipments">Shipments</Link>
      <Link to="/tracking">Tracking</Link>
    </nav>
  );
}
```

> [!warning] Never Use Plain `<a>` Tags for Internal Navigation
> `<a href="/shipments">` causes a full page reload, defeating the SPA purpose.
> Always use `<Link>` for internal routes. Think of it this way: `<a>` is sending a
> package via an external carrier, `<Link>` is moving it on your internal conveyor belt.

### 3.4 `NavLink`

Like `Link`, but automatically applies an "active" CSS class when the route matches.
Perfect for navigation menus where you highlight the current page.

```tsx
import { NavLink } from "react-router-dom";

function Sidebar() {
  return (
    <nav>
      <NavLink
        to="/shipments"
        className={({ isActive }) => isActive ? "nav-active" : "nav-link"}
      >
        Shipments
      </NavLink>
      <NavLink
        to="/warehouses"
        className={({ isActive }) => isActive ? "nav-active" : "nav-link"}
      >
        Warehouses
      </NavLink>
    </nav>
  );
}
```

### 3.5 `Outlet`

A placeholder inside a parent route that renders matched child routes.
Think of it as a **loading dock** — the outer structure (warehouse) stays the same,
but different cargo (child components) arrives at the dock.

```tsx
// We'll cover this in detail under Nested Routes (Phase 6)
```

---

## Phase 4 — Route Parameters (`useParams`)

Just like `@PathVariable` in Spring, React Router supports dynamic URL segments.

```
Spring:   @GetMapping("/shipments/{id}")
React:    <Route path="/shipments/:id" element={<ShipmentDetails />} />
```

### Defining the Route

```tsx
<Routes>
  <Route path="/shipments" element={<ShipmentList />} />
  <Route path="/shipments/:shipmentId" element={<ShipmentDetails />} />
</Routes>
```

### Reading the Parameter

```tsx
import { useParams } from "react-router-dom";

function ShipmentDetails() {
  // Extract :shipmentId from the URL
  const { shipmentId } = useParams<{ shipmentId: string }>();

  return (
    <div>
      <h2>Shipment Details</h2>
      <p>Viewing shipment: <strong>{shipmentId}</strong></p>
      {/* Fetch shipment data using shipmentId */}
    </div>
  );
}
```

| Spring Boot                                    | React Router                        |
| ---------------------------------------------- | ----------------------------------- |
| `@PathVariable Long id`                        | `useParams()` → `{ id: "123" }`    |
| Type is declared (`Long`, `String`)            | Always returns `string`             |
| Extracted in method signature                  | Extracted via hook inside component |
| `@GetMapping("/shipments/{id}")`               | `path="/shipments/:id"`             |

> [!warning] Always Parse Numeric Params
> `useParams()` returns **strings**. If you need a number:
> ```tsx
> const id = Number(shipmentId); // or parseInt(shipmentId, 10)
> ```

---

## Phase 5 — Query Parameters (`useSearchParams`)

Query parameters are the key-value pairs after `?` in the URL.

```
URL:     /shipments?status=in-transit&carrier=FedEx
Spring:  @RequestParam String status, @RequestParam String carrier
React:   useSearchParams()
```

```tsx
import { useSearchParams } from "react-router-dom";

function ShipmentList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const status = searchParams.get("status");     // "in-transit"
  const carrier = searchParams.get("carrier");    // "FedEx"

  // Update query params (e.g., when user selects a filter)
  function filterByStatus(newStatus: string) {
    setSearchParams({ status: newStatus });
  }

  return (
    <div>
      <h2>Shipments {status && `— ${status}`}</h2>
      <button onClick={() => filterByStatus("delivered")}>Show Delivered</button>
      <button onClick={() => filterByStatus("in-transit")}>Show In Transit</button>
      {/* Render shipments filtered by status/carrier */}
    </div>
  );
}
```

> [!tip] When to Use Path vs Query Params
> - **Path params** (`/shipments/:id`) → identify a **specific resource** (like a BOL number)
> - **Query params** (`?status=delivered`) → **filter/sort** a collection (like filtering by status)
>
> Same rule as in Spring — `@PathVariable` for identity, `@RequestParam` for filtering.

---

## Phase 6 — Nested Routes and Layouts (`Outlet`)

Nested routes let you build **shared layouts**. The parent renders common UI (sidebar,
header), and child routes render into an `<Outlet>`.

> [!info] Logistics Analogy
> A **nested route** is like a warehouse with multiple loading bays. The warehouse
> (layout) is always there — roof, walls, office. But each bay (`Outlet`) handles
> a different shipment (child route) at any given time.

```
┌──────────────────────────────────────────┐
│  DashboardLayout (always rendered)       │
│  ┌────────┐  ┌─────────────────────────┐ │
│  │Sidebar │  │  <Outlet />             │ │
│  │        │  │                         │ │
│  │ • Home │  │  (child route renders   │ │
│  │ • Ship │  │   here based on URL)    │ │
│  │ • Track│  │                         │ │
│  │ • Ware │  │                         │ │
│  └────────┘  └─────────────────────────┘ │
└──────────────────────────────────────────┘
```

### Layout Component

```tsx
import { Outlet, NavLink } from "react-router-dom";

function DashboardLayout() {
  return (
    <div style={{ display: "flex" }}>
      <nav style={{ width: "200px", borderRight: "1px solid #ccc" }}>
        <h3>🚚 Logistics Hub</h3>
        <NavLink to="/dashboard">Overview</NavLink>
        <NavLink to="/dashboard/shipments">Shipments</NavLink>
        <NavLink to="/dashboard/warehouses">Warehouses</NavLink>
        <NavLink to="/dashboard/carriers">Carriers</NavLink>
      </nav>
      <main style={{ flex: 1, padding: "1rem" }}>
        <Outlet />  {/* Child routes render here */}
      </main>
    </div>
  );
}
```

### Route Configuration

```tsx
<Routes>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<Overview />} />            {/* /dashboard        */}
    <Route path="shipments" element={<ShipmentList />} />   {/* /dashboard/shipments */}
    <Route path="warehouses" element={<WarehouseList />} /> {/* /dashboard/warehouses */}
    <Route path="carriers" element={<CarrierList />} />     {/* /dashboard/carriers   */}
  </Route>
</Routes>
```

> [!tip] `index` Route
> The `index` route renders when the parent path matches exactly — no extra segment.
> It's the **default cargo** loaded at the dock when no specific shipment is requested.

---

## Phase 7 — Programmatic Navigation (`useNavigate`)

Sometimes you need to navigate **in code** — after a form submission, on a button click,
or after an API call completes. This is like `RedirectView` or `redirect:` in Spring MVC.

```java
// Spring equivalent
@PostMapping("/shipments")
public String createShipment(Shipment s) {
    shipmentService.save(s);
    return "redirect:/shipments/" + s.getId();
}
```

```tsx
import { useNavigate } from "react-router-dom";

function CreateShipmentForm() {
  const navigate = useNavigate();

  async function handleSubmit(event: React.FormEvent) {
    event.preventDefault();
    const newShipment = await api.createShipment(formData);

    // Navigate to the new shipment's detail page
    navigate(`/shipments/${newShipment.id}`);
  }

  // Go back (like browser back button)
  function handleCancel() {
    navigate(-1);
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
      <button type="submit">Create Shipment</button>
      <button type="button" onClick={handleCancel}>Cancel</button>
    </form>
  );
}
```

| Action                | Spring MVC                       | React Router                 |
| --------------------- | -------------------------------- | ---------------------------- |
| Redirect after save   | `return "redirect:/path";`       | `navigate("/path")`          |
| Redirect with replace | `RedirectView(url, true)`        | `navigate("/path", { replace: true })` |
| Go back               | N/A (server-side)                | `navigate(-1)`               |
| Pass state            | Flash attributes                 | `navigate("/path", { state: { ... } })` |

---

## Phase 8 — Protected Routes (Auth Guards)

In Spring Security, you protect endpoints with filters and `@PreAuthorize`.
In React, you protect routes by wrapping them in a **guard component**.

> [!info] Logistics Analogy
> A protected route is like a **customs checkpoint**. Every shipment (request) must
> show valid documentation (auth token) to pass. No papers? You're redirected back
> to the origin port (login page).

```
┌───────────────────────────────────────────────────────────┐
│                  Route Navigation Flow                     │
│                                                           │
│  User navigates to /dashboard/shipments                   │
│         │                                                 │
│         ▼                                                 │
│  ┌─────────────────┐                                      │
│  │ ProtectedRoute  │                                      │
│  │ isAuthenticated? │                                     │
│  └────┬───────┬────┘                                      │
│       │       │                                           │
│    ✅ Yes   ❌ No                                          │
│       │       │                                           │
│       ▼       ▼                                           │
│  <Dashboard> <Navigate to="/login" />                     │
└───────────────────────────────────────────────────────────┘
```

### The Guard Component

```tsx
import { Navigate, useLocation } from "react-router-dom";
import { useAuth } from "../hooks/useAuth";

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  const location = useLocation();

  if (!isAuthenticated) {
    // Redirect to login, preserving the attempted URL
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children;
}
```

### Using the Guard

```tsx
<Routes>
  {/* Public routes */}
  <Route path="/login" element={<LoginPage />} />

  {/* Protected routes — must be authenticated */}
  <Route
    path="/dashboard"
    element={
      <ProtectedRoute>
        <DashboardLayout />
      </ProtectedRoute>
    }
  >
    <Route index element={<Overview />} />
    <Route path="shipments" element={<ShipmentList />} />
    <Route path="shipments/:id" element={<ShipmentDetails />} />
  </Route>
</Routes>
```

### Minimal `useAuth` Hook

```tsx
// hooks/useAuth.ts
import { createContext, useContext, useState } from "react";

interface AuthContextType {
  isAuthenticated: boolean;
  login: (token: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType>(null!);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  const login = (token: string) => {
    localStorage.setItem("token", token);
    setIsAuthenticated(true);
  };

  const logout = () => {
    localStorage.removeItem("token");
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider value={{ isAuthenticated, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

> [!warning] This Is Frontend-Only Protection
> A `ProtectedRoute` hides UI, but **does not secure your API**. Always validate
> JWTs server-side in your Spring Security filter chain too. Frontend guards are
> UX helpers, not security boundaries.

---

## Phase 9 — 404 Not Found Route

Use a `path="*"` catch-all route at the end of your route config. This matches
any URL that didn't match earlier routes — like a default case in a `switch` statement
or a `@ExceptionHandler` for `NoHandlerFoundException` in Spring.

```tsx
function NotFoundPage() {
  return (
    <div style={{ textAlign: "center", padding: "4rem" }}>
      <h1>404 — Shipment Not Found</h1>
      <p>The route you requested doesn't exist in our logistics network.</p>
      <Link to="/">Return to Dashboard</Link>
    </div>
  );
}
```

```tsx
<Routes>
  <Route path="/" element={<Dashboard />} />
  <Route path="/shipments" element={<ShipmentList />} />
  <Route path="/shipments/:id" element={<ShipmentDetails />} />

  {/* Catch-all — MUST be last */}
  <Route path="*" element={<NotFoundPage />} />
</Routes>
```

> [!tip] Always Place `path="*"` Last
> Routes are matched top-down. The catch-all must be the final route,
> like the **undeliverable mail** department — it only handles what
> no other route could claim.

---

## Phase 10 — Route-Based Code Splitting

Large apps shouldn't load every page upfront. Use `React.lazy()` and `<Suspense>`
to split routes into separate bundles that load on demand.

> [!info] Logistics Analogy
> Code splitting is like **just-in-time delivery**. Instead of shipping every product
> to the store upfront (increasing initial load time), you deliver each product only
> when a customer asks for it.

```tsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

// Lazy-loaded pages — each becomes its own JS chunk
const ShipmentList = lazy(() => import("./pages/ShipmentList"));
const ShipmentDetails = lazy(() => import("./pages/ShipmentDetails"));
const WarehouseList = lazy(() => import("./pages/WarehouseList"));
const CarrierList = lazy(() => import("./pages/CarrierList"));
const TrackingPage = lazy(() => import("./pages/TrackingPage"));

function App() {
  return (
    <Suspense fallback={<div>Loading shipment data...</div>}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/shipments" element={<ShipmentList />} />
        <Route path="/shipments/:id" element={<ShipmentDetails />} />
        <Route path="/warehouses" element={<WarehouseList />} />
        <Route path="/carriers" element={<CarrierList />} />
        <Route path="/tracking" element={<TrackingPage />} />
        <Route path="*" element={<NotFoundPage />} />
      </Routes>
    </Suspense>
  );
}
```

**Bundle impact:**

```
Without code splitting:
  bundle.js ........... 850 KB  (everything loaded upfront)

With code splitting:
  main.js ............. 120 KB  (core + dashboard)
  ShipmentList.js .....  95 KB  (loaded when visiting /shipments)
  ShipmentDetails.js ..  80 KB  (loaded when visiting /shipments/:id)
  WarehouseList.js ....  60 KB  (loaded when visiting /warehouses)
  ...
```

---

## Phase 11 — Full Example: Logistics Dashboard

Bringing everything together — a complete routing setup for a logistics dashboard application.

### File Structure

```
src/
├── main.tsx                  # Entry point with BrowserRouter
├── App.tsx                   # Route definitions
├── hooks/
│   └── useAuth.ts            # Auth context & hook
├── components/
│   ├── ProtectedRoute.tsx    # Auth guard
│   └── DashboardLayout.tsx   # Sidebar layout with Outlet
└── pages/
    ├── LoginPage.tsx
    ├── Overview.tsx
    ├── ShipmentList.tsx
    ├── ShipmentDetails.tsx
    ├── WarehouseList.tsx
    ├── CarrierList.tsx
    └── NotFoundPage.tsx
```

### `main.tsx`

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import { AuthProvider } from "./hooks/useAuth";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </React.StrictMode>
);
```

### `App.tsx`

```tsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";
import ProtectedRoute from "./components/ProtectedRoute";
import DashboardLayout from "./components/DashboardLayout";
import LoginPage from "./pages/LoginPage";
import NotFoundPage from "./pages/NotFoundPage";

const Overview = lazy(() => import("./pages/Overview"));
const ShipmentList = lazy(() => import("./pages/ShipmentList"));
const ShipmentDetails = lazy(() => import("./pages/ShipmentDetails"));
const WarehouseList = lazy(() => import("./pages/WarehouseList"));
const CarrierList = lazy(() => import("./pages/CarrierList"));

export default function App() {
  return (
    <Suspense fallback={<div className="loading">Loading module...</div>}>
      <Routes>
        {/* Public */}
        <Route path="/login" element={<LoginPage />} />

        {/* Protected dashboard with nested routes */}
        <Route
          path="/dashboard"
          element={
            <ProtectedRoute>
              <DashboardLayout />
            </ProtectedRoute>
          }
        >
          <Route index element={<Overview />} />
          <Route path="shipments" element={<ShipmentList />} />
          <Route path="shipments/:shipmentId" element={<ShipmentDetails />} />
          <Route path="warehouses" element={<WarehouseList />} />
          <Route path="carriers" element={<CarrierList />} />
        </Route>

        {/* Catch-all */}
        <Route path="*" element={<NotFoundPage />} />
      </Routes>
    </Suspense>
  );
}
```

### `DashboardLayout.tsx`

```tsx
import { Outlet, NavLink } from "react-router-dom";
import { useAuth } from "../hooks/useAuth";

export default function DashboardLayout() {
  const { logout } = useAuth();

  const linkStyle = ({ isActive }: { isActive: boolean }) => ({
    display: "block",
    padding: "0.5rem 1rem",
    backgroundColor: isActive ? "#0d6efd" : "transparent",
    color: isActive ? "#fff" : "#333",
    textDecoration: "none",
    borderRadius: "4px",
    marginBottom: "0.25rem",
  });

  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      {/* Sidebar */}
      <aside style={{ width: "220px", background: "#f4f4f4", padding: "1rem" }}>
        <h2>🚚 LogiTrack</h2>
        <nav>
          <NavLink to="/dashboard" end style={linkStyle}>📊 Overview</NavLink>
          <NavLink to="/dashboard/shipments" style={linkStyle}>📦 Shipments</NavLink>
          <NavLink to="/dashboard/warehouses" style={linkStyle}>🏭 Warehouses</NavLink>
          <NavLink to="/dashboard/carriers" style={linkStyle}>🚛 Carriers</NavLink>
        </nav>
        <hr />
        <button onClick={logout}>Logout</button>
      </aside>

      {/* Main content — child routes render here */}
      <main style={{ flex: 1, padding: "1.5rem" }}>
        <Outlet />
      </main>
    </div>
  );
}
```

### `ShipmentDetails.tsx`

```tsx
import { useParams, useNavigate } from "react-router-dom";
import { useEffect, useState } from "react";

interface Shipment {
  id: string;
  origin: string;
  destination: string;
  status: string;
  carrier: string;
  estimatedDelivery: string;
}

export default function ShipmentDetails() {
  const { shipmentId } = useParams<{ shipmentId: string }>();
  const navigate = useNavigate();
  const [shipment, setShipment] = useState<Shipment | null>(null);

  useEffect(() => {
    // Fetch shipment by ID from your Spring Boot API
    fetch(`/api/shipments/${shipmentId}`)
      .then((res) => res.json())
      .then(setShipment)
      .catch(() => navigate("/dashboard/shipments"));
  }, [shipmentId, navigate]);

  if (!shipment) return <p>Loading shipment {shipmentId}...</p>;

  return (
    <div>
      <button onClick={() => navigate(-1)}>← Back to list</button>
      <h2>Shipment {shipment.id}</h2>
      <table>
        <tbody>
          <tr><td><strong>Origin</strong></td><td>{shipment.origin}</td></tr>
          <tr><td><strong>Destination</strong></td><td>{shipment.destination}</td></tr>
          <tr><td><strong>Status</strong></td><td>{shipment.status}</td></tr>
          <tr><td><strong>Carrier</strong></td><td>{shipment.carrier}</td></tr>
          <tr><td><strong>ETA</strong></td><td>{shipment.estimatedDelivery}</td></tr>
        </tbody>
      </table>
    </div>
  );
}
```

---

## Phase 12 — React Router vs Angular Router

If you're also exploring [[Angular Routing]], here's how the two compare:

| Feature                 | React Router v6                   | Angular Router                       |
| ----------------------- | --------------------------------- | ------------------------------------ |
| Installation            | `npm i react-router-dom`          | Built-in (`@angular/router`)         |
| Route definition        | JSX `<Route>` components          | TypeScript array config              |
| Dynamic params          | `:id` + `useParams()`             | `:id` + `ActivatedRoute`            |
| Query params            | `useSearchParams()`               | `ActivatedRoute.queryParams`         |
| Navigation              | `useNavigate()`                   | `Router.navigate()`                  |
| Guards                  | Wrapper components                | `canActivate` / `canDeactivate`      |
| Lazy loading            | `React.lazy()` + `Suspense`       | `loadChildren` / `loadComponent`     |
| Nested routes           | `<Outlet />`                      | `<router-outlet>`                    |
| Active link styling     | `<NavLink>` with `isActive`       | `routerLinkActive` directive         |
| Type safety             | Manual with generics               | Built-in with strict typing          |

> [!tip] Key Takeaway
> React Router is **more flexible** (it's just components and hooks), while Angular Router
> is **more structured** (configuration-driven with built-in guards). Coming from Spring,
> Angular's approach may feel more familiar — but React's approach is simpler to get started with.

---

## Quick Reference Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│               React Router v6 — Quick Reference              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SETUP         BrowserRouter wraps entire app                │
│  DEFINE        <Route path="/x" element={<X />} />           │
│  LINK          <Link to="/x">Go</Link>                       │
│  ACTIVE LINK   <NavLink to="/x">Go</NavLink>                 │
│  PARAMS        /shipments/:id  → useParams()                 │
│  QUERY         ?status=open    → useSearchParams()           │
│  NAVIGATE      useNavigate() for programmatic redirect       │
│  NESTED        Parent <Outlet /> renders child routes        │
│  GUARD         Wrap route element in <ProtectedRoute>        │
│  404           <Route path="*" element={<NotFound />} />     │
│  LAZY          React.lazy(() => import("./Page"))            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Related Notes

- [[React Fundamentals]] — Core React concepts (components, state, hooks)
- [[Angular Routing]] — Angular's approach to routing for comparison
- [[React vs Angular Comparison]] — High-level framework comparison

---

*Next up: Learn how to fetch data inside routed components using [[React Fundamentals]] hooks like `useEffect` and libraries like TanStack Query.*
