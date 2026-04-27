---
tags:
  - react
  - styling
  - css
  - tailwind
  - frontend
  - phase3
created: 2025-07-18
---

> *"Good design is as little design as possible — less, but better."* — Dieter Rams

# React Styling

In a warehouse, every crate needs a **label**. Plain text on cardboard works for small operations, but at scale you need colour-coded stickers, barcodes, and standardised packaging. Styling in React follows the same progression: you start simple (inline), then graduate to systems that scale with the size of your UI.

This guide walks through **every major styling approach** in React — from the equivalent of hand-written shipping labels all the way to an automated labelling line.

```
┌──────────────────────────────────────────────────────┐
│                  React Styling Roadmap                │
├──────────────────────────────────────────────────────┤
│  Phase 1 │ Inline Styles         (hand-written tag)  │
│  Phase 2 │ CSS Files             (shared label roll)  │
│  Phase 3 │ CSS Modules           (per-crate labels)   │
│  Phase 4 │ styled-components     (smart label maker)  │
│  Phase 5 │ Tailwind CSS          (utility label kit)  │
│  Phase 6 │ Comparison & Advice                        │
└──────────────────────────────────────────────────────┘
```

---

## Phase 1 — Inline Styles

> [!info] Analogy
> Inline styles are like **hand-writing the destination address** directly on a parcel. Quick for one-off shipments, terrible at scale.

In React, inline styles are passed as a **JavaScript object** to the `style` prop. Properties use **camelCase** (not kebab-case like regular CSS).

### Syntax

```tsx
// ✅ React inline style — camelCase, values as strings or numbers
function ShipmentAlert() {
  return (
    <div
      style={{
        backgroundColor: '#fff3cd',
        color: '#856404',
        padding: '12px 16px',
        borderRadius: '4px',
        borderLeft: '4px solid #ffc107',
        fontWeight: 600,
      }}
    >
      ⚠️ Shipment #4821 is delayed — ETA updated to July 20
    </div>
  );
}
```

### Dynamic Inline Styles

Inline styles shine when the value depends on **runtime data** — like colouring a status indicator based on a shipment's state:

```tsx
function StatusDot({ status }: { status: string }) {
  // Pick colour dynamically — like routing a parcel to different lanes
  const colorMap: Record<string, string> = {
    pending:   '#f59e0b',  // amber
    in_transit: '#3b82f6', // blue
    delivered: '#10b981',  // green
    cancelled: '#ef4444',  // red
  };

  return (
    <span
      style={{
        display: 'inline-block',
        width: 10,
        height: 10,
        borderRadius: '50%',
        backgroundColor: colorMap[status] ?? '#9ca3af',
      }}
    />
  );
}
```

### Limitations

| Feature               | Supported? | Notes                               |
| --------------------- | ---------- | ----------------------------------- |
| Pseudo-classes `:hover` | ❌         | Cannot use `:hover`, `:focus`, etc. |
| Media queries          | ❌         | No responsive breakpoints           |
| Animations / keyframes | ❌         | Cannot define `@keyframes`          |
| Pseudo-elements `::before` | ❌    | Not accessible via inline styles    |
| Specificity conflicts  | ✅         | Inline always wins (highest spec.)  |

> [!warning] When to avoid
> If you need hover states, responsive layouts, or animations — **do not** use inline styles. They are the "hand-written label" approach: fast for prototypes, painful for production.

---

## Phase 2 — CSS Files (Global Stylesheets)

> [!info] Analogy
> A global CSS file is like a **single label printer shared by every department** in your warehouse. Everyone's labels come out of the same machine — and if two departments define a `.status` label, they clash.

### Basic Usage

Create a CSS file and import it directly into your component:

```css
/* ShipmentCard.css */
.card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 600;
  color: #111827;
}

.card-status {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
}

.card-status--delivered {
  background: #d1fae5;
  color: #065f46;
}

.card-status--pending {
  background: #fef3c7;
  color: #92400e;
}
```

```tsx
// ShipmentCard.tsx
import './ShipmentCard.css';

function ShipmentCard({ shipmentId, destination, status }: Props) {
  return (
    <div className="card">
      <h3 className="card-title">Shipment #{shipmentId}</h3>
      <p>Destination: {destination}</p>
      <span className={`card-status card-status--${status}`}>
        {status}
      </span>
    </div>
  );
}
```

### The Global Scope Problem

