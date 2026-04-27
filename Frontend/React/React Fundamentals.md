---
tags:
  - react
  - frontend
  - javascript
  - beginner
---

# React Fundamentals

> *"The best way to predict the future is to invent it."* — Alan Kay

---

## Phase 1: What is React?

### The One-Liner

React is an **open-source JavaScript library** for building user interfaces. It was created by **Facebook (Meta)** in 2013 and is maintained by Meta and a community of developers.

> [!note]
> React is a **library**, not a framework. It handles only the **view layer** (rendering UI). For routing, state management, HTTP calls, etc., you pick your own libraries. This is the opposite of Angular, which is a full framework with batteries included.

### Why React Exists

Before React, building dynamic UIs meant manually manipulating the DOM (Document Object Model) — the browser's internal representation of the page. This was slow, error-prone, and hard to maintain at scale.

React introduced a **declarative** approach: you describe **what** the UI should look like for a given state, and React figures out **how** to update the DOM efficiently.

```
Imperative (vanilla JS):                Declarative (React):
─────────────────────────               ────────────────────────
const el = document.                    function ShipmentStatus({ status }) {
  getElementById('status');               return <span>{status}</span>;
el.textContent = 'Delivered';           }
el.className = 'status-delivered';
// YOU manage every DOM mutation         // React manages DOM updates for you
```

### Component-Based Architecture

React apps are built from **components** — small, reusable, self-contained pieces of UI that can be composed together like building blocks.

```
┌─────────────────────────────────────────────┐
│                  <App>                       │
│  ┌───────────────────────────────────────┐  │
│  │            <Header />                  │  │
│  ├───────────────────────────────────────┤  │
│  │        <ShipmentDashboard>             │  │
│  │  ┌─────────┐  ┌───────────────────┐   │  │
│  │  │SearchBar│  │  <ShipmentList>    │   │  │
│  │  └─────────┘  │  ┌──────────────┐ │   │  │
│  │               │  │ ShipmentCard │ │   │  │
│  │               │  ├──────────────┤ │   │  │
│  │               │  │ ShipmentCard │ │   │  │
│  │               │  └──────────────┘ │   │  │
│  │               └───────────────────┘   │  │
│  ├───────────────────────────────────────┤  │
│  │            <Footer />                  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Analogy: React Components ≈ Java Classes

If you're coming from Java/Spring Boot, think of it this way:

| Java / Spring Boot | React |
|---|---|
| Classes are reusable units of business logic | Components are reusable units of UI |
| Classes have constructors (dependency injection) | Components have props (data passed in) |
| Classes have private fields (internal state) | Components have state (via `useState`) |
| Classes are composed into services/controllers | Components are composed into pages/layouts |
| Interfaces define contracts | TypeScript interfaces define prop shapes |
| `@Component` / `@Service` annotations | `export default function MyComponent()` |

> [!tip]
> If you can write a Java class, you can write a React component. The mental model is the same — **encapsulation, composition, reuse** — just applied to UI instead of business logic.

### React vs Angular — Quick Comparison

| Aspect | React | Angular |
|---|---|---|
| **Type** | Library (view only) | Full framework |
| **Language** | JavaScript / TypeScript (optional) | TypeScript (required) |
| **DOM** | Virtual DOM | Real DOM with change detection |
| **Data Binding** | One-way (top-down) | Two-way |
| **Learning Curve** | Gradual — pick what you need | Steep — learn the whole framework |
| **Flexibility** | Choose your own router, state, etc. | Built-in router, forms, HTTP, DI |
| **Created By** | Meta (Facebook) | Google |
| **Popularity (2024)** | #1 by npm downloads | #2 |

> For a deep dive, see [[React vs Angular Comparison]].

---

## Phase 2: JSX — JavaScript + HTML

### What is JSX?

JSX (JavaScript XML) is a **syntax extension** that lets you write HTML-like code inside JavaScript. It looks like HTML, but it's actually JavaScript.

```tsx
// This is JSX — looks like HTML, lives in a .tsx file
function Greeting() {
  return <h1>Hello, Freight World!</h1>;
}
```

Under the hood, the JSX compiler (Babel/SWC) transforms it into regular JavaScript:

```tsx
// What the compiler actually produces
function Greeting() {
  return React.createElement('h1', null, 'Hello, Freight World!');
}
```

> [!note]
> You never write `React.createElement()` by hand. JSX is syntactic sugar that makes component code readable. Think of it like Lombok in Java — you write less, the compiler generates the rest.

### JSX Rules

| Rule | Why | Example |
|---|---|---|
| **Single root element** | JSX must return one parent element | Wrap in `<div>` or `<>...</>` (Fragment) |
| **`className` not `class`** | `class` is a reserved word in JS | `<div className="card">` |
| **`htmlFor` not `for`** | `for` is a reserved word in JS | `<label htmlFor="name">` |
| **Self-closing tags** | Tags without children must self-close | `<img />`, `<br />`, `<input />` |
| **`{}` for expressions** | Embed any JS expression in curly braces | `<span>{shipment.status}</span>` |
| **camelCase attributes** | HTML attributes become camelCase | `onClick`, `onChange`, `tabIndex` |

### Embedding JavaScript Expressions

Use curly braces `{}` to embed any JavaScript expression inside JSX:

```tsx
function ShipmentSummary({ shipment }: { shipment: Shipment }) {
  // Variables
  const totalWeight = shipment.items.reduce((sum, item) => sum + item.weight, 0);
  
  return (
    <div className="summary">
      {/* String interpolation */}
      <h2>Tracking: {shipment.trackingId}</h2>
      
      {/* Computed values */}
      <p>Total weight: {totalWeight} kg</p>
      
      {/* Method calls */}
      <p>Origin: {shipment.origin.toUpperCase()}</p>
      
      {/* Ternary expressions */}
      <span>{shipment.isUrgent ? '🔴 URGENT' : '🟢 Standard'}</span>
      
      {/* Date formatting */}
      <p>ETA: {new Date(shipment.eta).toLocaleDateString()}</p>
    </div>
  );
}
```

> [!warning]
> You can only embed **expressions** (things that produce a value), not **statements**. No `if/else`, `for`, `switch` inside `{}`. Use ternary operators or extract logic above the `return`.

### Conditional Rendering

Three common patterns for showing/hiding UI based on conditions:

```tsx
function ShipmentStatus({ status, isDelayed }: ShipmentStatusProps) {
  // Pattern 1: Ternary operator — when you have two alternatives
  return (
    <div>
      {status === 'delivered' 
        ? <span className="success">✅ Delivered</span>
        : <span className="pending">⏳ In Transit</span>
      }
      
      {/* Pattern 2: Logical AND (&&) — when you only show something conditionally */}
      {isDelayed && <span className="warning">⚠️ Delayed</span>}
    </div>
  );
}

