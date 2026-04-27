---
tags:
  - react
  - state
  - hooks
  - lifecycle
  - beginner
---

# State and Lifecycle

> *"The only constant in life is change."* — Heraclitus

---

## Phase 1: What is State?

### The Core Idea

State is **data that changes over time** and affects what the component renders. When state changes, React **re-renders** the component to reflect the new data in the UI.

If props are the **inputs** to a component (read-only, set by the parent), state is the **internal memory** of the component (read-write, managed by the component itself).

### Analogy: Props vs State in Logistics

> [!tip]
> **Props are like a shipping order** — once the warehouse receives the order, it's fixed. The order says "Ship 100 units of Widget-A to Sydney." The warehouse doesn't change the order; it just executes it.
>
> **State is like the shipment's current GPS location** — it changes constantly as the truck moves. The tracking system updates the location every 30 seconds. The UI shows the latest position.

```
Props (immutable):                    State (mutable):
──────────────────                    ────────────────
Shipping Order:                       Shipment Tracker:
  From: Shanghai                        Current Location: 34.05°N, 118.25°W
  To: Sydney                            Speed: 18 knots
  Items: 100x Widget-A                  ETA: Dec 15, 2024
  Weight: 2,500 kg                      Status: in-transit → customs → delivered
                                        (changes over time)
Fixed at creation.                    Updates continuously.
Parent decides.                       Component manages internally.
```

### State vs Props — Comparison Table

| Aspect | Props | State |
|---|---|---|
| **Who controls it?** | Parent component | The component itself |
| **Mutable?** | No (read-only) | Yes (via setter function) |
| **When set?** | When parent renders the child | Inside the component (on events, effects) |
| **Triggers re-render?** | Yes (when parent passes new props) | Yes (when state is updated) |
| **Java analogy** | Method parameters / constructor args | Private instance fields |
| **Direction** | Flows DOWN (parent → child) | Local to the component |

```
Data flow in a component:

              ┌───────────────────────┐
  props ──►   │                       │
  (from       │     Component         │   ──►  JSX (UI output)
  parent)     │                       │
              │   state (internal)    │
              │   ┌─────────────┐     │
              │   │ count: 5    │     │
              │   │ items: [...] │    │
              │   └─────────────┘     │
              └───────────────────────┘
                       ▲
                       │
                  setState()
                  (user action, API response, timer, etc.)
```

---

## Phase 2: useState Hook

### The Most Fundamental Hook

`useState` is the hook that lets functional components **hold and update state**. It's the React equivalent of declaring a private field in a Java class.

**Syntax:**

```tsx
const [currentValue, setterFunction] = useState(initialValue);
//      ▲                ▲                        ▲
//      │                │                        │
//  Current state    Function to update it    Starting value
```

> [!note]
> `useState` returns an **array** with exactly two elements. We use JavaScript **array destructuring** to name them. The naming convention is `[thing, setThing]`.

### Basic Examples

```tsx
import { useState } from 'react';

function ShipmentTracker() {
  // String state — current shipment status
  const [status, setStatus] = useState<string>('pending');

  // Number state — count of items
  const [itemCount, setItemCount] = useState<number>(0);

  // Boolean state — loading indicator
  const [isLoading, setIsLoading] = useState<boolean>(false);

  // Array state — list of shipments
  const [shipments, setShipments] = useState<Shipment[]>([]);

  // Object state — filter criteria
  const [filters, setFilters] = useState<FilterState>({
    status: 'all',
    dateRange: 'last-7-days',
    carrier: '',
  });

  return (
    <div>
      <p>Status: {status}</p>
      <p>Items: {itemCount}</p>
      <button onClick={() => setStatus('in-transit')}>
        Mark In Transit
      </button>
      <button onClick={() => setItemCount((prev) => prev + 1)}>
        Add Item
      </button>
    </div>
  );
}
```