```
┌──────────────────────────────────────────────────────────┐
│                   Global CSS = Shared Context             │
│                                                          │
│   ShipmentCard.css    defines  .card { padding: 16px }   │
│   InvoiceCard.css     defines  .card { padding: 24px }   │
│                                                          │
│   Result: CONFLICT — last import wins, styles leak       │
│                                                          │
│   Like having ALL Spring beans in one ApplicationContext  │
│   with no @Qualifier — name collisions everywhere        │
└──────────────────────────────────────────────────────────┘
```

> [!warning] Global CSS is like a monolith
> In Spring Boot, you isolate beans with `@Qualifier` or separate `@Configuration` classes. Global CSS has **no isolation**. Class names from one component bleed into another. This becomes unmanageable in large apps — just like a monolithic deployment pipeline for 50 microservices.

### When Global CSS is Fine

- ✅ Small apps or prototypes
- ✅ Global resets / base typography (`normalize.css`)
- ✅ Truly global styles (body background, font family)
- ❌ Component-specific styles in a multi-team project

---

## Phase 3 — CSS Modules

> [!info] Analogy
> CSS Modules are like giving **each warehouse zone its own label printer**. The Inbound team's `.status` label and the Outbound team's `.status` label never collide because the machine stamps each with a unique zone code.

### How They Work

CSS Modules are regular CSS files with a `.module.css` extension. At build time, the bundler (Vite, Webpack) **hashes every class name** to make it unique.

```
  Source:    .card { ... }
  Compiled:  .Shipment_card__x7k2q { ... }
```

This is compile-time scoping — like how Spring's component scanning prefixes bean names with the package to avoid collisions.

### Full Example — ShipmentCard

```css
/* ShipmentCard.module.css */
.card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: box-shadow 0.2s ease;
}

.card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.shipmentId {
  font-size: 1.125rem;
  font-weight: 600;
  color: #111827;
}

.badge {
  display: inline-block;
  padding: 2px 10px;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.delivered {
  composes: badge;
  background: #d1fae5;
  color: #065f46;
}

.inTransit {
  composes: badge;
  background: #dbeafe;
  color: #1e40af;
}

.pending {
  composes: badge;
  background: #fef3c7;
  color: #92400e;
}

.details {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  font-size: 0.875rem;
  color: #6b7280;
}

.label {
  font-weight: 500;
  color: #374151;
}
```

```tsx
// ShipmentCard.tsx
import styles from './ShipmentCard.module.css';

interface ShipmentCardProps {
  id: string;
  origin: string;
  destination: string;
  status: 'delivered' | 'in_transit' | 'pending';
  eta: string;
}

// Map status to the correct CSS Module class
const statusClass: Record<string, string> = {
  delivered:  styles.delivered,
  in_transit: styles.inTransit,
  pending:    styles.pending,
};

export function ShipmentCard({ id, origin, destination, status, eta }: ShipmentCardProps) {
  return (
    <div className={styles.card}>
      <div className={styles.header}>
        <span className={styles.shipmentId}>#{id}</span>
        <span className={statusClass[status]}>{status.replace('_', ' ')}</span>
      </div>
      <div className={styles.details}>
        <span className={styles.label}>Origin</span>
        <span>{origin}</span>
        <span className={styles.label}>Destination</span>
        <span>{destination}</span>
        <span className={styles.label}>ETA</span>
        <span>{eta}</span>
      </div>
    </div>
  );
}
```

### Composing Styles

The `composes` keyword lets you share styles between classes — like a base shipping label template that each carrier customises:

```css
/* base.module.css */
.baseCard {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
}

/* ShipmentCard.module.css */
.card {
  composes: baseCard from './base.module.css';
  background: #ffffff;
}
```

> [!tip] CSS Modules — Key Takeaways
> - ✅ Scoped by default — no class name collisions
> - ✅ Works with existing CSS knowledge
> - ✅ Zero runtime cost (resolved at build time)
> - ✅ Supports `:hover`, media queries, animations — full CSS
> - ❌ Slightly more verbose import syntax
> - ❌ Dynamic styles still need inline or class toggling

---

## Phase 4 — styled-components (CSS-in-JS)

> [!info] Analogy
> `styled-components` is like a **smart label maker** connected to your WMS. It reads shipment data (props) and prints the correct label automatically — colour, size, warnings — all driven by data.

### Installation

```bash
npm install styled-components
npm install -D @types/styled-components   # TypeScript types
```

### Basic Usage

```tsx
import styled from 'styled-components';

// A styled div — like defining a label template
const Card = styled.div`
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);

  &:hover {
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  }
`;

const Title = styled.h3`
  font-size: 1.125rem;
  font-weight: 600;
  color: #111827;
  margin: 0 0 8px 0;
