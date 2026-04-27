---
tags:
  - react
  - performance
  - optimization
  - memoization
  - frontend
  - phase4
created: 2025-07-18
---

# React Performance Optimization

> *"Premature optimization is the root of all evil — but so is shipping a warehouse system that takes 10 seconds to render a shipment list."* — Every frustrated logistics developer

---

## Why This Matters

Think of a **warehouse management dashboard** displaying thousands of shipments, tracking numbers, carrier statuses, and real-time ETAs. If React re-renders every single row every time one shipment status changes, your app crawls like a cargo ship through a canal blockage.

As a Spring Boot developer, you already know this instinct — you don't slap `@Transactional` on every method or cache every query. You **profile first**, then optimize the bottleneck. The same discipline applies to React.

> [!info] The Golden Rule
> **Don't optimize what you haven't measured.** React is fast by default. Only optimize when you see actual lag — just like you'd profile a Spring Boot app with JMX or Micrometer before tuning JPA queries.

```
┌─────────────────────────────────────────────────┐
│           OPTIMIZATION DECISION TREE            │
├─────────────────────────────────────────────────┤
│                                                 │
│   Is the UI visibly slow / laggy?               │
│        │                                        │
│        ├── NO  → Don't optimize. Ship it. ✅    │
│        │                                        │
│        └── YES → Profile with React DevTools    │
│              │                                  │
│              ├── Expensive computation?          │
│              │     └── useMemo                  │
│              │                                  │
│              ├── Unnecessary child re-renders?   │
│              │     └── React.memo + useCallback  │
│              │                                  │
│              ├── Huge list rendering?            │
│              │     └── Virtualization            │
│              │                                  │
│              └── Large bundle size?              │
│                    └── Code Splitting            │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Phase 1 — How React Renders (The Render Pipeline)

Before optimizing, understand what you're optimizing. In Spring Boot, you wouldn't tune Hibernate without understanding the entity lifecycle. Same idea here.

### The React Render Cycle

```
  State Change / Props Change
           │
           ▼
  ┌─────────────────┐
  │  Render Phase    │  ← React calls your component function
  │  (Pure, no side  │     Builds a new Virtual DOM tree
  │   effects)       │     This is like generating a new "shipping manifest"
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │  Reconciliation  │  ← React DIFFS old tree vs new tree
  │  (Diffing)       │     Finds what actually changed
  │                  │     Like comparing two BOL (Bill of Lading) versions
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │  Commit Phase    │  ← React applies ONLY the changes to real DOM
  │  (DOM Updates)   │     Minimal mutations
  │                  │     Like only updating the changed line items
  └─────────────────┘
