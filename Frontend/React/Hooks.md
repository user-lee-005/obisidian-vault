---
tags:
  - react
  - hooks
  - frontend
  - typescript
  - beginner
created: 2025-07-14
related:
  - "[[State and Lifecycle]]"
  - "[[Components and Props]]"
  - "[[React Performance Optimization]]"
  - "[[React State Management]]"
---

# React Hooks

> *"Hooks let you use state and other React features without writing a class. They're functions that let you 'hook into' React state and lifecycle from functional components."* — React Documentation

---

## Phase 1: What Are Hooks?

### The Big Picture

Before Hooks (React 16.8, Feb 2019), you **had** to use class components to manage state or run side effects. Hooks eliminated that requirement — you can now do everything in **functional components**.

**Analogy for a Spring Boot developer:**

Think of Hooks as **Spring annotations for React**. In Spring, you don't manually wire beans or manage lifecycle — annotations do it for you:

| Spring Boot | React Hooks | What It Does |
|---|---|---|
| `@Autowired` | `useContext()` | Injects a dependency / shared value |
| `@PostConstruct` | `useEffect(() => {}, [])` | Runs code after initialization |
| `@PreDestroy` | `useEffect` cleanup (`return () => {}`) | Runs cleanup before teardown |
| `@Value("${prop}")` | `useState()` | Holds a value that can change |
| `@Cacheable` | `useMemo()` | Caches an expensive computation |
| `@EventListener` | Event handlers (`onClick`) | Responds to events |

```
Spring Boot Class                    React Functional Component
┌──────────────────────┐             ┌──────────────────────────┐
│ @Service             │             │ function ShipmentPage()  │
│ public class Svc {   │             │ {                        │
│                      │             │                          │
│   @Autowired         │  ≈ same as  │   const ctx = useContext  │
│   private Repo repo; │ ──────────→ │     (ShipmentContext);   │
│                      │             │                          │
│   @PostConstruct     │  ≈ same as  │   useEffect(() => {      │
│   void init() {..}   │ ──────────→ │     fetchData();         │
│                      │             │   }, []);                │
│   @PreDestroy        │  ≈ same as  │   useEffect(() => {      │
│   void cleanup() {}  │ ──────────→ │     return () => close();│
│ }                    │             │   }, []);                │
└──────────────────────┘             └──────────────────────────┘
```

### The Core Hooks at a Glance

```
Built-in Hooks
├── State
│   ├── useState        → Simple state (a variable that triggers re-render)
│   └── useReducer      → Complex state with transitions (like a state machine)
├── Side Effects
│   └── useEffect       → Run code after render (API calls, subscriptions)
├── Context
│   └── useContext       → Read shared data from a parent provider
├── Refs
│   └── useRef          → Mutable value that persists without re-rendering
├── Performance
│   ├── useMemo         → Memoize expensive computations
│   └── useCallback     → Memoize function references
└── Custom Hooks
    └── useAnything()   → YOUR reusable stateful logic
```

### Rules of Hooks

There are exactly **two rules** you must follow. Break them and React breaks.

> [!warning] Rules of Hooks — Non-Negotiable
> 1. **Only call Hooks at the TOP LEVEL** — never inside loops, conditions, or nested functions
> 2. **Only call Hooks from React functions** — either functional components or custom Hooks

**Why the top-level rule?** React relies on the **order** of Hook calls to match state to the right Hook. If you put a Hook inside an `if`, the order changes between renders, and React gets confused.

```tsx
// ❌ WRONG — conditional Hook call
function ShipmentTracker({ trackingId }: { trackingId: string }) {
  if (trackingId) {
    // React can't guarantee this runs on every render
    const [status, setStatus] = useState('pending');
  }
}

// ✅ CORRECT — Hook at top level, condition inside
function ShipmentTracker({ trackingId }: { trackingId: string }) {
  const [status, setStatus] = useState('pending');

  useEffect(() => {
    if (trackingId) {
      // Condition INSIDE the Hook — perfectly fine
      fetchStatus(trackingId).then(setStatus);
    }
  }, [trackingId]);
}
```