`;

function ShipmentCard({ id, destination }: { id: string; destination: string }) {
  return (
    <Card>
      <Title>Shipment #{id}</Title>
      <p>Destination: {destination}</p>
    </Card>
  );
}
```

### Props-Based Dynamic Styling

This is where styled-components truly shines — styling driven by **component props**, like a label printer that adjusts colour based on shipment priority:

```tsx
interface StatusBadgeProps {
  status: 'delivered' | 'in_transit' | 'pending' | 'cancelled';
}

const statusColors: Record<string, { bg: string; text: string }> = {
  delivered:  { bg: '#d1fae5', text: '#065f46' },
  in_transit: { bg: '#dbeafe', text: '#1e40af' },
  pending:    { bg: '#fef3c7', text: '#92400e' },
  cancelled:  { bg: '#fee2e2', text: '#991b1b' },
};

const StatusBadge = styled.span<StatusBadgeProps>`
  display: inline-block;
  padding: 4px 12px;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: ${({ status }) => statusColors[status]?.bg ?? '#f3f4f6'};
  color: ${({ status }) => statusColors[status]?.text ?? '#374151'};
`;

// Usage — the label adapts to the shipment status automatically
<StatusBadge status="delivered">Delivered</StatusBadge>
<StatusBadge status="in_transit">In Transit</StatusBadge>
<StatusBadge status="cancelled">Cancelled</StatusBadge>
```

### Extending Styles

Like extending a base `@Configuration` class in Spring:

```tsx
const BaseButton = styled.button`
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.2s;

  &:hover {
    opacity: 0.85;
  }
`;

// Extend — like @Bean overriding a default
const ConfirmButton = styled(BaseButton)`
  background: #10b981;
  color: white;
`;

const CancelButton = styled(BaseButton)`
  background: #ef4444;
  color: white;
`;

const TrackButton = styled(BaseButton)`
  background: #3b82f6;
  color: white;
`;
```

### Theming with ThemeProvider

Theming in styled-components is like having a **global configuration** (think `application.yml`) that every label template reads from:

```tsx
import { ThemeProvider } from 'styled-components';

// Define the theme — like application.yml for your UI
const logisticsTheme = {
  colors: {
    primary:   '#2563eb',
    success:   '#10b981',
    warning:   '#f59e0b',
    danger:    '#ef4444',
    bgPrimary: '#ffffff',
    bgSecondary: '#f9fafb',
    text:      '#111827',
    textMuted: '#6b7280',
    border:    '#e5e7eb',
  },
  spacing: {
    xs: '4px',
    sm: '8px',
    md: '16px',
    lg: '24px',
    xl: '32px',
  },
  borderRadius: '8px',
};

// Wrap your app
function App() {
  return (
    <ThemeProvider theme={logisticsTheme}>
      <Dashboard />
    </ThemeProvider>
  );
}

// Any styled component can access the theme
const TrackingPanel = styled.div`
  background: ${({ theme }) => theme.colors.bgSecondary};
  border: 1px solid ${({ theme }) => theme.colors.border};
  border-radius: ${({ theme }) => theme.borderRadius};
  padding: ${({ theme }) => theme.spacing.md};
`;
```

> [!tip] styled-components — Key Takeaways
> - ✅ True dynamic styling via props
> - ✅ Scoped automatically (unique class names at runtime)
> - ✅ Theming built-in
> - ✅ Full CSS support (pseudo-classes, media queries, nesting)
> - ❌ Runtime cost — styles generated in JS at render time
> - ❌ Larger bundle than pure CSS approaches
> - ❌ Requires learning a new API

---

## Phase 5 — Tailwind CSS

> [!info] Analogy
> Tailwind is like a **modular labelling kit** with pre-cut stickers for every property: colour, size, border, spacing. Instead of designing a label from scratch, you assemble one from standardised pieces. Fast, consistent, and your whole warehouse uses the same visual language.

### Setup with Vite

```bash
# 1. Install Tailwind and its dependencies
npm install -D tailwindcss @tailwindcss/vite

# 2. Add the plugin to vite.config.ts
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
});
```

```css
/* src/index.css — import Tailwind */
@import "tailwindcss";
```

### Example — Shipment Card with Utility Classes