```

> [!tip] Spring Boot Analogy
> Think of this like Hibernate's **dirty checking**. Hibernate doesn't write every entity to the DB on flush — it compares the snapshot to the current state and only generates UPDATE statements for changed fields. React's reconciliation does the same thing with the DOM.

### What Triggers a Re-Render?

| Trigger                    | Example                                       |
| -------------------------- | --------------------------------------------- |
| `setState` called          | `setShipments([...shipments, newShipment])`    |
| Parent re-renders          | Parent's state changes → all children re-render |
| Context value changes      | `ShipmentContext` provider value updates        |
| Custom hook state changes  | `useShipmentStatus()` updates internally       |

> [!warning] The Big Gotcha
> When a parent re-renders, **ALL its children re-render by default** — even if their props haven't changed. This is like a warehouse doing a full inventory recount every time one pallet moves. That's where `React.memo` comes in.

---

## Phase 2 — React.memo (Skip Unnecessary Re-Renders)

### The Problem

```jsx
function ShipmentDashboard() {
  const [searchTerm, setSearchTerm] = useState('');
  const [shipments, setShipments] = useState(initialShipments);

  return (
    <div>
      {/* Typing here causes ALL ShipmentCards to re-render */}
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      {shipments.map((s) => (
        // ❌ Every ShipmentCard re-renders on every keystroke!
        <ShipmentCard key={s.id} shipment={s} />
      ))}
    </div>
  );
}
```

Every time you type in the search box, `searchTerm` state changes → `ShipmentDashboard` re-renders → **every `ShipmentCard` re-renders** even though `shipment` props didn't change.

### The Solution: React.memo

`React.memo` is a **higher-order component** that wraps your component and skips re-rendering if props haven't changed (shallow comparison).

> [!tip] Spring Boot Analogy
> `React.memo` is like `@Cacheable` on a service method. If the input (props) is the same as last time, return the cached result (previous render) instead of recalculating.

```jsx
// ✅ Wrapped with React.memo — only re-renders when `shipment` prop changes
const ShipmentCard = React.memo(function ShipmentCard({ shipment }) {
  console.log(`Rendering ShipmentCard: ${shipment.trackingNumber}`);

  return (
    <div className="shipment-card">
      <h3>{shipment.trackingNumber}</h3>
      <p>Origin: {shipment.origin}</p>
      <p>Destination: {shipment.destination}</p>
      <span className={`status-badge ${shipment.status}`}>
        {shipment.status}
      </span>
    </div>
  );
});
```

```
  WITHOUT React.memo                WITH React.memo
  ─────────────────                 ───────────────

  Parent re-renders                 Parent re-renders
       │                                 │
       ▼                                 ▼
  ┌──────────┐                    ┌──────────────┐
  │ Child A   │ ← re-renders     │ Props same?   │
  │ Child B   │ ← re-renders     │   A: YES → ⏭  │ skip
  │ Child C   │ ← re-renders     │   B: YES → ⏭  │ skip
  └──────────┘                    │   C: NO  → 🔄 │ re-render
                                  └──────────────┘
```

### Custom Comparison Function

By default, `React.memo` does a **shallow comparison** of props (like `Objects.equals()` on each prop). For complex objects, you can provide a custom comparator:

```jsx
// Custom comparison — only re-render if tracking number or status changed
const ShipmentCard = React.memo(
  function ShipmentCard({ shipment, onSelect }) {
    return (
      <div onClick={() => onSelect(shipment.id)}>
        <h3>{shipment.trackingNumber}</h3>
        <p>Status: {shipment.status}</p>
        <p>ETA: {shipment.eta}</p>
      </div>
    );
  },
  // arePropsEqual — return TRUE to SKIP re-render
  (prevProps, nextProps) => {
    return (
      prevProps.shipment.trackingNumber === nextProps.shipment.trackingNumber &&
      prevProps.shipment.status === nextProps.shipment.status
    );
  }
);
```

> [!warning] When React.memo Hurts
> - If props change **every render** anyway → memo adds overhead (comparison cost) with zero benefit
> - If the component is **very cheap** to render → the comparison might cost more than just re-rendering
> - If you pass **new object/array/function references** every render → memo becomes useless (we'll fix this with `useMemo` and `useCallback`)

### Quick Reference: When to Use React.memo

| Scenario                              | Use React.memo? |
| ------------------------------------- | ---------------- |
| Component renders often with same props | ✅ Yes          |
| Component is expensive to render       | ✅ Yes          |
| Component always receives new props    | ❌ No — waste   |
| Component is tiny/cheap               | ❌ No — overhead |
| List item component in a large list   | ✅ Yes          |

---

## Phase 3 — useMemo (Memoize Expensive Computations)

### The Problem

```jsx
function ShipmentList({ shipments, filterStatus, sortField }) {
  // ❌ This runs on EVERY render, even if shipments/filter haven't changed
  const filteredAndSorted = shipments
    .filter((s) => s.status === filterStatus)
    .sort((a, b) => a[sortField].localeCompare(b[sortField]));

  return (
    <ul>
      {filteredAndSorted.map((s) => (
        <li key={s.id}>{s.trackingNumber} — {s.status}</li>
      ))}
    </ul>
  );
}
```

If you have 10,000 shipments and a user types in a search box (unrelated state change), this filter + sort runs again **for no reason**.

> [!tip] Spring Boot Analogy
> This is like executing an expensive SQL query on every HTTP request when the underlying data hasn't changed. You'd use `@Cacheable` or Redis caching — `useMemo` is React's equivalent.

### The Solution: useMemo

```jsx
function ShipmentList({ shipments, filterStatus, sortField }) {
  // ✅ Only recomputes when shipments, filterStatus, or sortField change
  const filteredAndSorted = useMemo(() => {
    console.log('Recomputing filtered list...'); // only logs when deps change
    return shipments
      .filter((s) => s.status === filterStatus)
      .sort((a, b) => a[sortField].localeCompare(b[sortField]));
  }, [shipments, filterStatus, sortField]); // ← dependency array

  return (
    <ul>
      {filteredAndSorted.map((s) => (
        <li key={s.id}>{s.trackingNumber} — {s.status}</li>
      ))}
    </ul>
  );
}
```

### The Dependency Array

```
  useMemo(() => computation, [dep1, dep2, dep3])
                                │     │     │
                                ▼     ▼     ▼
         React checks: did any of these change since last render?
                                │
                    ┌───────────┴───────────┐
                    │                       │
                  YES                      NO
              Recompute &              Return cached
              cache result              result