> [!tip] ESLint Plugin
> Install `eslint-plugin-react-hooks` — it enforces both rules automatically. The React team considers this **essential**, not optional.

---

## Phase 2: useEffect — Side Effects

### Why useEffect Matters

`useEffect` is the **most important Hook** after `useState`. It handles everything that isn't pure rendering: API calls, subscriptions, timers, DOM manipulation, logging.

**Analogy:** In Spring Boot, `@PostConstruct` runs setup after bean creation, and `@PreDestroy` cleans up before shutdown. `useEffect` does **both** — plus it can re-run whenever specific values change, like a `@Scheduled` method that watches a condition.

### The Four Patterns

```tsx
// Pattern 1: Run after EVERY render (rarely needed)
useEffect(() => {
  console.log('Component rendered');
});

// Pattern 2: Run ONCE on mount (like @PostConstruct)
useEffect(() => {
  console.log('Component mounted — fetch initial data');
}, []);  // Empty array = no dependencies = run once

// Pattern 3: Run when SPECIFIC values change
useEffect(() => {
  console.log(`Carrier changed to: ${selectedCarrier}`);
  fetchCarrierDetails(selectedCarrier);
}, [selectedCarrier]);  // Re-runs only when selectedCarrier changes

// Pattern 4: Run on mount + CLEANUP on unmount (like @PreDestroy)
useEffect(() => {
  const ws = new WebSocket('wss://tracking.api/live');
  ws.onmessage = (msg) => updatePosition(JSON.parse(msg.data));

  // Cleanup function — called on unmount OR before the effect re-runs
  return () => {
    ws.close();
    console.log('WebSocket connection closed');
  };
}, []);
```

### How the Dependency Array Works

```
Dependency Array          When Effect Runs               Spring Equivalent
─────────────────────────────────────────────────────────────────────────
(none / omitted)          After EVERY render              @Scheduled(fixedRate=0)
[]                        Once on mount only              @PostConstruct
[dep1]                    On mount + when dep1 changes    @EventListener for dep1
[dep1, dep2]              On mount + when either changes  @EventListener for both
```

```
                    Component Lifecycle with useEffect
                    ══════════════════════════════════

    Mount                    Update (deps changed)         Unmount
    ─────                    ─────────────────────         ───────
      │                            │                         │
      ▼                            ▼                         │
 ┌─────────┐               ┌─────────────┐                  │
 │ Render  │               │ Re-render   │                  │
 └────┬────┘               └──────┬──────┘                  │
      │                           │                         │
      ▼                           ▼                         ▼
 ┌─────────────────┐   ┌──────────────────────┐   ┌────────────────┐
 │ Run effect      │   │ Run CLEANUP of prev  │   │ Run CLEANUP    │
 │ (setup function)│   │ effect first, then   │   │ of last effect │
 └─────────────────┘   │ run new effect       │   └────────────────┘
                        └──────────────────────┘
```

### Real Example: Fetching Shipments from API

This is the pattern you'll use **daily** — fetching data when a component mounts.