```tsx
function ShipmentCard({ id, origin, destination, status, eta }: ShipmentCardProps) {
  // Map status to Tailwind colour classes — like routing rules
  const badgeClasses: Record<string, string> = {
    delivered:  'bg-green-100 text-green-800',
    in_transit: 'bg-blue-100 text-blue-800',
    pending:    'bg-amber-100 text-amber-800',
    cancelled:  'bg-red-100 text-red-800',
  };

  return (
    <div className="bg-white border border-gray-200 rounded-lg p-4 shadow-sm
                    hover:shadow-md transition-shadow duration-200">
      {/* Header row */}
      <div className="flex items-center justify-between mb-3">
        <span className="text-lg font-semibold text-gray-900">
          Shipment #{id}
        </span>
        <span className={`inline-block px-3 py-0.5 rounded-full text-xs
                         font-semibold uppercase tracking-wide
                         ${badgeClasses[status] ?? 'bg-gray-100 text-gray-600'}`}>
          {status.replace('_', ' ')}
        </span>
      </div>

      {/* Details grid */}
      <div className="grid grid-cols-2 gap-2 text-sm text-gray-500">
        <span className="font-medium text-gray-700">Origin</span>
        <span>{origin}</span>
        <span className="font-medium text-gray-700">Destination</span>
        <span>{destination}</span>
        <span className="font-medium text-gray-700">ETA</span>
        <span>{eta}</span>
      </div>

      {/* Action buttons */}
      <div className="flex items-center gap-2 mt-4 pt-3 border-t border-gray-100">
        <button className="px-3 py-1.5 bg-blue-600 text-white text-sm
                          font-medium rounded hover:bg-blue-700
                          transition-colors">
          Track
        </button>
        <button className="px-3 py-1.5 bg-gray-100 text-gray-700 text-sm
                          font-medium rounded hover:bg-gray-200
                          transition-colors">
          Details
        </button>
      </div>
    </div>
  );
}
```

### Tracking Timeline Example

```tsx
function TrackingTimeline({ events }: { events: TrackingEvent[] }) {
  return (
    <div className="space-y-4 pl-4 border-l-2 border-gray-200">
      {events.map((event, i) => (
        <div key={i} className="relative flex items-start gap-3">
          {/* Timeline dot */}
          <div className={`absolute -left-[21px] w-3 h-3 rounded-full border-2
                          border-white ${i === 0 ? 'bg-blue-500' : 'bg-gray-300'}`}
          />
          <div>
            <p className="text-sm font-medium text-gray-900">{event.title}</p>
            <p className="text-xs text-gray-500">{event.timestamp}</p>
            <p className="text-sm text-gray-600 mt-0.5">{event.location}</p>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Pros and Cons

| Aspect              | ✅ Pro                                      | ❌ Con                                     |
| ------------------- | ------------------------------------------- | ------------------------------------------ |
| Speed               | Extremely fast to build UIs                 | Long `className` strings get messy         |
| Consistency         | Design tokens enforced by config            | Devs can still pick arbitrary values       |
| File size           | Purges unused CSS — tiny prod bundle        | Dev build can feel large                   |
| Learning curve      | Low if you know CSS                         | Memorising utility names takes time        |
| Reusability         | Extract components, not classes             | No built-in "component style" abstraction  |
| Responsive          | `sm:`, `md:`, `lg:` prefixes built-in       | Breakpoint classes add to verbosity        |
| Dark mode           | `dark:` prefix out of the box               | Requires `darkMode` config                 |

### Custom Configuration

```ts
// tailwind.config.ts — extend with your logistics brand colours
import type { Config } from 'tailwindcss';

export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        // Custom freight status colours
        freight: {
          pending:   '#f59e0b',
          transit:   '#3b82f6',
          delivered: '#10b981',
          exception: '#ef4444',
        },
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
    },
  },
  plugins: [],
} satisfies Config;
```

Usage after custom config:

```tsx
<span className="bg-freight-delivered/20 text-freight-delivered">
  Delivered