```

> [!warning] Dependency Array Rules
> - Include **every** variable from the component scope that the computation uses
> - Missing a dependency = stale data (like a cache that never invalidates)
> - Extra dependencies = unnecessary recomputation (but at least it's correct)
> - **Never lie about dependencies** — ESLint `exhaustive-deps` rule will warn you

### When NOT to Use useMemo

```jsx
// ❌ Don't memoize cheap operations — the memoization overhead isn't worth it
const fullName = useMemo(
  () => `${shipment.origin} → ${shipment.destination}`,
  [shipment.origin, shipment.destination]
);

// ✅ Just compute it inline — this is basically free
const fullName = `${shipment.origin} → ${shipment.destination}`;
```

| Use useMemo When...                                | Skip When...                          |
| -------------------------------------------------- | ------------------------------------- |
| Filtering/sorting large arrays (1000+ items)       | Simple string concatenation           |
| Complex calculations (rate calculations, ETAs)     | Basic arithmetic                      |
| Creating derived data structures                   | Trivial transformations               |
| Object/array references needed for memo'd children | The component rarely re-renders anyway |

---

## Phase 4 — useCallback (Memoize Function References)

### The Problem

In JavaScript, every time a component renders, functions defined inside it get **new references** — even if the code is identical.

```jsx
function ShipmentDashboard({ shipments }) {
  const [selectedId, setSelectedId] = useState(null);

  // ❌ New function reference on every render
  const handleSelect = (id) => {
    setSelectedId(id);
  };

  return (
    <>
      {shipments.map((s) => (
        // Even with React.memo, ShipmentCard re-renders because
        // handleSelect is a NEW reference every time
        <ShipmentCard
          key={s.id}
          shipment={s}
          onSelect={handleSelect} // ← new ref each render!
        />
      ))}
    </>
  );
}
```

> [!tip] Spring Boot Analogy
> Imagine you have a `@Bean` method that returns a new `ShipmentValidator` instance every time it's called — but the validator logic is identical. Spring would tell you to make it a singleton. `useCallback` does the same: it returns the **same function reference** unless dependencies change.

### The Solution: useCallback

```jsx
function ShipmentDashboard({ shipments }) {
  const [selectedId, setSelectedId] = useState(null);

  // ✅ Same function reference across renders (unless dependencies change)
  const handleSelect = useCallback((id) => {
    setSelectedId(id);
  }, []); // empty deps — setSelectedId is stable from useState

  return (
    <>
      {shipments.map((s) => (
        // Now React.memo on ShipmentCard actually works!
        <ShipmentCard
          key={s.id}
          shipment={s}
          onSelect={handleSelect} // ← same ref across renders ✅
        />
      ))}
    </>
  );
}