```tsx
import { useState, useEffect } from 'react';

// Type definition — like a Java DTO
interface Shipment {
  id: string;
  trackingId: string;
  origin: string;
  destination: string;
  status: 'pending' | 'in-transit' | 'delivered' | 'cancelled';
  carrier: string;
  estimatedDelivery: string;
}

function ShipmentList() {
  // State — like fields in a Spring @Component
  const [shipments, setShipments] = useState<Shipment[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  // useEffect — like @PostConstruct in Spring Boot
  useEffect(() => {
    // We define the async function INSIDE the effect
    // because useEffect itself can't be async
    const fetchShipments = async () => {
      try {
        const response = await fetch('/api/shipments');

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: Failed to fetch shipments`);
        }

        const data: Shipment[] = await response.json();
        setShipments(data);      // Update state → triggers re-render
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);       // Always stop loading spinner
      }
    };

    fetchShipments();
  }, []);  // Empty deps = run once on mount, like @PostConstruct

  // Render based on state
  if (loading) return <p>Loading shipments...</p>;
  if (error) return <p className="error">Error: {error}</p>;

  return (
    <ul>
      {shipments.map((shipment) => (
        <li key={shipment.id}>
          {shipment.trackingId} — {shipment.origin} → {shipment.destination}
          ({shipment.status})
        </li>
      ))}
    </ul>
  );
}
```

> [!tip] Why not make useEffect async directly?
> `useEffect` expects its callback to return either `void` or a cleanup function. An `async` function returns a `Promise`, which breaks this contract. Always define the async function **inside** and call it immediately.

### Common Mistakes with useEffect

#### Mistake 1: Infinite Loop

```tsx
// ❌ INFINITE LOOP — sets state on every render, which triggers re-render
useEffect(() => {
  setShipments(fetchAllShipments());  // This triggers re-render
});  // No dependency array = runs EVERY render = infinite loop

// ✅ FIX — add the dependency array
useEffect(() => {
  fetchAllShipments().then(setShipments);
}, []);  // Runs once
```

#### Mistake 2: Missing Dependencies

```tsx
// ❌ BUG — carrierId changes but effect doesn't re-run
useEffect(() => {
  fetch(`/api/shipments?carrier=${carrierId}`)  // Uses carrierId
    .then(res => res.json())
    .then(setShipments);
}, []);  // carrierId not listed — effect is stale!

// ✅ FIX — include carrierId in dependencies
useEffect(() => {
  fetch(`/api/shipments?carrier=${carrierId}`)
    .then(res => res.json())
    .then(setShipments);
}, [carrierId]);  // Re-runs whenever carrierId changes
```

#### Mistake 3: Forgetting Cleanup

```tsx
// ❌ MEMORY LEAK — WebSocket stays open after component unmounts
useEffect(() => {
  const ws = new WebSocket('wss://tracking.live/updates');
  ws.onmessage = (e) => setPosition(JSON.parse(e.data));
}, []);

// ✅ FIX — return a cleanup function
useEffect(() => {
  const ws = new WebSocket('wss://tracking.live/updates');
  ws.onmessage = (e) => setPosition(JSON.parse(e.data));

  return () => ws.close();  // Cleanup on unmount
}, []);
```

#### Mistake 4: Stale Closures

```tsx
// ❌ BUG — count is captured at the time the interval is created
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);  // Always adds 1 to the INITIAL count (stale)
  }, 1000);
  return () => clearInterval(interval);
}, []);

// ✅ FIX — use functional update to always get the latest value
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1);  // Always uses the current value
  }, 1000);
  return () => clearInterval(interval);
}, []);
```

---

## Phase 3: useContext — Sharing Global Data

### The Problem Context Solves

Without context, passing data through many layers of components creates **"prop drilling"** — the React equivalent of passing a parameter through 5 method signatures in Java just to reach the one method that needs it.

```
Prop Drilling (BAD)                     Context (GOOD)
─────────────────                       ──────────────
App (has user data)                     App (provides UserContext)
  └── Layout (passes user)                └── Layout (no props needed)
        └── Sidebar (passes user)               └── Sidebar
              └── UserMenu (uses user)                └── UserMenu (consumes UserContext)

Every intermediate component must                Only the component that NEEDS it
accept and pass the prop — even if               reads from context directly.
it doesn't use it.
```

**Spring Boot Analogy:** `useContext` is like `@Autowired`. You don't pass the `CarrierService` through every method call — Spring injects it wherever it's needed. Context works the same way.

### Three Steps: Create → Provide → Consume

```tsx
// ──────────────────────────────────────
// Step 1: CREATE the context (like defining a @Bean)
// ──────────────────────────────────────
import { createContext, useContext, useState, ReactNode } from 'react';

