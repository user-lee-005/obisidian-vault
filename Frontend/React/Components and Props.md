---
tags:
  - react
  - components
  - props
  - typescript
  - beginner
---

# Components and Props

> *"The secret to building large apps is never build large apps. Break your application into small pieces. Then, assemble those small pieces into your big application."* — Justin Meyer

---

## Phase 1: What are Components?

### The Building Blocks of React

Components are the **fundamental building blocks** of any React application. Every piece of UI you see — a button, a card, a form, a navigation bar — is a component. Components can be **nested inside other components** to form a tree.

> [!tip]
> **Analogy: Components are like microservices for UI.** In your backend world, each microservice does one thing well, is independently testable, and can be composed into a larger system. React components work the same way — each component handles one piece of UI, can be tested in isolation, and is assembled into a full page.

### The Component Tree

Every React app has a **component tree** — a hierarchy of nested components, starting from a single root (`<App>`).

```
<App>
├── <Header />
│   ├── <Logo />
│   ├── <Navigation />
│   │   ├── <NavLink to="/dashboard" />
│   │   ├── <NavLink to="/shipments" />
│   │   └── <NavLink to="/reports" />
│   └── <UserMenu />
├── <ShipmentDashboard>
│   ├── <SearchBar />
│   ├── <FilterPanel />
│   │   ├── <StatusFilter />
│   │   ├── <DateRangeFilter />
│   │   └── <CarrierFilter />
│   ├── <ShipmentList>
│   │   ├── <ShipmentCard />   ← repeated for each shipment
│   │   ├── <ShipmentCard />
│   │   ├── <ShipmentCard />
│   │   └── <ShipmentCard />
│   └── <Pagination />
└── <Footer />
```

**Data flows DOWN the tree** (parent → child via props).
**Events bubble UP the tree** (child → parent via callback functions).

```
Data Flow (props):        Event Flow (callbacks):
─────────────────         ─────────────────────
     <App>                      <App>
       │ props                    ▲ onSearch()
       ▼                          │
  <Dashboard>                <Dashboard>
       │ props                    ▲ onSearch()
       ▼                          │
  <SearchBar>                <SearchBar>
                              user types → calls onSearch()
```

### Analogy: Logistics Network = Component Tree

```
Supply Chain:                        React App:
─────────────                        ──────────
Headquarters (App)                   <App>
  ├── Regional Hub (Dashboard)         ├── <Dashboard>
  │   ├── Warehouse A (ShipmentList)   │   ├── <ShipmentList>
  │   │   ├── Pallet 1 (Card)         │   │   ├── <ShipmentCard>
  │   │   ├── Pallet 2 (Card)         │   │   ├── <ShipmentCard>
  │   │   └── Pallet 3 (Card)         │   │   └── <ShipmentCard>
  │   └── Dispatch Office (Search)     │   └── <SearchBar>
  └── Admin Office (Settings)          └── <Settings>

Orders flow DOWN from HQ.             Props flow DOWN from App.
Status updates flow UP from pallets.   Events flow UP from Cards.
```

---

## Phase 2: Functional Components

### The Modern Standard

Since React 16.8 (2019), **functional components** with Hooks are the standard. They replaced class components as the recommended way to write React.

A functional component is simply a **JavaScript/TypeScript function that returns JSX**:

```tsx
// The simplest possible component
function Welcome() {
  return <h1>Welcome to Freight Tracker</h1>;
}
```

### Increasing Complexity — Step by Step

**Level 1: Static component (no inputs)**

```tsx
function AppFooter() {
  return (
    <footer className="app-footer">
      <p>© 2024 Freight Tracker. All rights reserved.</p>
    </footer>
  );
}
```

**Level 2: Component with props (inputs from parent)**

```tsx
interface GreetingProps {
  userName: string;
  role: string;
}

function Greeting({ userName, role }: GreetingProps) {
  return (
    <div className="greeting">
      <h2>Hello, {userName}</h2>
      <p>Role: {role}</p>
    </div>
  );
}
```

**Level 3: Component with logic and conditional rendering**