### Updating State: Direct Value vs Updater Function

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ Direct value — can cause bugs with batched updates
  const handleIncrement = () => {
    setCount(count + 1);   // Uses the count value from THIS render
    setCount(count + 1);   // Still uses the SAME count value — only increments by 1!
  };

  // ✅ Updater function — always uses the latest state
  const handleIncrementCorrect = () => {
    setCount((prev) => prev + 1);  // prev is guaranteed to be the latest value
    setCount((prev) => prev + 1);  // prev is now count+1 — increments by 2!
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleIncrementCorrect}>+2</button>
    </div>
  );
}
```

> [!warning]
> **Rule of thumb:** If your new state depends on the previous state, **always use the updater function** `(prev) => newValue`. This prevents bugs from React's batched updates. Think of it like optimistic locking in databases — you need the latest version.

### State is Asynchronous — Batched Updates

React **batches** multiple state updates within the same event handler into a single re-render for performance:

```tsx
function ShipmentForm() {
  const [origin, setOrigin] = useState('');
  const [destination, setDestination] = useState('');
  const [weight, setWeight] = useState(0);

  const handleQuickFill = () => {
    // All three updates happen in ONE re-render (batched)
    setOrigin('Sydney');
    setDestination('Melbourne');
    setWeight(500);
    // React doesn't re-render 3 times — it batches into 1 re-render
  };

  // After calling handleQuickFill, you can NOT immediately read the new values:
  // console.log(origin); ← Still '' — state update hasn't been applied yet!

  return (
    <div>
      <p>{origin} → {destination} ({weight} kg)</p>
      <button onClick={handleQuickFill}>Fill Sydney → Melbourne</button>
    </div>
  );
}
```

> [!note]
> **Java comparison:** This is similar to how JPA batches SQL statements. You call `entityManager.persist()` multiple times, but the actual SQL executes at flush time — not immediately. React works the same way with state.

### Never Mutate State Directly

```tsx
function ShipmentManager() {
  const [shipments, setShipments] = useState<Shipment[]>([
    { id: '1', trackingId: 'TRK-001', status: 'pending' },
    { id: '2', trackingId: 'TRK-002', status: 'in-transit' },
  ]);

  // ❌ WRONG — mutating the existing array. React won't detect the change.
  const addShipmentBad = (newShipment: Shipment) => {
    shipments.push(newShipment);  // Mutates the array in place
    setShipments(shipments);       // Same reference — React thinks nothing changed!
  };

  // ✅ CORRECT — create a NEW array with the spread operator
  const addShipment = (newShipment: Shipment) => {
    setShipments((prev) => [...prev, newShipment]);  // New array reference
  };

  // ✅ CORRECT — removing an item (creates new array)
  const removeShipment = (id: string) => {
    setShipments((prev) => prev.filter((s) => s.id !== id));
  };

  // ✅ CORRECT — updating an item (creates new array with one modified element)
  const updateStatus = (id: string, newStatus: string) => {
    setShipments((prev) =>
      prev.map((s) => (s.id === id ? { ...s, status: newStatus } : s))
    );
  };

  return <div>{/* render shipments */}</div>;
}
```

---

## Phase 3: How Re-rendering Works

### The Render Cycle

When state changes, React **re-executes the component function** to produce new JSX, then diffs it against the previous output and patches the DOM.

```
User clicks "Mark Delivered"
        │
        ▼
setStatus('delivered')           ← State update triggered
        │
        ▼
React schedules a re-render
        │
        ▼
┌────────────────────────────────────────┐
│  React calls ShipmentCard() again       │
│                                         │
│  function ShipmentCard({ trackingId }) { │
│    const [status, setStatus]            │
│      = useState('delivered'); ← NEW     │
│                                         │
│    return (                             │
│      <div>                              │
│        <span>delivered</span> ← CHANGED │
│      </div>                             │
│    );                                   │
│  }                                      │
└──────────────────┬─────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│   Virtual DOM Diff                    │
│                                       │
│   Old: <span>in-transit</span>        │
│   New: <span>delivered</span>         │
│                                       │
│   Diff: text content changed          │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│   Commit to Real DOM                  │
│                                       │
│   Update only the text node:          │
│   "in-transit" → "delivered"          │
└──────────────────────────────────────┘
```

### What Triggers a Re-render?

| Trigger | Example |
|---|---|
| **State change** (`setState`) | `setCount(5)` — component re-renders |
| **Parent re-renders** | If `<App>` re-renders, all children re-render too |
| **Context change** | A value in React Context changes → all consumers re-render |
| **Force update** | `forceUpdate()` (class components) — avoid this |

### Re-render Cascade

When a parent re-renders, **all children re-render by default**, even if their props haven't changed:

```
<App> re-renders (state changed)
  │
  ├── <Header /> re-renders       ← even if no prop changed
  ├── <Dashboard> re-renders      ← state changed here
  │   ├── <SearchBar /> re-renders
  │   ├── <ShipmentList> re-renders
  │   │   ├── <ShipmentCard /> re-renders
  │   │   ├── <ShipmentCard /> re-renders   ← ALL cards re-render
  │   │   └── <ShipmentCard /> re-renders
  │   └── <Pagination /> re-renders
  └── <Footer /> re-renders       ← even if no prop changed