interface User {
  id: string;
  name: string;
  role: 'admin' | 'dispatcher' | 'driver';
}

interface AuthContextType {
  user: User | null;
  login: (user: User) => void;
  logout: () => void;
  isAuthenticated: boolean;
}

// Create with a default value (used when no Provider is found above)
const AuthContext = createContext<AuthContextType | undefined>(undefined);

// ──────────────────────────────────────
// Step 2: PROVIDE the context (like @Configuration + @Bean)
// ──────────────────────────────────────
function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);

  const login = (userData: User) => setUser(userData);
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{
      user,
      login,
      logout,
      isAuthenticated: user !== null
    }}>
      {children}
    </AuthContext.Provider>
  );
}

// ──────────────────────────────────────
// Step 3: CONSUME the context (like @Autowired)
// ──────────────────────────────────────
function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}

// Usage in any nested component — no prop drilling!
function ShipmentActions() {
  const { user, isAuthenticated } = useAuth();

  if (!isAuthenticated) return <p>Please log in to manage shipments.</p>;

  return (
    <div>
      <p>Welcome, {user!.name} ({user!.role})</p>
      {user!.role === 'admin' && <button>Delete Shipment</button>}
    </div>
  );
}
```

> [!warning] When NOT to Use Context
> Context re-renders **every component** that consumes it when the value changes. If your context value changes frequently (e.g., mouse position, live tracking coordinates), every consumer re-renders — even if they only need one field.
>
> **Solutions:**
> - Split into multiple smaller contexts
> - Use a state management library like [[React State Management|Zustand or Redux]]
> - Use `useMemo` on the provider value to reduce unnecessary re-renders

---

## Phase 4: useRef — Persistent Mutable Values

### What useRef Does

`useRef` gives you a **mutable box** that:
1. **Persists** across re-renders (unlike a local variable)
2. **Does NOT trigger re-renders** when changed (unlike `useState`)

**Analogy:** Think of a warehouse clipboard. Workers write notes on it throughout the day (mutable), it stays at the same station (persistent), but writing on it doesn't trigger an inventory recount (no re-render).

### Use Case 1: DOM Element References

```tsx
import { useRef, useEffect } from 'react';

function TrackingIdInput() {
  // Create a ref — like a pointer to a DOM element
  const inputRef = useRef<HTMLInputElement>(null);

  // Auto-focus the input on mount
  useEffect(() => {
    inputRef.current?.focus();  // Access the actual DOM element
  }, []);

  return (
    <div>
      <label>Tracking ID:</label>
      {/* Attach the ref to the element */}
      <input ref={inputRef} type="text" placeholder="Enter tracking ID" />
      <button onClick={() => inputRef.current?.select()}>
        Select All
      </button>
    </div>
  );
}
```

### Use Case 2: Tracking Previous Values

```tsx
function ShipmentStatusDisplay({ status }: { status: string }) {
  const prevStatusRef = useRef<string>(status);

  useEffect(() => {
    // After render, store the current status for next comparison
    prevStatusRef.current = status;
  });

  return (
    <div>
      <p>Current status: {status}</p>
      {prevStatusRef.current !== status && (
        <p>Changed from: {prevStatusRef.current}</p>
      )}
    </div>
  );
}
```

### Use Case 3: Interval/Timer IDs

```tsx
function LiveTrackingPoller({ shipmentId }: { shipmentId: string }) {
  const [position, setPosition] = useState({ lat: 0, lng: 0 });
  const intervalRef = useRef<number | null>(null);

  const startPolling = () => {
    // Store the interval ID in a ref — we need it to stop later
    // but changing it should NOT re-render the component
    intervalRef.current = window.setInterval(async () => {
      const res = await fetch(`/api/tracking/${shipmentId}/position`);
      const data = await res.json();
      setPosition(data);
    }, 5000);
  };

  const stopPolling = () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  // Cleanup on unmount
  useEffect(() => {
    return () => stopPolling();
  }, []);

  return (
    <div>
      <p>Lat: {position.lat}, Lng: {position.lng}</p>
      <button onClick={startPolling}>Start Live Tracking</button>
      <button onClick={stopPolling}>Stop</button>
    </div>
  );
}
```

> [!tip] useState vs useRef — When to Use Which
> - Need the UI to **update** when the value changes? → `useState`
> - Need to **store a value** that doesn't affect rendering? → `useRef`
> - Need a **reference to a DOM element**? → `useRef`

---

## Phase 5: useMemo and useCallback — Performance

### The Re-Rendering Problem

React re-renders a component whenever its state or props change. During a re-render, **every variable and function inside the component is recreated**. For most cases, this is fine — JavaScript is fast. But for **expensive computations** or **large lists**, it matters.

**Analogy:** Imagine a warehouse where every time a new package arrives, workers re-sort the **entire inventory** — even if the new package goes in one spot. `useMemo` is like saying "only re-sort the section that changed."

### useMemo — Memoize Expensive Computations

```tsx
import { useState, useMemo } from 'react';