```tsx
interface ShipmentBadgeProps {
  status: 'pending' | 'in-transit' | 'delivered' | 'cancelled';
  isUrgent: boolean;
}

function ShipmentBadge({ status, isUrgent }: ShipmentBadgeProps) {
  // Compute derived values (like a Java getter or computed property)
  const statusColors: Record<string, string> = {
    'pending': '#FFA500',
    'in-transit': '#007BFF',
    'delivered': '#28A745',
    'cancelled': '#DC3545',
  };

  const color = statusColors[status] || '#6C757D';

  return (
    <span
      className="shipment-badge"
      style={{ backgroundColor: color, color: 'white', padding: '4px 8px' }}
    >
      {isUrgent && '🔴 '}{status.toUpperCase()}
    </span>
  );
}
```

**Level 4: Component with state and event handlers**

```tsx
import { useState } from 'react';

interface ShipmentNoteProps {
  shipmentId: string;
  onSave: (shipmentId: string, note: string) => void;
}

function ShipmentNoteEditor({ shipmentId, onSave }: ShipmentNoteProps) {
  const [note, setNote] = useState('');
  const [isSaving, setIsSaving] = useState(false);

  const handleSave = async () => {
    setIsSaving(true);
    await onSave(shipmentId, note);
    setNote('');
    setIsSaving(false);
  };

  return (
    <div className="note-editor">
      <textarea
        value={note}
        onChange={(e) => setNote(e.target.value)}
        placeholder="Add a note to this shipment..."
        rows={3}
      />
      <button onClick={handleSave} disabled={isSaving || note.trim() === ''}>
        {isSaving ? 'Saving...' : 'Save Note'}
      </button>
    </div>
  );
}
```

### Arrow Function vs Function Declaration

Both are valid. Pick one and be consistent in your project:

```tsx
// Function declaration (recommended — matches Java method style)
function ShipmentCard({ trackingId }: ShipmentCardProps) {
  return <div>{trackingId}</div>;
}

// Arrow function (common in many codebases)
const ShipmentCard = ({ trackingId }: ShipmentCardProps) => {
  return <div>{trackingId}</div>;
};
```

> [!note]
> Coming from Java, function declarations feel more natural — they look like method definitions. Use whichever your team prefers, but be consistent.

---

## Phase 3: Props — Passing Data Down

### What are Props?

Props (short for "properties") are the **inputs to a component**. They are passed from a parent component to a child component and are **read-only** — a component cannot modify its own props.

> [!tip]
> **Analogy: Props are like method parameters in Java.** When you call `shipmentService.getShipment(trackingId)`, `trackingId` is a parameter. When you use `<ShipmentCard trackingId="TRK-001" />`, `trackingId` is a prop. Same concept, different syntax.

### TypeScript Interfaces for Props (Like DTOs)

Define your props using TypeScript interfaces — they serve the same purpose as Java DTOs:

```tsx
// Java DTO equivalent
// public record ShipmentDTO(String trackingId, String origin, ...) {}

// TypeScript interface for props
interface ShipmentProps {
  trackingId: string;           // Required — like @NotNull in Java
  origin: string;
  destination: string;
  status: 'pending' | 'in-transit' | 'delivered';  // Union type — like an enum
  weight: number;
  estimatedArrival?: Date;      // Optional — like @Nullable
  tags?: string[];              // Optional array
}
```

**Java vs TypeScript Type Comparison:**

| Java Type | TypeScript Equivalent |
|---|---|
| `String` | `string` (lowercase!) |
| `int`, `long`, `double` | `number` |
| `boolean` | `boolean` |
| `Date` | `Date` |
| `List<String>` | `string[]` |
| `Map<String, Object>` | `Record<string, unknown>` |
| `Optional<String>` | `string \| undefined` or `string?` |
| `enum Status { ... }` | `'pending' \| 'in-transit' \| 'delivered'` |

### Destructuring Props

Instead of accessing `props.trackingId`, destructure directly in the function signature:

```tsx
// ❌ Verbose — accessing props.X every time
function ShipmentCard(props: ShipmentProps) {
  return (
    <div>
      <h3>{props.trackingId}</h3>
      <p>{props.origin} → {props.destination}</p>
    </div>
  );
}

// ✅ Clean — destructure in the function signature
function ShipmentCard({ trackingId, origin, destination, status }: ShipmentProps) {
  return (
    <div>
      <h3>{trackingId}</h3>
      <p>{origin} → {destination}</p>
      <span>{status}</span>
    </div>
  );
}
```

### Default Values for Optional Props