// This memo is now effective because onSelect won't trigger re-renders
const ShipmentCard = React.memo(function ShipmentCard({ shipment, onSelect }) {
  return (
    <div onClick={() => onSelect(shipment.id)}>
      <h3>{shipment.trackingNumber}</h3>
      <p>{shipment.status}</p>
    </div>
  );
});
```

### The useCallback + React.memo Connection

```
  ┌──────────────────────────────────────────────────────────┐
  │  useCallback + React.memo = The Performance Power Couple │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  useCallback alone?     → Saves a function reference.    │
  │                            Useless without memo.         │
  │                                                          │
  │  React.memo alone?      → Checks props for changes.     │
  │                            Defeated by new fn refs.      │
  │                                                          │
  │  useCallback + memo?    → Chef's kiss. 🤌               │
  │                            Stable refs + shallow check   │
  │                            = skipped re-renders.         │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

> [!info] Think of It This Way
> `React.memo` is the **security guard** at the warehouse gate checking IDs.
> `useCallback` ensures you hand the guard the **same ID card** each time, so they wave you through instead of running a full background check.

### useMemo vs useCallback — What's the Difference?

```jsx
// useCallback: memoizes the FUNCTION itself
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);

// useMemo: memoizes the RETURN VALUE of a function
const expensiveResult = useMemo(() => {
  return computeExpensiveThing(data);
}, [data]);

// In fact, useCallback is just shorthand:
// useCallback(fn, deps)  ===  useMemo(() => fn, deps)
```

---

## Phase 5 — Code Splitting (Lazy Loading)

### The Problem

By default, your entire React app is bundled into **one big JavaScript file**. Users download everything — even pages they may never visit.

```
  WITHOUT Code Splitting:
  ┌────────────────────────────────────────────┐
  │              main.bundle.js (2.5 MB)       │
  │  ┌────────┬────────┬────────┬────────────┐ │
  │  │ Home   │ Track  │ Report │ Admin      │ │
  │  │ Page   │ Page   │ Page   │ Dashboard  │ │
  │  └────────┴────────┴────────┴────────────┘ │
  │  User loads Home → downloads ALL of this   │
  └────────────────────────────────────────────┘

  WITH Code Splitting:
  ┌──────────────────┐
  │  main.bundle.js  │  ← 500 KB (shared code + Home page)
  │  (initial load)  │
  └──────────────────┘
       │
       ├── track.chunk.js   (200 KB) ← loaded when user visits /track
       ├── report.chunk.js  (300 KB) ← loaded when user visits /reports
       └── admin.chunk.js   (400 KB) ← loaded when user visits /admin
```

> [!tip] Spring Boot Analogy
> This is like **lazy-loading Spring beans**. With `@Lazy`, Spring doesn't instantiate a bean until it's first requested. Code splitting does the same — don't load the Reports module JavaScript until the user actually navigates to Reports.

### React.lazy + Suspense

```jsx
import { lazy, Suspense } from 'react';

// Dynamic imports — each becomes a separate chunk
const ShipmentTracker = lazy(() => import('./pages/ShipmentTracker'));
const ReportsPage = lazy(() => import('./pages/ReportsPage'));
const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));

function App() {
  return (
    <Routes>
      {/* Home page loads immediately */}
      <Route path="/" element={<HomePage />} />

      {/* These load on demand, wrapped in Suspense */}
      <Route
        path="/track"
        element={
          <Suspense fallback={<LoadingSpinner message="Loading tracker..." />}>
            <ShipmentTracker />
          </Suspense>
        }
      />
      <Route
        path="/reports"
        element={
          <Suspense fallback={<LoadingSpinner message="Loading reports..." />}>
            <ReportsPage />
          </Suspense>
        }
      />
      <Route
        path="/admin"
        element={
          <Suspense fallback={<LoadingSpinner message="Loading admin..." />}>
            <AdminDashboard />
          </Suspense>
        }
      />
    </Routes>
  );
}
```