interface Shipment {
  id: string;
  trackingId: string;
  status: string;
  carrier: string;
  weight: number;
}

function ShipmentDashboard({ shipments }: { shipments: Shipment[] }) {
  const [selectedStatus, setSelectedStatus] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');

  // ❌ WITHOUT useMemo — filters run on EVERY re-render
  // Even when the user is just typing in a search box (doesn't affect filter)
  // const filteredShipments = shipments.filter(s =>
  //   selectedStatus === 'all' || s.status === selectedStatus
  // );

  // ✅ WITH useMemo — only re-filters when shipments or selectedStatus change
  const filteredShipments = useMemo(() => {
    console.log('Filtering shipments...');  // You'll see this only when deps change
    return shipments.filter(s =>
      selectedStatus === 'all' || s.status === selectedStatus
    );
  }, [shipments, selectedStatus]);  // Recompute ONLY when these change

  // Another useMemo — aggregate stats from filtered results
  const stats = useMemo(() => ({
    total: filteredShipments.length,
    totalWeight: filteredShipments.reduce((sum, s) => sum + s.weight, 0),
    carriers: [...new Set(filteredShipments.map(s => s.carrier))].length,
  }), [filteredShipments]);

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search..."
      />
      <select
        value={selectedStatus}
        onChange={(e) => setSelectedStatus(e.target.value)}
      >
        <option value="all">All Statuses</option>
        <option value="in-transit">In Transit</option>
        <option value="delivered">Delivered</option>
      </select>

      <p>{stats.total} shipments | {stats.totalWeight}kg | {stats.carriers} carriers</p>
      {/* render filteredShipments */}
    </div>
  );
}
```

### useCallback — Memoize Function References

`useCallback` memoizes a **function itself** (not its result). This is important when passing callbacks to child components that use `React.memo`.

```tsx
import { useState, useCallback } from 'react';

function ShipmentManager() {
  const [shipments, setShipments] = useState<Shipment[]>([]);

  // ❌ WITHOUT useCallback — new function reference every render
  // This defeats React.memo on ShipmentRow because the prop changes every time
  // const handleDelete = (id: string) => {
  //   setShipments(prev => prev.filter(s => s.id !== id));
  // };

  // ✅ WITH useCallback — same function reference across renders
  const handleDelete = useCallback((id: string) => {
    setShipments(prev => prev.filter(s => s.id !== id));
  }, []);  // No deps — uses functional update, so always has latest state

  return (
    <div>
      {shipments.map(s => (
        <ShipmentRow key={s.id} shipment={s} onDelete={handleDelete} />
      ))}
    </div>
  );
}