// Pattern 3: Early return — when the whole component shouldn't render
function ShipmentCard({ shipment }: { shipment: Shipment | null }) {
  if (!shipment) {
    return <p>No shipment data available.</p>;
  }
  
  return (
    <div className="shipment-card">
      <h3>{shipment.trackingId}</h3>
      <p>{shipment.status}</p>
    </div>
  );
}
```

### Lists and Keys — Rendering Arrays

Use `.map()` to render a list of items. Every list item needs a unique `key` prop:

```tsx
function ShipmentList({ shipments }: { shipments: Shipment[] }) {
  return (
    <ul>
      {shipments.map((shipment) => (
        // key helps React identify which items changed/added/removed
        // Use a unique, stable ID — NEVER use array index
        <li key={shipment.trackingId}>
          <ShipmentCard shipment={shipment} />
        </li>
      ))}
    </ul>
  );
}
```

**Why keys matter:**

```
Without keys — React re-renders ALL items:
[Item A] [Item B] [Item C]  →  insert Item X at position 1
[Item X] [Item A] [Item B] [Item C]  ← React thinks ALL items changed

With keys — React knows exactly what changed:
[A:key1] [B:key2] [C:key3]  →  insert X:key4 at position 1
[X:key4] [A:key1] [B:key2] [C:key3]  ← React only inserts one new node
```

> [!warning]
> **Anti-pattern:** Using array index as key (`key={index}`). If the list is reordered, filtered, or items are inserted, indices shift and React confuses items. Always use a stable unique ID (like `trackingId`, `shipmentId`, database primary key).

---

## Phase 3: Setting Up a React Project

### Using Vite (Recommended)

Vite is the modern, fast build tool for React. It replaced Create React App (CRA), which is now deprecated.

```bash
# Create a new React project with TypeScript template
npm create vite@latest my-app -- --template react-ts