```

> [!tip]
> This is React's default behavior. For most apps, it's fast enough. If performance becomes an issue, you can optimize with `React.memo`, `useMemo`, and `useCallback` — covered in [[Hooks]].

### StrictMode Double-Rendering

In development mode, React's `<StrictMode>` wrapper **calls your component function twice** to help detect side effects:

```tsx
// In main.tsx — wraps the entire app in StrictMode
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
);
```

> [!warning]
> Don't panic when you see `console.log` firing twice in development! It's StrictMode verifying your component is pure. This **does not happen in production builds**. It's like having a test runner that executes your function twice to check for unintended side effects.

---

## Phase 4: Lifting State Up

### The Problem

Sometimes two sibling components need to share the same state. Since data flows **down** (parent → child), siblings can't directly share state with each other.

### The Solution: Lift State to the Common Parent

Move the shared state to the **nearest common ancestor**, then pass it down via props and update it via callback functions.

> [!tip]
> **Analogy: The logistics control tower.** Instead of two delivery trucks communicating directly (which is fragile and uncoordinated), they both report to the **control tower** (parent component). The control tower has the full picture and coordinates between them. The trucks receive instructions (props) and report status (callback functions).

```
❌ Before — siblings can't share state:

  <Dashboard>
  ├── <SearchBar>         state: { query: "TRK-001" }
  └── <ShipmentList>      state: { filteredItems: [...] }
      Problem: How does ShipmentList know about SearchBar's query?

✅ After — state lifted to parent:

  <Dashboard>              state: { query: "TRK-001", filteredItems: [...] }
  ├── <SearchBar>          props: { query, onQueryChange }    ← reads + reports
  └── <ShipmentList>       props: { filteredItems }           ← reads
```

### Full Example

```tsx
// Parent component — owns the shared state (the "control tower")
function ShipmentDashboard() {
  const [searchQuery, setSearchQuery] = useState('');
  const [shipments] = useState<Shipment[]>([
    { trackingId: 'TRK-001', origin: 'Sydney', status: 'pending' },
    { trackingId: 'TRK-002', origin: 'Melbourne', status: 'delivered' },
    { trackingId: 'TRK-003', origin: 'Brisbane', status: 'in-transit' },
  ]);

  // Derived state — filtered list based on search query
  const filteredShipments = shipments.filter((s) =>
    s.trackingId.toLowerCase().includes(searchQuery.toLowerCase()) ||
    s.origin.toLowerCase().includes(searchQuery.toLowerCase())
  );

  return (
    <div>
      {/* SearchBar reports changes UP via callback */}
      <SearchBar query={searchQuery} onQueryChange={setSearchQuery} />

      {/* ShipmentList receives filtered data DOWN via props */}
      <ShipmentList shipments={filteredShipments} />

      <p>{filteredShipments.length} of {shipments.length} shipments</p>
    </div>
  );
}

// Child 1 — receives state and a callback to update it
interface SearchBarProps {
  query: string;
  onQueryChange: (newQuery: string) => void;
}

function SearchBar({ query, onQueryChange }: SearchBarProps) {
  return (
    <input
      type="text"
      value={query}
      onChange={(e) => onQueryChange(e.target.value)}
      placeholder="Search shipments..."
    />
  );
}

// Child 2 — receives filtered data (doesn't know about search logic)
interface ShipmentListProps {
  shipments: Shipment[];
}

function ShipmentList({ shipments }: ShipmentListProps) {
  if (shipments.length === 0) {
    return <p>No shipments found.</p>;
  }

  return (
    <ul>
      {shipments.map((s) => (
        <li key={s.trackingId}>
          {s.trackingId} — {s.origin} — {s.status}
        </li>
      ))}
    </ul>
  );
}
```

### Data Flow Diagram

```
User types "TRK-001" in SearchBar
        │
        ▼
SearchBar calls onQueryChange("TRK-001")     ← event flows UP
        │
        ▼
ShipmentDashboard receives callback
        │
        ▼
setSearchQuery("TRK-001")                    ← state updated in parent
        │
        ▼
ShipmentDashboard re-renders
        │
        ├── filteredShipments recalculated
        │
        ├── SearchBar receives query="TRK-001"     ← data flows DOWN
        │
        └── ShipmentList receives [matching items]  ← data flows DOWN