// React.memo — only re-renders if props actually change
const ShipmentRow = React.memo(({ shipment, onDelete }: {
  shipment: Shipment;
  onDelete: (id: string) => void;
}) => {
  console.log(`Rendering row: ${shipment.trackingId}`);
  return (
    <div>
      <span>{shipment.trackingId}</span>
      <button onClick={() => onDelete(shipment.id)}>Delete</button>
    </div>
  );
});
```

> [!warning] Don't Overuse Memoization
> **Premature optimization is the root of all evil.** Only use `useMemo`/`useCallback` when:
> - You have a measurably **expensive computation** (filtering 10,000+ items)
> - You're passing callbacks to **memoized child components** (`React.memo`)
> - React DevTools Profiler shows **unnecessary re-renders**
>
> For simple operations on small datasets, the overhead of memoization can be **worse** than just recomputing.

See also: [[React Performance Optimization]] for profiling techniques and advanced patterns.

---

## Phase 6: useReducer — Complex State Logic

### When useState Isn't Enough

`useState` is great for simple, independent state values. But when you have **complex state with multiple related transitions**, `useReducer` gives you better structure.

**Analogy:** A shipment in logistics follows a strict state machine — it can only transition through specific statuses in specific ways. You wouldn't store that as a loose variable that anything can set to anything. You'd define a state machine with valid transitions. That's `useReducer`.

```
Shipment State Machine
═══════════════════════

  ┌──────────┐   MARK_SHIPPED   ┌────────────┐   MARK_DELIVERED   ┌───────────┐
  │ PENDING  │ ───────────────→ │ IN-TRANSIT │ ────────────────→ │ DELIVERED │
  └──────────┘                   └────────────┘                    └───────────┘
       │                              │
       │ CANCEL                       │ CANCEL
       ▼                              ▼
  ┌───────────┐                  ┌───────────┐
  │ CANCELLED │                  │ CANCELLED │
  └───────────┘                  └───────────┘
```

### Full useReducer Example

```tsx
import { useReducer } from 'react';

// ── Types ──────────────────────────────────────────────────
type ShipmentStatus = 'pending' | 'in-transit' | 'delivered' | 'cancelled';

interface ShipmentState {
  trackingId: string;
  status: ShipmentStatus;
  carrier: string;
  reason?: string;       // Only set if cancelled
  deliveredAt?: string;  // Only set if delivered
  history: string[];     // Audit trail of status changes
}

// ── Actions (like Command pattern in Java) ─────────────────
type ShipmentAction =
  | { type: 'MARK_SHIPPED'; carrier: string }
  | { type: 'MARK_DELIVERED' }
  | { type: 'CANCEL'; reason: string }
  | { type: 'RESET' };

// ── Reducer (pure function — like a switch-case service) ───
function shipmentReducer(state: ShipmentState, action: ShipmentAction): ShipmentState {
  const timestamp = new Date().toISOString();

  switch (action.type) {
    case 'MARK_SHIPPED':
      if (state.status !== 'pending') {
        console.warn(`Cannot ship: current status is ${state.status}`);
        return state;  // Invalid transition — return unchanged
      }
      return {
        ...state,
        status: 'in-transit',
        carrier: action.carrier,
        history: [...state.history, `${timestamp}: Shipped via ${action.carrier}`],
      };

    case 'MARK_DELIVERED':
      if (state.status !== 'in-transit') {
        console.warn(`Cannot deliver: current status is ${state.status}`);
        return state;
      }
      return {
        ...state,
        status: 'delivered',
        deliveredAt: timestamp,
        history: [...state.history, `${timestamp}: Delivered`],
      };

    case 'CANCEL':
      if (state.status === 'delivered' || state.status === 'cancelled') {
        console.warn(`Cannot cancel: current status is ${state.status}`);
        return state;
      }
      return {
        ...state,
        status: 'cancelled',
        reason: action.reason,
        history: [...state.history, `${timestamp}: Cancelled — ${action.reason}`],
      };

    case 'RESET':
      return { ...initialState, trackingId: state.trackingId };

    default:
      return state;
  }
}

// ── Initial State ──────────────────────────────────────────
const initialState: ShipmentState = {
  trackingId: 'SHIP-2024-001',
  status: 'pending',
  carrier: '',
  history: ['Created'],
};