# Navigate into the project and install dependencies
cd my-app && npm install

# Start the development server (hot-reload at http://localhost:5173)
npm run dev
```

> [!tip]
> Always use the `react-ts` template. TypeScript catches bugs at compile time, just like Java's type system. You'll feel right at home.

### Project Structure

```
my-app/
├── public/                  # Static assets (favicon, images)
│   └── vite.svg
├── src/                     # All source code lives here
│   ├── assets/              # Images, fonts imported by components
│   ├── components/          # Reusable UI components
│   │   ├── Header.tsx
│   │   ├── ShipmentCard.tsx
│   │   └── SearchBar.tsx
│   ├── hooks/               # Custom React hooks
│   ├── types/               # TypeScript type definitions
│   │   └── shipment.ts
│   ├── App.tsx              # Root component (like your @SpringBootApplication)
│   ├── App.css              # Styles for App component
│   ├── main.tsx             # Entry point (like public static void main)
│   └── index.css            # Global styles
├── index.html               # Single HTML page (SPA entry point)
├── package.json             # Dependencies and scripts (like pom.xml)
├── tsconfig.json            # TypeScript config (like compiler settings)
└── vite.config.ts           # Build tool config (like Maven plugins)
```

### Spring Boot vs React — Project Structure Comparison

```
Spring Boot:                          React:
──────────────────────                ──────────────────────
src/main/java/                        src/
  com/company/app/                      components/
    controller/                           ShipmentCard.tsx
      ShipmentController.java             Header.tsx
    service/                            hooks/
      ShipmentService.java                useShipments.ts
    repository/                         services/
      ShipmentRepository.java             shipmentApi.ts
    model/                              types/
      Shipment.java                       shipment.ts
    dto/                                pages/
      ShipmentDTO.java                    Dashboard.tsx

src/main/resources/                   public/
  application.yml                       favicon.ico

pom.xml                               package.json
                                       tsconfig.json
                                       vite.config.ts
```

| Spring Boot Concept | React Equivalent |
|---|---|
| `pom.xml` (Maven dependencies) | `package.json` (npm dependencies) |
| `mvn spring-boot:run` | `npm run dev` |
| `mvn clean install` | `npm run build` |
| `application.yml` | `.env` files or `vite.config.ts` |
| `@SpringBootApplication` (entry) | `main.tsx` (entry) |
| `@RestController` (endpoint) | Page component (route handler) |
| DTO class | TypeScript `interface` |
| `@Service` class | Custom hook or service module |

---

## Phase 4: The Virtual DOM

### What is the DOM?

The DOM (Document Object Model) is the browser's in-memory representation of the HTML page. Every element (`<div>`, `<p>`, `<span>`) is a node in a tree. Manipulating the real DOM is **slow** because the browser must recalculate layouts, repaint pixels, and reflow elements.

### What is the Virtual DOM?

The Virtual DOM is a **lightweight JavaScript copy** of the real DOM that React keeps in memory. When state changes, React:

1. Creates a **new** Virtual DOM tree
2. **Diffs** the new tree against the previous one
3. Calculates the **minimum set of changes** needed
4. Applies **only those changes** to the real DOM (batch update)

```
State Change Triggered
        │
        ▼