```tsx
interface PaginationProps {
  totalItems: number;
  pageSize?: number;      // Optional with default
  currentPage?: number;
}

function Pagination({
  totalItems,
  pageSize = 20,          // Default to 20 if not provided
  currentPage = 1,        // Default to page 1
}: PaginationProps) {
  const totalPages = Math.ceil(totalItems / pageSize);

  return (
    <div className="pagination">
      <span>Page {currentPage} of {totalPages}</span>
      <span>({totalItems} total items)</span>
    </div>
  );
}

// Usage — pageSize and currentPage default to 20 and 1
<Pagination totalItems={150} />
<Pagination totalItems={150} pageSize={50} currentPage={3} />
```

### The `children` Prop — Composition Pattern

The `children` prop is a special prop that contains **whatever you put between the opening and closing tags** of a component. This enables composition — the React equivalent of Java's template method pattern.

```tsx
// A reusable Card wrapper component
interface CardProps {
  title: string;
  children: React.ReactNode;  // Accepts any valid JSX as children
}

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <div className="card-header">
        <h3>{title}</h3>
      </div>
      <div className="card-body">
        {children}  {/* Whatever the parent puts inside <Card>...</Card> */}
      </div>
    </div>
  );
}

// Usage — different content inside the same Card layout
function Dashboard() {
  return (
    <div>
      <Card title="Shipment Summary">
        <p>Total shipments: 142</p>
        <p>In transit: 38</p>
        <p>Delivered today: 17</p>
      </Card>

      <Card title="Recent Activity">
        <ul>
          <li>TRK-001 departed Sydney</li>
          <li>TRK-002 arrived Melbourne</li>
        </ul>
      </Card>
    </div>
  );
}
```

### Props vs Java Constructor Injection

```
Spring Boot (Constructor Injection):      React (Props):
────────────────────────────────          ──────────────
@Service                                  interface DashboardProps {
public class ShipmentService {              shipments: Shipment[];
  private final ShipmentRepo repo;          onRefresh: () => void;
  private final CarrierClient client;     }

  // Dependencies injected via constructor  // Props passed by parent component
  public ShipmentService(                 function Dashboard({
    ShipmentRepo repo,                      shipments,
    CarrierClient client                    onRefresh,
  ) {                                     }: DashboardProps) {
    this.repo = repo;                       return (
    this.client = client;                     <div>
  }                                             {shipments.map(s =>
}                                                 <ShipmentCard key={s.id} {...s} />
                                                )}
// Usage — Spring container injects deps        <button onClick={onRefresh}>
@Autowired ShipmentService service;               Refresh
                                                </button>
                                              </div>
// Usage — parent component passes props      );
<Dashboard                                  }
  shipments={shipmentList}
  onRefresh={handleRefresh}
/>
```

---

## Phase 4: Component Composition

### Composition Over Inheritance

React strongly favors **composition over inheritance**. You should almost never use `extends` or class inheritance in React. Instead, compose components by nesting them and using the `children` prop.

> [!warning]
> React's own documentation says: "We have not found any use cases where we would recommend creating component inheritance hierarchies." Use composition patterns instead.

### Pattern 1: Container / Presentational

Separate **logic** (data fetching, state) from **UI** (rendering):

```tsx
// Presentational — only renders UI, receives everything via props
// Like a Thymeleaf template — just displays data
interface ShipmentTableProps {
  shipments: Shipment[];
  isLoading: boolean;
  onSort: (column: string) => void;
}

function ShipmentTable({ shipments, isLoading, onSort }: ShipmentTableProps) {
  if (isLoading) return <p>Loading shipments...</p>;

  return (
    <table>
      <thead>
        <tr>
          <th onClick={() => onSort('trackingId')}>Tracking ID</th>
          <th onClick={() => onSort('status')}>Status</th>
          <th onClick={() => onSort('origin')}>Origin</th>
        </tr>
      </thead>
      <tbody>
        {shipments.map((s) => (
          <tr key={s.trackingId}>
            <td>{s.trackingId}</td>
            <td>{s.status}</td>
            <td>{s.origin}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Container — handles logic, passes data to presentational component
// Like a Spring Controller — coordinates data flow
function ShipmentDashboard() {
  const [shipments, setShipments] = useState<Shipment[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [sortBy, setSortBy] = useState('trackingId');

  useEffect(() => {
    fetchShipments(sortBy).then((data) => {
      setShipments(data);
      setIsLoading(false);
    });
  }, [sortBy]);

  return (
    <ShipmentTable
      shipments={shipments}
      isLoading={isLoading}
      onSort={setSortBy}
    />
  );
}
```