// ── Component ──────────────────────────────────────────────
function ShipmentTracker() {
  // useReducer returns [currentState, dispatchFunction]
  // Like injecting a service that processes commands
  const [shipment, dispatch] = useReducer(shipmentReducer, initialState);

  return (
    <div>
      <h2>Shipment: {shipment.trackingId}</h2>
      <p>Status: <strong>{shipment.status}</strong></p>
      {shipment.carrier && <p>Carrier: {shipment.carrier}</p>}
      {shipment.reason && <p>Cancel reason: {shipment.reason}</p>}

      <div>
        <button
          onClick={() => dispatch({ type: 'MARK_SHIPPED', carrier: 'MAERSK' })}
          disabled={shipment.status !== 'pending'}
        >
          Ship via Maersk
        </button>
        <button
          onClick={() => dispatch({ type: 'MARK_DELIVERED' })}
          disabled={shipment.status !== 'in-transit'}
        >
          Mark Delivered
        </button>
        <button
          onClick={() => dispatch({ type: 'CANCEL', reason: 'Customer request' })}
          disabled={shipment.status === 'delivered' || shipment.status === 'cancelled'}
        >
          Cancel
        </button>
      </div>

      <h3>History</h3>
      <ul>
        {shipment.history.map((entry, i) => <li key={i}>{entry}</li>)}
      </ul>
    </div>
  );
}
```

### useState vs useReducer — Decision Table

| Criteria | `useState` | `useReducer` |
|---|---|---|
| **State shape** | Simple / primitive values | Objects with multiple related fields |
| **Transitions** | Set to any value freely | Defined, predictable transitions |
| **Number of state updates** | 1–3 per event | Multiple fields change together |
| **Business logic** | Minimal | Complex rules / validation |
| **Testing** | Test the component | Test the reducer as a pure function |
| **Spring analogy** | `@Value` field | Command pattern + state machine |

> [!tip] Rule of Thumb
> If you find yourself calling `setState` 3+ times in a single event handler to update related fields, that's a signal to switch to `useReducer`.

---

## Phase 7: Custom Hooks — Reusable Logic

### What Are Custom Hooks?

Custom Hooks are functions that **start with `use`** and encapsulate reusable stateful logic. They're React's answer to the DRY principle for component logic.

**Analogy:** In Spring Boot, you extract common logic into `@Service` classes that multiple controllers can inject. In React, you extract common logic into custom Hooks that multiple components can call.

### Naming Convention

```
useShipments()       → Hook that manages shipment data
useFetch()           → Hook that handles API fetching
useDebounce()        → Hook that debounces a value
useLocalStorage()    → Hook that syncs state with localStorage
useAuth()            → Hook that provides auth context
```

> [!tip] The `use` prefix is mandatory
> React uses the `use` prefix to identify Hooks and enforce the Rules of Hooks. Without it, the ESLint plugin can't check your code.

### Example 1: useFetch — Generic Data Fetching

```tsx
import { useState, useEffect } from 'react';

interface FetchResult<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
  refetch: () => void;
}

// Generic custom Hook — works with any API endpoint
function useFetch<T>(url: string): FetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [trigger, setTrigger] = useState(0);

  useEffect(() => {
    // AbortController — like a CancellationToken in .NET
    // Prevents setting state on an unmounted component
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then((json: T) => {
        setData(json);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err.message);
          setLoading(false);
        }
      });

    // Cleanup: abort the fetch if component unmounts or URL changes
    return () => controller.abort();
  }, [url, trigger]);

  const refetch = () => setTrigger(prev => prev + 1);

  return { data, loading, error, refetch };
}

// ── Usage — clean and reusable ─────────────────────────────
function ShipmentPage() {
  const { data: shipments, loading, error, refetch } = useFetch<Shipment[]>(
    '/api/shipments'
  );

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error} <button onClick={refetch}>Retry</button></p>;

  return (
    <ul>
      {shipments?.map(s => <li key={s.id}>{s.trackingId}</li>)}
    </ul>
  );
}