┌──────────────────┐
│   New Virtual DOM │ ◄── React re-renders component in memory
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────┐
│        DIFFING (Reconciliation)   │
│                                   │
│   Old Virtual DOM    New Virtual  │
│   ┌──────────┐      ┌──────────┐ │
│   │   <div>  │      │   <div>  │ │
│   │  ┌─────┐ │      │  ┌─────┐ │ │
│   │  │ "A" │ │  vs   │  │ "B" │ │ │  ◄── Only this text changed
│   │  └─────┘ │      │  └─────┘ │ │
│   │  ┌─────┐ │      │  ┌─────┐ │ │
│   │  │ "C" │ │      │  │ "C" │ │ │  ◄── Same, skip
│   │  └─────┘ │      │  └─────┘ │ │
│   └──────────┘      └──────────┘ │
│                                   │
│   Diff result: Change "A" → "B"  │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       COMMIT TO REAL DOM          │
│                                   │
│   Only patch the single text node │
│   "A" → "B" (minimal mutation)   │
└──────────────────────────────────┘
```

### Why is the Virtual DOM Fast?

| Technique | What It Does |
|---|---|
| **Batching** | Multiple state changes in one event handler are batched into a single re-render |
| **Minimal Mutations** | Only the changed nodes are updated in the real DOM |
| **Asynchronous Rendering** | React can prioritize urgent updates (e.g., input) over less urgent ones (e.g., list re-sort) |
| **Tree Pruning** | If a subtree hasn't changed, React skips it entirely |

### Analogy: Shipping Manifest Diff

> [!tip]
> Imagine you have a shipping manifest with 500 line items. One item's delivery address changes. You don't **reprint the entire manifest** — you print only the **change slip** for that one line item and attach it. That's what React's Virtual DOM does — it computes the **diff** and applies only the minimal patch to the real DOM.

```
Old Manifest (Virtual DOM v1):          New Manifest (Virtual DOM v2):
──────────────────────────────          ──────────────────────────────
Line 001: Item A → Sydney    ──same──   Line 001: Item A → Sydney
Line 002: Item B → Melbourne ──DIFF──   Line 002: Item B → Brisbane   ← CHANGED
Line 003: Item C → Perth     ──same──   Line 003: Item C → Perth

Real DOM update: Only update line 002's destination from Melbourne → Brisbane
```

---

## Phase 5: React Developer Tools

### Installation

React Developer Tools is a browser extension available for **Chrome** and **Firefox**. Install it from:
- Chrome: [Chrome Web Store — React Developer Tools](https://chrome.google.com/webstore/detail/react-developer-tools)
- Firefox: [Firefox Add-ons — React Developer Tools](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/)

After installation, two new tabs appear in your browser's DevTools: **⚛️ Components** and **⚛️ Profiler**.

### Components Tab

Lets you inspect the **component tree** — similar to the Spring Boot Actuator's bean graph, but for UI components.

```
What you see in the Components tab:

⚛️ Components
├── <App>
│   ├── <Header>
│   │   props: { title: "Freight Dashboard" }
│   ├── <ShipmentDashboard>
│   │   state: { shipments: [...], loading: false }
│   │   ├── <SearchBar>
│   │   │   props: { onSearch: ƒ }
│   │   ├── <ShipmentList>
│   │   │   props: { shipments: [...] }
│   │   │   ├── <ShipmentCard> props: { trackingId: "TRK-001", status: "in-transit" }
│   │   │   ├── <ShipmentCard> props: { trackingId: "TRK-002", status: "delivered" }
│   │   │   └── <ShipmentCard> props: { trackingId: "TRK-003", status: "pending" }
│   │   └── <Pagination>
│   └── <Footer>
```

**Key features:**
- Click any component to see its **props** and **state** in the right panel
- Edit props/state live to test different scenarios
- Search for components by name
- Highlight updates — see which components re-render on each action

### Profiler Tab

Records **performance data** during interactions. Shows:
- Which components rendered and **why**
- How long each render took (in milliseconds)
- Flame chart of the render tree

> [!tip]
> The Profiler is like JVisualVM or YourKit for React. Use it to find slow components, unnecessary re-renders, and performance bottlenecks. Start profiling → interact with the app → stop → analyze the flame chart.

### Common DevTools Workflow

| Task | How |
|---|---|
| Find a component | Use the element picker (🔍) or search by name |
| Inspect props | Click component → right panel shows all props |
| Debug state | Click component → see current state values |
| Test changes | Edit a prop/state value in the panel → UI updates instantly |
| Find re-renders | Enable "Highlight updates" in ⚙️ settings |
| Profile performance | Profiler tab → Record → Interact → Stop → Analyze |

---

## Phase 6: Your First React Component

### The Anatomy of a Functional Component

A React component is just a **function that returns JSX**. Here's the full anatomy:

```tsx
// 1. Imports — like Java import statements
import { useState } from 'react';
import './ShipmentCard.css';

// 2. Props interface — like a Java DTO (Data Transfer Object)
interface ShipmentCardProps {
  trackingId: string;
  origin: string;
  destination: string;
  status: 'pending' | 'in-transit' | 'delivered';
  weight: number;
  estimatedArrival?: Date;  // Optional prop (like @Nullable in Java)
}