### Pattern 2: Slot Pattern with `children`

Create layout components that accept content via `children`:

```tsx
// A reusable page layout — like a page template in Thymeleaf
interface PageLayoutProps {
  title: string;
  sidebar?: React.ReactNode;   // Optional slot for sidebar content
  children: React.ReactNode;   // Main content area
}

function PageLayout({ title, sidebar, children }: PageLayoutProps) {
  return (
    <div className="page-layout">
      <header>
        <h1>{title}</h1>
      </header>
      <div className="page-body">
        {sidebar && <aside className="sidebar">{sidebar}</aside>}
        <main className="content">{children}</main>
      </div>
    </div>
  );
}

// Usage — plug different content into the same layout
function ShipmentPage() {
  return (
    <PageLayout
      title="Shipments"
      sidebar={<FilterPanel />}
    >
      <ShipmentList />    {/* This becomes the `children` prop */}
      <Pagination />
    </PageLayout>
  );
}
```

### Pattern 3: Higher-Order Components (HOC)

A HOC is a **function that takes a component and returns a new enhanced component**. Like Java's **decorator pattern** or Spring's `@Transactional` wrapper.

```tsx
// HOC that adds loading state to any component
// Think of it like a Java decorator: withLoading(ShipmentTable) wraps ShipmentTable
function withLoading<P extends object>(
  WrappedComponent: React.ComponentType<P>
) {
  return function WithLoadingComponent(props: P & { isLoading: boolean }) {
    const { isLoading, ...restProps } = props;

    if (isLoading) {
      return <div className="spinner">Loading...</div>;
    }

    return <WrappedComponent {...(restProps as P)} />;
  };
}

// Usage — wrap any component with loading behavior
const ShipmentTableWithLoading = withLoading(ShipmentTable);

// Now ShipmentTableWithLoading automatically shows a spinner when isLoading=true
<ShipmentTableWithLoading
  isLoading={true}
  shipments={[]}
  onSort={handleSort}
/>
```

> [!note]
> HOCs were more common before Hooks. Today, custom hooks are preferred for most cases. But you'll encounter HOCs in older codebases, so it's good to understand the pattern.

### Pattern 4: Render Props

A component that takes a **function as a prop** and calls it to determine what to render:

```tsx
// A component that provides mouse position to any child
interface MouseTrackerProps {
  render: (position: { x: number; y: number }) => React.ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e: React.MouseEvent) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return (
    <div onMouseMove={handleMouseMove} style={{ height: '100%' }}>
      {render(position)}
    </div>
  );
}

// Usage — the parent decides HOW to render the mouse position
<MouseTracker
  render={({ x, y }) => (
    <p>Cursor at: ({x}, {y})</p>
  )}
/>
```

> [!note]
> Like HOCs, render props are less common now that Hooks exist. Custom hooks (e.g., `useMouse()`) are the modern equivalent. But understanding render props helps when reading older React code.

---

## Phase 5: Keys in Lists

### Why React Needs Keys

When React renders a list, it needs a way to **identify each item uniquely** so it can efficiently update the DOM when the list changes (items added, removed, or reordered).

```tsx
// ✅ Correct — use a stable, unique identifier
function ShipmentList({ shipments }: { shipments: Shipment[] }) {
  return (
    <ul>
      {shipments.map((shipment) => (
        <li key={shipment.trackingId}>
          <ShipmentCard {...shipment} />
        </li>
      ))}
    </ul>
  );
}
```

### What Happens Without Proper Keys

```
Scenario: User deletes the SECOND item from a list of 3

With unique keys (trackingId):         With index as key:
─────────────────────────────          ─────────────────────
Before:                                Before:
  key="TRK-001" → Item A                key=0 → Item A
  key="TRK-002" → Item B  ← DELETE      key=1 → Item B  ← DELETE
  key="TRK-003" → Item C                key=2 → Item C

After:                                 After:
  key="TRK-001" → Item A  (kept)        key=0 → Item A  (kept)
  key="TRK-003" → Item C  (kept)        key=1 → Item C  (REACT THINKS B CHANGED TO C)
                                                         (DESTROYS key=2 — was Item C!)

Result: 1 DOM removal (efficient)     Result: 2 DOM updates (wasteful + buggy)
```