</span>
```

---

## Phase 6 — Comparison & Recommendation

### Side-by-Side Comparison

```
┌───────────────────┬────────────┬─────────────┬──────────────┬──────────────┬──────────────┐
│     Approach      │  Scoping   │ Performance │   Dev Ex.    │ Learn Curve  │ Dynamic      │
├───────────────────┼────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ Inline Styles     │ Component  │ ✅ No CSS   │ ❌ Verbose   │ ✅ Trivial   │ ✅ Full      │
│                   │            │    parse    │              │              │              │
├───────────────────┼────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ CSS Files         │ ❌ Global  │ ✅ Native   │ ✅ Familiar  │ ✅ Trivial   │ ❌ Class     │
│                   │            │    CSS      │              │              │    toggle    │
├───────────────────┼────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ CSS Modules       │ ✅ Scoped  │ ✅ Build-   │ ✅ Good      │ ✅ Easy      │ ⚠️ Class     │
│                   │            │    time     │              │              │    toggle    │
├───────────────────┼────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ styled-components │ ✅ Scoped  │ ⚠️ Runtime │ ✅ Great     │ ⚠️ Medium   │ ✅ Full      │
│                   │            │    cost     │              │              │              │
├───────────────────┼────────────┼─────────────┼──────────────┼──────────────┼──────────────┤
│ Tailwind CSS      │ ✅ Utility │ ✅ Purged   │ ✅ Fast      │ ⚠️ Medium   │ ⚠️ Class     │
│                   │            │    CSS      │              │              │    concat    │
└───────────────────┴────────────┴─────────────┴──────────────┴──────────────┴──────────────┘
```

### Detailed Comparison Table

| Criteria        | Inline       | CSS Files     | CSS Modules     | styled-components | Tailwind        |
| --------------- | ------------ | ------------- | --------------- | ----------------- | --------------- |
| **Scoping**     | Component    | Global ❌     | Scoped ✅       | Scoped ✅         | Utility ✅      |
| **Performance** | No overhead  | Native CSS    | Build-time      | Runtime JS cost   | Purged, tiny    |
| **Bundle Size** | Zero extra   | Small         | Small           | +12 KB (lib)      | Very small prod |
| **Pseudo :hover** | ❌         | ✅            | ✅              | ✅                | ✅              |
| **Media queries** | ❌         | ✅            | ✅              | ✅                | ✅ (prefixes)   |
| **Theming**     | Manual       | CSS variables | CSS variables   | ThemeProvider     | Config file     |
| **TypeScript**  | Built-in     | No type safety| Typed imports   | Generic props     | No type safety  |
| **SSR support** | ✅           | ✅            | ✅              | ⚠️ Extra setup   | ✅              |
| **Popularity**  | Universal    | Universal     | Very popular    | Popular (waning)  | Dominant (2024) |

### Bundle Size Impact

```
Bundle Size Comparison (approximate, gzipped)
──────────────────────────────────────────────
  Inline Styles       │ 0 KB   │ ████
  CSS Files           │ ~2 KB  │ ████████
  CSS Modules         │ ~2 KB  │ ████████
  styled-components   │ ~12 KB │ ████████████████████████████████████
  Tailwind (prod)     │ ~4 KB  │ ████████████████
──────────────────────────────────────────────
  (Tailwind purges unused utilities at build time)
```

### Decision Flowchart

```
  Start
    │
    ▼
  Is it a one-off dynamic style?
    │
   YES ──► Inline Style
    │
   NO
    │
    ▼
  Small project / few components?
    │
   YES ──► CSS Modules
    │
   NO
    │
    ▼
  Need rapid prototyping / utility-first?
    │
   YES ──► Tailwind CSS
    │
   NO
    │
    ▼
  Heavy dynamic theming / design system?
    │
   YES ──► styled-components (or Emotion)
    │
   NO
    │
    ▼
  Default ──► CSS Modules + CSS Variables
```

> [!tip] Recommendation for Backend Devs Moving to Frontend
> **Start with CSS Modules** — they feel closest to the "scoped beans" model you know from Spring. Each component owns its styles, no collisions, zero runtime cost. Think of `.module.css` as the `@Scope("prototype")` of styling.
>
> **Graduate to Tailwind** when you need to build UIs fast and want consistent spacing/colours without writing CSS files. It's the "convention-over-configuration" of styling — much like Spring Boot's opinionated defaults.

### The Shipping Analogy Summary

```
┌──────────────────────────────────────────────────────────────┐
│  Styling Approach     │  Shipping Equivalent                 │
├───────────────────────┼──────────────────────────────────────┤
│  Inline Styles        │  Hand-writing on the box             │
│  CSS Files            │  One label printer for the warehouse │
│  CSS Modules          │  Dedicated printer per zone          │
│  styled-components    │  Smart printer reads WMS data        │
│  Tailwind CSS         │  Modular sticker kit (peel & stick)  │
└───────────────────────┴──────────────────────────────────────┘
```

---

## Angular Comparison

For how Angular handles component styling (ViewEncapsulation, `:host`, `styleUrls`), see:
→ [[Components and Templates]]

Angular scopes styles by default using emulated shadow DOM — similar to CSS Modules in concept but handled by the framework automatically at runtime.

---

## Related Notes

- [[Components and Props]] — React component fundamentals
- [[Components and Templates]] — Angular's approach to component styling
- [[React vs Angular Comparison]] — Full framework comparison