```

---

## Phase 5: State Immutability

### Why Immutability Matters

React uses **reference comparison** (`===`) to detect state changes. If you mutate an object/array in place, the reference stays the same, and React **won't detect the change** — your UI won't update.

```tsx
// ❌ Mutation — same reference, React ignores it
const obj = { name: 'Sydney' };
obj.name = 'Melbourne';       // Same object reference
// obj === obj is true → React: "nothing changed"

// ✅ New object — new reference, React detects the change
const newObj = { ...obj, name: 'Melbourne' };  // New object reference
// newObj === obj is false → React: "state changed, re-render!"
```

> [!note]
> **Java comparison:** This is like the difference between modifying a mutable HashMap vs creating a new unmodifiable Map (via `Map.copyOf()`). In React, always create new copies — like using immutable collections in Java.

### Updating Objects

```tsx
const [shipment, setShipment] = useState<Shipment>({
  trackingId: 'TRK-001',
  origin: 'Sydney',
  destination: 'Melbourne',
  status: 'pending',
  weight: 500,
});

// ❌ WRONG — mutating the existing object
const updateStatusBad = () => {
  shipment.status = 'delivered';     // Mutation!
  setShipment(shipment);              // Same reference — React won't re-render
};

// ✅ CORRECT — spread operator creates a new object
const updateStatus = () => {
  setShipment((prev) => ({
    ...prev,                          // Copy all existing fields
    status: 'delivered',              // Override the one field you're changing
  }));
};

// ✅ Updating nested objects
const [order, setOrder] = useState({
  id: 'ORD-001',
  customer: {
    name: 'Acme Corp',
    address: {
      city: 'Sydney',
      country: 'Australia',
    },
  },
});

const updateCity = (newCity: string) => {
  setOrder((prev) => ({
    ...prev,
    customer: {
      ...prev.customer,
      address: {
        ...prev.customer.address,
        city: newCity,                 // Only this field changes
      },
    },
  }));
};
```

### Updating Arrays

```tsx
const [shipments, setShipments] = useState<Shipment[]>([]);

// ✅ ADD an item to the end
const addShipment = (newShipment: Shipment) => {
  setShipments((prev) => [...prev, newShipment]);
};

// ✅ ADD an item to the beginning
const prependShipment = (newShipment: Shipment) => {
  setShipments((prev) => [newShipment, ...prev]);
};

// ✅ REMOVE an item by ID
const removeShipment = (id: string) => {
  setShipments((prev) => prev.filter((s) => s.id !== id));
};

// ✅ UPDATE one item in the array
const updateShipmentStatus = (id: string, newStatus: string) => {
  setShipments((prev) =>
    prev.map((s) =>
      s.id === id
        ? { ...s, status: newStatus }  // Create new object for the changed item
        : s                             // Keep unchanged items as-is
    )
  );
};

// ✅ SORT the array (creates a new sorted copy)
const sortByTrackingId = () => {
  setShipments((prev) =>
    [...prev].sort((a, b) => a.trackingId.localeCompare(b.trackingId))
  );
};
```

### Common Mutation Mistakes

| ❌ Mutation (Broken) | ✅ Immutable (Correct) |
|---|---|
| `array.push(item)` | `[...array, item]` |
| `array.splice(i, 1)` | `array.filter((_, idx) => idx !== i)` |
| `array[i] = newItem` | `array.map((item, idx) => idx === i ? newItem : item)` |
| `array.sort()` | `[...array].sort()` |
| `array.reverse()` | `[...array].reverse()` |
| `obj.key = value` | `{ ...obj, key: value }` |

> [!warning]
> **Memorize this rule:** `push`, `pop`, `splice`, `sort`, `reverse`, and direct property assignment (`obj.x = y`) are **mutations**. They modify in place. In React state, **never use them directly on state**. Always create a copy first with `[...array]` or `{ ...obj }`.

---

## Phase 6: Class Component Lifecycle (Legacy Reference)

### The Lifecycle Phases

Class components have **lifecycle methods** — special methods that React calls at specific points in a component's life. Think of them like JPA entity listeners (`@PostPersist`, `@PreUpdate`, `@PreRemove`).

```
MOUNTING (component is created and added to DOM):
─────────────────────────────────────────────────
constructor(props)         ← Initialize state, bind methods
        │
        ▼
static getDerivedStateFromProps(props, state)   ← Rarely needed
        │
        ▼