### ✅ Good Key Choices

| Source | Example | When to Use |
|---|---|---|
| Database ID | `key={shipment.id}` | Data from an API/database |
| Tracking number | `key={shipment.trackingId}` | Business identifier |
| UUID | `key={crypto.randomUUID()}` | ⚠️ Only in `useMemo`, not on every render |
| Composite key | `key={`${origin}-${dest}-${date}`}` | When no single unique field exists |

### ❌ Bad Key Choices

| Anti-Pattern | Why It's Bad |
|---|---|
| `key={index}` | Breaks when list is reordered, filtered, or items are inserted/deleted |
| `key={Math.random()}` | Creates a NEW key every render → React destroys/recreates every item |
| No key at all | React warns in console; falls back to index (same problems) |

> [!warning]
> **Rule of thumb:** If your items have a unique ID (like `shipmentId` or `trackingNumber`), always use it. If they don't, create one when the data is first created (like generating a UUID when the user adds an item to a form).

---

## Phase 6: Class Components (Legacy)

### Why You Need to Know Them

Class components are the **old way** of writing React (pre-Hooks, before 2019). You'll encounter them in:
- Legacy codebases that haven't migrated
- Older tutorials, Stack Overflow answers, and documentation
- Libraries that expose class-based APIs

> [!note]
> You should **not** write new class components. Use functional components + Hooks for all new code. This section is for reading and understanding legacy code.

### Class Component Syntax

```tsx
import React from 'react';

// Props interface — same as functional components
interface ShipmentCardProps {
  trackingId: string;
  status: string;
}

// State interface — unique to class components
interface ShipmentCardState {
  isExpanded: boolean;
  noteCount: number;
}

// Class component extends React.Component<Props, State>
// Like: public class ShipmentCard extends JPanel { ... }
class ShipmentCard extends React.Component<ShipmentCardProps, ShipmentCardState> {
  // Constructor — initialize state
  constructor(props: ShipmentCardProps) {
    super(props);
    this.state = {
      isExpanded: false,
      noteCount: 0,
    };
  }

  // Methods — event handlers (must bind `this` or use arrow functions)
  handleToggle = () => {
    // setState() is the ONLY way to update state in class components
    this.setState((prevState) => ({
      isExpanded: !prevState.isExpanded,
    }));
  };

  // render() — the ONLY required method. Returns JSX.
  render() {
    // Access props via this.props, state via this.state
    const { trackingId, status } = this.props;
    const { isExpanded } = this.state;

    return (
      <div className="shipment-card" onClick={this.handleToggle}>
        <h3>{trackingId}</h3>
        <span>{status}</span>
        {isExpanded && <p>Details shown...</p>}
      </div>
    );
  }
}
```

### Class vs Functional — Side by Side

```
Class Component:                        Functional Component:
──────────────────                      ──────────────────────
class ShipmentCard                      function ShipmentCard(
  extends React.Component<P, S> {         { trackingId, status }: Props
                                        ) {
  constructor(props) {
    super(props);                         const [isExpanded, setIsExpanded]
    this.state = { isExpanded: false };     = useState(false);
  }

  handleToggle = () => {                  const handleToggle = () => {
    this.setState({                         setIsExpanded(prev => !prev);
      isExpanded: !this.state.isExpanded   };
    });
  };

  render() {                              return (
    return (                                <div onClick={handleToggle}>
      <div onClick={this.handleToggle}>       <h3>{trackingId}</h3>
        <h3>{this.props.trackingId}</h3>      {isExpanded && <p>Details</p>}
        {this.state.isExpanded &&            </div>
          <p>Details</p>}                  );
      </div>                             }
    );
  }
}
```

### Why Functional Components Won

| Aspect | Class Components | Functional Components |
|---|---|---|
| **Syntax** | Verbose (`this.props`, `this.state`, `this.setState`) | Clean (destructured props, `useState`) |
| **`this` binding** | Must bind methods or use arrow functions | No `this` — just closures |
| **Code reuse** | HOCs and render props (complex) | Custom hooks (simple) |
| **Bundle size** | Larger (class overhead) | Smaller |
| **Readability** | Harder (lifecycle methods scattered) | Easier (hooks in order of use) |
| **Testing** | Harder (instance methods, `this`) | Easier (plain functions) |
| **React team** | No longer adding features for classes | All new features are hooks-based |