### A Reusable Loading Fallback

```jsx
function LoadingSpinner({ message = 'Loading...' }) {
  return (
    <div className="loading-container">
      <div className="spinner" />
      <p>{message}</p>
    </div>
  );
}
```

> [!warning] Code Splitting Gotcha
> `React.lazy` only works with **default exports**. If your component uses named exports, create a small intermediate module:
> ```jsx
> // ShipmentDetails.lazy.js
> export { ShipmentDetails as default } from './ShipmentDetails';
> ```

---

## Phase 6 — Virtualization (Render Only What's Visible)

### The Problem

Your logistics dashboard needs to display **10,000 shipment rows**. Rendering all 10,000 DOM nodes at once is catastrophic for performance.

> [!tip] Logistics Analogy
> Think of a container ship. It can carry 20,000 TEU containers — but you don't hoist all 20,000 onto the deck at once. You load what fits, and the rest stays in the yard until needed. **Virtualization** renders only the rows visible in the viewport, and swaps in new rows as the user scrolls.

```
  WITHOUT Virtualization (all 10,000 rows in DOM):
  ┌──────────────────────────┐
  │  Row 1    ← visible      │ ▲
  │  Row 2    ← visible      │ │  Viewport
  │  Row 3    ← visible      │ │  (what user sees)
  │  Row 4    ← visible      │ ▼
  │  Row 5    ← hidden below │
  │  Row 6    ← hidden below │
  │  ...                     │
  │  Row 9999 ← hidden below │  ❌ All 10,000 in DOM!
  │  Row 10000               │
  └──────────────────────────┘

  WITH Virtualization (only ~20 rows in DOM):
  ┌──────────────────────────┐
  │  [spacer div: 0px]       │
  │  Row 1    ← visible      │ ▲
  │  Row 2    ← visible      │ │  Viewport
  │  Row 3    ← visible      │ │
  │  Row 4    ← visible      │ ▼
  │  [spacer div: 399,200px] │  ✅ Only visible rows rendered!
  └──────────────────────────┘
```

### Using @tanstack/react-virtual