render()                   ← Return JSX (PURE — no side effects!)
        │
        ▼
componentDidMount()        ← Component is now in the DOM
                              ✅ Fetch data, set up subscriptions,
                                 start timers, measure DOM elements


UPDATING (state or props change):
──────────────────────────────────
shouldComponentUpdate(nextProps, nextState)  ← Optimization: skip render?
        │
        ▼
render()                   ← Re-render with new state/props
        │
        ▼
componentDidUpdate(prevProps, prevState)     ← DOM updated
                              ✅ React to changes, conditional fetch


UNMOUNTING (component is removed from DOM):
───────────────────────────────────────────
componentWillUnmount()     ← Component is being removed
                              ✅ Clean up: cancel timers, unsubscribe,
                                 close WebSocket connections
```

### Lifecycle Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│                      COMPONENT LIFECYCLE                         │
│                                                                  │
│  ┌─── MOUNTING ───┐  ┌──── UPDATING ────┐  ┌── UNMOUNTING ──┐  │
│  │                 │  │                   │  │                 │  │
│  │  constructor()  │  │  New Props/State  │  │                 │  │
│  │       │         │  │       │           │  │                 │  │
│  │       ▼         │  │       ▼           │  │                 │  │
│  │    render()     │  │    render()       │  │                 │  │
│  │       │         │  │       │           │  │                 │  │
│  │       ▼         │  │       ▼           │  │       │         │  │
│  │  componentDid   │  │  componentDid     │  │  componentWill  │  │
│  │    Mount()      │  │    Update()       │  │    Unmount()    │  │
│  │                 │  │                   │  │                 │  │
│  │  ✅ API calls   │  │  ✅ React to      │  │  ✅ Cleanup     │  │
│  │  ✅ Timers      │  │     changes       │  │  ✅ Unsubscribe │  │
│  │  ✅ Subscribe   │  │  ✅ Conditional   │  │  ✅ Cancel       │  │
│  │                 │  │     re-fetch      │  │     requests    │  │
│  └─────────────────┘  └───────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Class Lifecycle Example

```tsx
class ShipmentTracker extends React.Component<TrackerProps, TrackerState> {
  private timerRef: number | null = null;

  constructor(props: TrackerProps) {
    super(props);
    this.state = {
      location: null,
      lastUpdated: null,
    };
  }

  // Called once after first render — like @PostConstruct in Spring
  componentDidMount() {
    this.fetchLocation();
    // Poll for GPS updates every 30 seconds
    this.timerRef = window.setInterval(() => this.fetchLocation(), 30000);
  }

  // Called after every re-render (except the first)
  componentDidUpdate(prevProps: TrackerProps) {
    // Only re-fetch if the shipment ID changed
    if (prevProps.shipmentId !== this.props.shipmentId) {
      this.fetchLocation();
    }
  }

  // Called right before the component is removed — like @PreDestroy in Spring
  componentWillUnmount() {
    // ALWAYS clean up timers, subscriptions, etc.
    if (this.timerRef) {
      window.clearInterval(this.timerRef);
    }
  }

  async fetchLocation() {
    const response = await fetch(`/api/shipments/${this.props.shipmentId}/location`);
    const location = await response.json();
    this.setState({ location, lastUpdated: new Date() });
  }

  render() {
    const { location, lastUpdated } = this.state;
    return (
      <div>
        {location ? (
          <p>Location: {location.lat}, {location.lng}</p>
        ) : (
          <p>Loading location...</p>
        )}
        {lastUpdated && <p>Updated: {lastUpdated.toLocaleTimeString()}</p>}
      </div>
    );
  }
}
```

### How Hooks Replaced Lifecycle Methods

The **modern equivalent** of every lifecycle method is a combination of `useState` and `useEffect`:

| Class Lifecycle Method | Hook Equivalent | When It Runs |
|---|---|---|
| `constructor()` | `useState(initialValue)` | On first render (initialization) |
| `componentDidMount()` | `useEffect(() => { ... }, [])` | Once, after first render |
| `componentDidUpdate()` | `useEffect(() => { ... }, [dep])` | When `dep` changes |
| `componentDidMount` + `componentDidUpdate` | `useEffect(() => { ... })` | After every render |
| `componentWillUnmount()` | `useEffect(() => { return cleanup }, [])` | When component is removed |
| `shouldComponentUpdate()` | `React.memo()` | Wrap the component |

### The Same Example — Rewritten with Hooks

```tsx
import { useState, useEffect } from 'react';