---

## Phase 7: Best Practices

### ✅ Do

| Practice | Why |
|---|---|
| **Single responsibility** — one component, one job | Same as SRP in Java. A `<ShipmentCard>` renders a shipment card. That's it. |
| **Keep components small** (< 100 lines) | If a component file exceeds 100 lines, it probably does too much. Extract. |
| **Extract reusable components early** | If you copy-paste JSX, extract it into a component immediately |
| **Use TypeScript for ALL props** | Catch bugs at compile time. You wouldn't write Java without types. |
| **Name components with PascalCase** | `ShipmentCard`, not `shipmentCard` or `shipment_card` |
| **Name files to match component** | `ShipmentCard.tsx` exports `ShipmentCard` |
| **Use destructuring for props** | `({ trackingId, status })` not `(props)` |
| **Co-locate styles** | `ShipmentCard.tsx` + `ShipmentCard.css` in same directory |

### ❌ Don't

| Anti-Pattern | Why It's Bad |
|---|---|
| **God component** — one massive component for the whole page | Impossible to test, reuse, or maintain |
| **Prop drilling** — passing props through 5+ levels | Use Context API or state management instead |
| **Business logic in components** | Extract to custom hooks or utility functions |
| **Mutating props** | Props are read-only. Always. Period. |
| **Anonymous components** | `export default () => <div>...</div>` — hard to debug (no name in DevTools) |
| **Inline object/array creation in JSX** | `style={{ color: 'red' }}` creates a new object every render → unnecessary re-renders |

### Component Organization

```
src/
├── components/
│   ├── common/              # Shared/generic components (Button, Card, Modal)
│   │   ├── Button.tsx
│   │   ├── Button.css
│   │   ├── Card.tsx
│   │   └── Modal.tsx
│   ├── shipment/            # Feature-specific components
│   │   ├── ShipmentCard.tsx
│   │   ├── ShipmentCard.css
│   │   ├── ShipmentList.tsx
│   │   ├── ShipmentFilter.tsx
│   │   └── index.ts         # Barrel file for clean imports
│   └── layout/              # Layout components (Header, Footer, Sidebar)
│       ├── Header.tsx
│       ├── Footer.tsx
│       └── PageLayout.tsx
├── hooks/                   # Custom hooks
├── types/                   # Shared TypeScript types/interfaces
└── pages/                   # Page-level components (one per route)
```

> [!tip]
> **Barrel files** (`index.ts`) let you import cleanly:
> ```tsx
> // Without barrel file
> import { ShipmentCard } from './components/shipment/ShipmentCard';
> 
> // With barrel file (components/shipment/index.ts exports everything)
> import { ShipmentCard, ShipmentList, ShipmentFilter } from './components/shipment';
> ```

---

## Quick Reference Card

```
Component Anatomy:
──────────────────
1. Imports          → import { useState } from 'react';
2. Props interface  → interface Props { trackingId: string; }
3. Function         → function ShipmentCard({ trackingId }: Props) {
4. Hooks            →   const [open, setOpen] = useState(false);
5. Derived values   →   const label = open ? 'Hide' : 'Show';
6. Handlers         →   const handleClick = () => setOpen(!open);
7. Return JSX       →   return <div onClick={handleClick}>{label}</div>;
8. Export           → export default ShipmentCard;

Props Cheat Sheet:
──────────────────
Required prop:     trackingId: string
Optional prop:     estimatedArrival?: Date
Default value:     pageSize = 20 (in destructuring)
Children:          children: React.ReactNode
Callback:          onSearch: (query: string) => void
Spread:            <ShipmentCard {...shipment} />
```

---

## What's Next?

- [[State and Lifecycle]] — How to manage data that changes over time
- [[Hooks]] — `useState`, `useEffect`, `useMemo`, and custom hooks
- [[React Fundamentals]] — Review the basics if anything was unclear

> [!tip]
> **Practice exercise:** Build a `<FreightCalculator>` component that takes `origin`, `destination`, and `weight` as props, and displays an estimated cost. Add a `<CurrencyToggle>` child component that lets the user switch between USD and AUD. This will solidify props, composition, and callback patterns.