// 3. Component function — accepts props, returns JSX
function ShipmentCard({
  trackingId,
  origin,
  destination,
  status,
  weight,
  estimatedArrival,
}: ShipmentCardProps) {
  // 4. State — internal data that can change (like private fields)
  const [isExpanded, setIsExpanded] = useState(false);

  // 5. Derived values — computed from props/state (like getter methods)
  const statusEmoji = status === 'delivered' ? '✅' : status === 'in-transit' ? '🚚' : '⏳';
  const isHeavy = weight > 1000;

  // 6. Event handlers — functions triggered by user actions
  const handleToggle = () => {
    setIsExpanded((prev) => !prev);
  };

  // 7. JSX return — the UI this component renders
  return (
    <div className={`shipment-card status-${status}`}>
      <div className="card-header" onClick={handleToggle}>
        <h3>{statusEmoji} {trackingId}</h3>
        <span className="route">{origin} → {destination}</span>
      </div>

      {/* Conditional rendering — show details only when expanded */}
      {isExpanded && (
        <div className="card-details">
          <p>Weight: {weight} kg {isHeavy && '⚠️ Heavy'}</p>
          <p>Status: {status}</p>
          {estimatedArrival && (
            <p>ETA: {estimatedArrival.toLocaleDateString()}</p>
          )}
        </div>
      )}
    </div>
  );
}

// 8. Export — makes the component available to other files
export default ShipmentCard;
```

### Mapping to Java Concepts

```
Java Class                              React Component
──────────────────────────────          ──────────────────────────────
public class ShipmentCard {             function ShipmentCard(props) {
  // Constructor params = Props            // Props = function params
  private String trackingId;               const { trackingId, status } = props;

  // Private fields = State                // State via hooks
  private boolean isExpanded;              const [isExpanded, setIsExpanded]
                                              = useState(false);

  // Methods = Event handlers              // Event handlers
  public void toggle() {                   const handleToggle = () => {
    this.isExpanded = !isExpanded;            setIsExpanded(prev => !prev);
  }                                        };

  // toString() = render (JSX)             // Return JSX
  public String render() {                 return (
    return "<div>...</div>";                 <div>...</div>
  }                                        );
}                                       }
```

### Using Your Component

```tsx
// In App.tsx or any parent component
import ShipmentCard from './components/ShipmentCard';

function App() {
  return (
    <div className="app">
      <h1>Freight Dashboard</h1>

      {/* Using the component — pass data via props */}
      <ShipmentCard
        trackingId="TRK-2024-001"
        origin="Sydney"
        destination="Melbourne"
        status="in-transit"
        weight={450}
        estimatedArrival={new Date('2024-12-15')}
      />

      <ShipmentCard
        trackingId="TRK-2024-002"
        origin="Shanghai"
        destination="Los Angeles"
        status="delivered"
        weight={12500}
      />

      <ShipmentCard
        trackingId="TRK-2024-003"
        origin="Rotterdam"
        destination="Mumbai"
        status="pending"
        weight={780}
      />
    </div>
  );
}
```

### Quick Summary Cheat Sheet

| Concept | What It Means | Java Analogy |
|---|---|---|
| **Component** | A function returning UI (JSX) | A class with a `render()` method |
| **JSX** | HTML-like syntax in JavaScript | Template engine (Thymeleaf) |
| **Props** | Read-only inputs from parent | Constructor parameters / DI |
| **State** | Internal mutable data | Private fields |
| **Virtual DOM** | In-memory UI diff engine | — (no direct equivalent) |
| **Re-render** | React re-runs the component function | Calling `render()` again |
| **Key** | Unique identifier for list items | Primary key in a database table |

---

## What's Next?

Now that you understand the fundamentals, move on to:

- [[Components and Props]] — Deep dive into building and composing components
- [[State and Lifecycle]] — Managing data that changes over time
- [[Hooks]] — The modern way to add features to functional components
- [[Frontend Fundamentals]] — HTML, CSS, and JavaScript basics if you need a refresher

> [!tip]
> **Learning path for Java devs:** React Fundamentals → Components and Props → State and Lifecycle → Hooks → React Router → API Integration (you'll feel at home with `fetch` — it's like RestTemplate/WebClient but for the browser).