function ShipmentTracker({ shipmentId }: { shipmentId: string }) {
  const [location, setLocation] = useState<Location | null>(null);
  const [lastUpdated, setLastUpdated] = useState<Date | null>(null);

  useEffect(() => {
    // This runs on mount AND when shipmentId changes
    const fetchLocation = async () => {
      const response = await fetch(`/api/shipments/${shipmentId}/location`);
      const data = await response.json();
      setLocation(data);
      setLastUpdated(new Date());
    };

    fetchLocation();

    // Set up polling interval
    const timer = setInterval(fetchLocation, 30000);

    // Cleanup function — runs on unmount OR before the effect re-runs
    // This replaces componentWillUnmount
    return () => {
      clearInterval(timer);
    };
  }, [shipmentId]);  // Dependency array — re-run when shipmentId changes

  return (
    <div>
      {location ? (
        <p>Location: {location.lat}, {location.lng}</p>
      ) : (
        <p>Loading location...</p>
      )}
      {lastUpdated && <p>Updated: {lastUpdated.toLocaleTimeString()}</p>}
    </div>
  );
}
```

### Why Hooks Are Better

| Aspect | Class Lifecycle | Hooks |
|---|---|---|
| **Code organization** | Logic split across 3+ lifecycle methods | Related logic grouped in one `useEffect` |
| **Reuse** | HOCs or render props (complex) | Custom hooks (simple) |
| **Cleanup** | Easy to forget `componentWillUnmount` | Cleanup is returned from the effect — co-located |
| **Multiple concerns** | All in `componentDidMount` | Separate `useEffect` for each concern |
| **`this` keyword** | Constant source of bugs | No `this` — just closures |

```
Class component — related logic scattered:        Functional — related logic together:
───────────────────────────────────────────        ────────────────────────────────────
componentDidMount() {                              useEffect(() => {
  this.fetchData();          // Concern A            fetchData();           // Concern A
  this.startTimer();         // Concern B            return () => cleanup(); // Concern A
  this.setupWebSocket();     // Concern C          }, []);
}
                                                   useEffect(() => {
componentDidUpdate() {                               startTimer();          // Concern B
  this.fetchData();          // Concern A again      return () => clearTimer();
}                                                  }, []);

componentWillUnmount() {                           useEffect(() => {
  this.cleanup();            // Concern A cleanup    const ws = setupWebSocket();
  this.clearTimer();         // Concern B cleanup    return () => ws.close();
  this.closeWebSocket();     // Concern C cleanup  }, []);
}
```

---

## Quick Reference

### useState Cheat Sheet

```tsx
// Primitive types
const [count, setCount] = useState(0);
const [name, setName] = useState('');
const [isOpen, setIsOpen] = useState(false);

// With explicit TypeScript type
const [status, setStatus] = useState<'pending' | 'done'>('pending');

// Objects
const [user, setUser] = useState<User>({ name: '', email: '' });
setUser(prev => ({ ...prev, name: 'New Name' }));

// Arrays
const [items, setItems] = useState<Item[]>([]);
setItems(prev => [...prev, newItem]);          // Add
setItems(prev => prev.filter(i => i.id !== id)); // Remove
setItems(prev => prev.map(i =>                // Update
  i.id === id ? { ...i, done: true } : i
));

// Lazy initialization (expensive initial value)
const [data, setData] = useState(() => computeExpensiveValue());
```

### State Update Patterns

```
Action:              Pattern:
────────             ────────
Set value            setState(newValue)
Toggle boolean       setState(prev => !prev)
Increment            setState(prev => prev + 1)
Add to array         setState(prev => [...prev, item])
Remove from array    setState(prev => prev.filter(i => i.id !== id))
Update in array      setState(prev => prev.map(i => i.id === id ? {...i, key: val} : i))
Update object field  setState(prev => ({...prev, field: newValue}))
Reset to initial     setState(initialValue)
```

---

## What's Next?

- [[Hooks]] — Deep dive into `useEffect`, `useContext`, `useMemo`, `useCallback`, `useRef`, and custom hooks
- [[Components and Props]] — Review component composition patterns
- [[React Fundamentals]] — Revisit the basics if anything was unclear

> [!tip]
> **Practice exercise:** Build a `<ShipmentManager>` with `useState` that manages a list of shipments. Add buttons to: add a new shipment (with random tracking ID), update a shipment's status, remove a shipment, and sort by tracking ID. Focus on doing all updates immutably.