function CarrierPage() {
  // Same Hook, different endpoint — full reuse!
  const { data: carriers, loading } = useFetch<Carrier[]>('/api/carriers');
  // ...
}
```

### Example 2: useDebounce — Delay Search Input

```tsx
import { useState, useEffect } from 'react';

// Delays updating a value until the user stops changing it
// Like a buffer that flushes after a quiet period
function useDebounce<T>(value: T, delayMs: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs);
    return () => clearTimeout(timer);  // Reset timer if value changes before delay
  }, [value, delayMs]);

  return debouncedValue;
}

// ── Usage — search input that doesn't fire on every keystroke ──
function ShipmentSearch() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearch = useDebounce(searchTerm, 300);  // Wait 300ms

  // This effect only fires when debouncedSearch changes (after 300ms of no typing)
  const { data: results } = useFetch<Shipment[]>(
    `/api/shipments/search?q=${debouncedSearch}`
  );

  return (
    <input
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Search shipments..."
    />
  );
}
```

### Example 3: useLocalStorage — Persistent Preferences

```tsx
function useLocalStorage<T>(key: string, initialValue: T) {
  // Initialize state from localStorage (if available) or use default
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  // Update localStorage whenever the state changes
  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.warn(`Failed to save to localStorage: ${error}`);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue] as const;
}

// ── Usage ───────────────────────────────────────────────────
function DashboardSettings() {
  const [carrier, setCarrier] = useLocalStorage('preferredCarrier', 'MAERSK');
  const [pageSize, setPageSize] = useLocalStorage('pageSize', 25);
  // Values persist across page reloads!
}
```

---

## Phase 8: Hooks Cheat Sheet

### Quick Reference Table

| Hook | Purpose | When to Use | Spring Boot Equivalent |
|---|---|---|---|
| `useState` | Simple reactive state | Any component-level state | `@Value` / mutable field |
| `useEffect` | Side effects after render | API calls, subscriptions, timers | `@PostConstruct` / `@PreDestroy` / `@Scheduled` |
| `useContext` | Read shared context | Theme, auth, locale | `@Autowired` |
| `useRef` | Mutable value, no re-render | DOM refs, interval IDs, prev values | Instance variable (not `@State`) |
| `useMemo` | Memoize computation result | Expensive filters/transforms | `@Cacheable` |
| `useCallback` | Memoize function reference | Callbacks passed to memoized children | Method reference caching |
| `useReducer` | Complex state transitions | Multi-field state, state machines | Command pattern + service |
| Custom Hooks | Reusable stateful logic | Shared patterns across components | `@Service` / utility classes |

### Decision Flowchart

```
Do I need to store a value that affects the UI?
├── YES → Is the state logic complex with multiple related transitions?
│          ├── YES → useReducer
│          └── NO  → useState
├── NO → Do I need to run code after render? (API calls, subscriptions)
│         ├── YES → useEffect
│         └── NO  → Do I need a reference to a DOM element?
│                    ├── YES → useRef
│                    └── NO  → Do I need to read shared data from a parent?
│                               ├── YES → useContext
│                               └── NO  → Do I need to optimize performance?
│                                          ├── Expensive computation → useMemo
│                                          └── Callback to memo'd child → useCallback
```

### Rules Recap

> [!warning] The Two Rules — Always Remember
> 1. ✅ Call Hooks at the **top level** of your component or custom Hook
> 2. ✅ Call Hooks only from **React function components** or **custom Hooks**
> 3. ❌ Never call inside loops, conditions, or nested functions
> 4. ❌ Never call from regular JavaScript functions or class components

---

## Further Reading

- [[State and Lifecycle]] — Understanding React's render cycle
- [[Components and Props]] — Building blocks that use Hooks
- [[React Performance Optimization]] — Profiling and advanced memoization
- [[React State Management]] — When to go beyond built-in Hooks (Zustand, Redux)