```bash
npm install @tanstack/react-virtual
```

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function ShipmentTable({ shipments }) {
  // shipments = array of 10,000 items
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: shipments.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50, // estimated row height in px
    overscan: 5, // extra rows above/below viewport for smooth scrolling
  });

  return (
    <div
      ref={parentRef}
      style={{ height: '600px', overflow: 'auto' }}
    >
      {/* Total height container — enables proper scroll bar */}
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const shipment = shipments[virtualRow.index];
          return (
            <div
              key={shipment.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <span>{shipment.trackingNumber}</span>
              <span>{shipment.origin} → {shipment.destination}</span>
              <span>{shipment.status}</span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### Performance Comparison

| Approach               | DOM Nodes | Initial Render | Scroll Performance |
| ---------------------- | --------- | -------------- | ------------------ |
| Render all 10,000      | ~10,000   | 2–5 seconds    | Janky, laggy       |
| Virtualized            | ~20–30    | < 50ms         | Smooth 60fps       |
| Paginated (50/page)    | ~50       | Fast           | N/A (no scroll)    |

---

## Phase 7 — Key Optimization Patterns

### Pattern 1: Move State Down

If only one part of the tree needs the state, move it there. Don't hoist state to the top and re-render the whole tree.

```jsx
// ❌ BAD: search state lives too high — entire dashboard re-renders on typing
function ShipmentDashboard() {
  const [search, setSearch] = useState('');
  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <ShipmentStats />        {/* re-renders on every keystroke! */}
      <CarrierPerformance />   {/* re-renders on every keystroke! */}
      <ShipmentList />         {/* re-renders on every keystroke! */}
    </div>
  );
}
```

```jsx
// ✅ GOOD: extract the search into its own component
function SearchBox() {
  const [search, setSearch] = useState(''); // state is local
  return <input value={search} onChange={(e) => setSearch(e.target.value)} />;
}

function ShipmentDashboard() {
  return (
    <div>
      <SearchBox />            {/* only this re-renders on typing */}
      <ShipmentStats />        {/* unaffected ✅ */}
      <CarrierPerformance />   {/* unaffected ✅ */}
      <ShipmentList />         {/* unaffected ✅ */}
    </div>
  );
}
```

### Pattern 2: Composition Pattern (Children Don't Re-Render)

```jsx
// ✅ The children prop is a React element created by the PARENT.
//    When SlowWrapper re-renders, children is the same reference.
function SlowWrapper({ children }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      {children} {/* Does NOT re-render when count changes! */}
    </div>
  );
}

// Usage:
<SlowWrapper>
  <ExpensiveShipmentMap />  {/* Passed as children — immune to count changes */}
</SlowWrapper>
```

> [!info] Why This Works
> `children` was created in the **parent's** render scope. When `SlowWrapper` re-renders (due to `count` changing), the `children` prop is still the same React element reference from the parent. React sees the same reference and skips re-rendering it. This is free optimization — no `memo` needed.

### Pattern 3: useTransition (React 18+)

Mark non-urgent updates as **transitions** so they don't block urgent UI updates like typing.

```jsx
import { useState, useTransition } from 'react';

function ShipmentSearch({ shipments }) {
  const [query, setQuery] = useState('');
  const [filteredResults, setFilteredResults] = useState(shipments);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (e) => {
    const value = e.target.value;

    // URGENT: update the input immediately (user sees keystrokes)
    setQuery(value);

    // NON-URGENT: filter 10,000 shipments in the background
    startTransition(() => {
      const results = shipments.filter((s) =>
        s.trackingNumber.toLowerCase().includes(value.toLowerCase())
      );
      setFilteredResults(results);
    });
  };

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending && <p>Filtering shipments...</p>}
      <ShipmentTable shipments={filteredResults} />
    </div>
  );
}
```

> [!tip] Spring Boot Analogy
> `useTransition` is like using `@Async` in Spring. The main thread (UI) stays responsive while the heavy work (filtering) happens in the background. If a new keystroke arrives, React **interrupts** the old transition and starts the new one — like cancelling a stale async task.

---

## Phase 8 — Profiling with React DevTools

### Setup

1. Install the **React Developer Tools** browser extension (Chrome / Firefox)
2. Open DevTools → **Profiler** tab
3. Click the **Record** button (⏺)
4. Interact with your app (click, type, scroll)
5. Click **Stop** and analyze the flamegraph

### Reading the Flamegraph

```
  ┌─────────────────────────────────────────────────────────┐
  │  Flamegraph Example                                     │
  ├─────────────────────────────────────────────────────────┤
  │                                                         │
  │  App ██████████████████████████████████  12ms            │
  │    ├─ Header █████  2ms                                 │
  │    ├─ ShipmentDashboard ███████████████████  8ms        │
  │    │    ├─ SearchBox ██  1ms                            │
  │    │    ├─ ShipmentList ██████████████  6ms  ← 🔥      │
  │    │    │    ├─ ShipmentCard █  0.5ms                   │
  │    │    │    ├─ ShipmentCard █  0.5ms                   │
  │    │    │    ├─ ShipmentCard █  0.5ms                   │
  │    │    │    └─ ... (200 more)                          │
  │    │    └─ FilterPanel ██  1ms                          │
  │    └─ Footer █  0.5ms                                   │
  │                                                         │
  │  🔥 = Bottleneck: 200 ShipmentCards re-rendering        │
  │       Action: wrap ShipmentCard with React.memo          │
  └─────────────────────────────────────────────────────────┘
```

### Color Coding in Profiler

| Color       | Meaning                                           |
| ----------- | ------------------------------------------------- |
| **Grey**    | Component did NOT re-render (skipped) ✅           |
| **Blue**    | Component re-rendered, quick render                |
| **Yellow**  | Component re-rendered, moderate time ⚠️           |
| **Orange**  | Component re-rendered, slow — investigate! 🔥     |

### why-did-you-render (Advanced Debugging)

```bash
npm install @welldone-software/why-did-you-render --save-dev
```

```jsx
// wdyr.js — import this BEFORE React in your entry point
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}
```

```jsx
// Tag specific components to track
ShipmentCard.whyDidYouRender = true;
```

This logs to the console exactly **why** a component re-rendered and whether it was avoidable.

---

## Phase 9 — Performance Checklist

Use this before shipping any performance-sensitive React page:

### Rendering

- ✅ Profiled with React DevTools before optimizing
- ✅ Wrapped frequently re-rendered components with `React.memo`
- ✅ Used `useCallback` for event handlers passed to memoized children
- ✅ Used `useMemo` for expensive computations (large list filtering, sorting)
- ✅ Moved state down to the component that actually needs it
- ✅ Used composition pattern (`children` prop) where possible

### Data Loading

- ✅ Implemented pagination or infinite scroll for large datasets
- ✅ Used virtualization (`@tanstack/react-virtual`) for 1000+ item lists
- ✅ Debounced search inputs that trigger API calls or heavy filtering
- ✅ Used `useTransition` for non-urgent state updates (React 18+)

### Bundle Size

- ✅ Code-split routes with `React.lazy` + `Suspense`
- ✅ Checked bundle size with `webpack-bundle-analyzer` or `source-map-explorer`
- ✅ Tree-shaken unused imports (use named imports, not `import * as`)
- ✅ Lazy-loaded heavy third-party libraries (chart libs, date-fns, etc.)

### General

- ✅ Avoided inline object/array creation in JSX props
- ✅ Used stable `key` props (IDs, not array indices) in lists
- ✅ No unnecessary `useEffect` re-runs (correct dependency arrays)
- ✅ Images are optimized and lazy-loaded (`loading="lazy"`)

---

## Quick Reference: Optimization Tools at a Glance

```
  ┌───────────────────┬────────────────────┬──────────────────────────────┐
  │ Tool              │ What It Does       │ Logistics Analogy            │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ React.memo        │ Skip re-render if  │ @Cacheable — return cached   │
  │                   │ props unchanged    │ response if input matches    │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ useMemo           │ Cache expensive    │ Pre-computed shipment ETAs   │
  │                   │ calculations       │ — don't recalculate if data  │
  │                   │                    │ hasn't changed               │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ useCallback       │ Stable function    │ Singleton @Bean — same       │
  │                   │ reference          │ instance every time          │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ React.lazy        │ Load code on       │ @Lazy bean initialization    │
  │                   │ demand             │ — load module when needed    │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ Virtualization    │ Render only        │ Container ship — only load   │
  │                   │ visible rows       │ containers needed on deck    │
  ├───────────────────┼────────────────────┼──────────────────────────────┤
  │ useTransition     │ Background updates │ @Async — heavy work off the  │
  │                   │ don't block UI     │ main thread                  │
  └───────────────────┴────────────────────┴──────────────────────────────┘
```

---

## Cross-References

- [[Hooks]] — Deep dive into `useState`, `useEffect`, and custom hooks
- [[Components and Props]] — Understanding component re-rendering and prop flow
- [[React vs Angular Comparison]] — How Angular handles change detection differently

---

> [!tip] Remember
> The fastest code is the code that never runs. Don't render what you don't need, don't compute what hasn't changed, and don't load what the user hasn't asked for. **Measure first, optimize second.**
