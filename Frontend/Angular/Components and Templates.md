---
tags:
  - angular
  - components
  - templates
  - data-binding
  - frontend
created: 2025-07-15
---

# Components and Templates

> *"Components are the main building block for Angular applications. Each component consists of a TypeScript class, an HTML template, and styles."* — Angular Documentation

> *If a Spring `@Controller` is a back-end handler for a URL, an Angular `@Component` is a front-end handler for a piece of UI. Same idea, different side of the wire.*

---

## Phase 1: Angular Component Anatomy

### The Three Pillars of a Component

Every Angular component consists of three parts — just like a Spring MVC controller has a handler class, a view template, and sometimes associated styles:

```
Angular Component
├── shipment-card.component.ts       ← Class (logic + data)
├── shipment-card.component.html     ← Template (view / HTML)
└── shipment-card.component.css      ← Styles (scoped CSS)
```

```
Spring Boot Equivalent
├── ShipmentController.java          ← Handler class (logic + model)
├── shipment-card.html               ← Thymeleaf template (view)
└── styles.css                       ← CSS (not scoped)
```

### The @Component Decorator — Metadata That Tells Angular What This Class Is

```typescript
// src/app/shipment-card/shipment-card.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  // The CSS selector used to insert this component into HTML
  // Usage: <app-shipment-card /> in a parent template
  selector: 'app-shipment-card',

  // This component manages its own dependencies (no NgModule needed)
  standalone: true,

  // Other Angular modules/components/pipes this component needs
  imports: [CommonModule],

  // Path to the HTML template file
  templateUrl: './shipment-card.component.html',

  // Path to the CSS file(s) — scoped to this component only
  styleUrls: ['./shipment-card.component.css']
})
export class ShipmentCardComponent {
  // @Input() marks properties that a PARENT component can set
  // Like parameters in a Spring Controller method
  @Input() trackingId!: string;     // ! = definite assignment assertion
  @Input() status!: string;
  @Input() origin!: string;
  @Input() destination!: string;
  @Input() carrier!: string;
  @Input() estimatedDelivery!: Date;
}
```

### Decorator Properties Explained

| Property | Purpose | Required? |
|---|---|---|
| `selector` | CSS selector to use in HTML (`<app-shipment-card />`) | ✅ Yes |
| `standalone` | Self-contained component (no module) | ✅ Recommended |
| `imports` | Dependencies (other components, modules, pipes) | Only if standalone |
| `templateUrl` | Path to external HTML template | One of template/templateUrl |
| `template` | Inline HTML template | One of template/templateUrl |
| `styleUrls` | Path(s) to external CSS files | ❌ Optional |
| `styles` | Inline CSS styles | ❌ Optional |
| `changeDetection` | Performance optimization strategy | ❌ Optional |
| `encapsulation` | CSS encapsulation mode | ❌ Optional |

### Inline vs External Templates

```typescript
// External template (recommended for larger components)
@Component({
  selector: 'app-shipment-card',
  standalone: true,
  templateUrl: './shipment-card.component.html',  // Separate file
  styleUrls: ['./shipment-card.component.css']
})

// Inline template (good for small, simple components)
@Component({
  selector: 'app-status-badge',
  standalone: true,
  template: `
    <span [class]="'badge badge-' + status">
      {{ status | titlecase }}
    </span>
  `,
  styles: [`
    .badge { padding: 4px 8px; border-radius: 4px; font-size: 0.8rem; }
    .badge-delivered { background: #c8e6c9; color: #2e7d32; }
    .badge-pending { background: #fff3e0; color: #e65100; }
    .badge-in-transit { background: #e3f2fd; color: #1565c0; }
  `]
})
export class StatusBadgeComponent {
  @Input() status!: string;
}
```

> [!tip] Rule of Thumb
> Use **external templates** (`templateUrl`) when the HTML is more than ~10 lines. Use **inline templates** (`template`) for tiny components like badges, icons, or wrappers. Same logic as deciding whether a Spring controller method is simple enough for `@ResponseBody` inline vs a full Thymeleaf template.

---

## Phase 2: Data Binding — The Four Types

Data binding is how Angular keeps the component class and the template in sync. There are **four types**, and understanding them is crucial.

### The Four Binding Types at a Glance

```
Component Class ──── {{ value }} ────────→ Template     (Interpolation)
Component Class ──── [property]="val" ───→ Template     (Property Binding)
Component Class ←─── (event)="handler()" ─ Template     (Event Binding)
Component Class ←──→ [(ngModel)]="val" ←─→ Template     (Two-Way Binding)
```

### Comparison Table

| Type | Syntax | Direction | Spring Boot Analogy |
|---|---|---|---|
| **Interpolation** | `{{ expression }}` | Class → Template | `th:text="${shipment.status}"` |
| **Property Binding** | `[property]="expr"` | Class → Template | `th:value="${shipment.id}"` |
| **Event Binding** | `(event)="handler()"` | Template → Class | Form `action` / `@PostMapping` |
| **Two-Way Binding** | `[(ngModel)]="prop"` | Class ↔ Template | (no direct equiv in Thymeleaf) |

---

### Type 1: Interpolation — `{{ expression }}`

One-way binding from class to template. Angular evaluates the expression and inserts the result as text.

```typescript
// shipment-dashboard.component.ts
export class ShipmentDashboardComponent {
  companyName = 'WiseTech Logistics';
  totalShipments = 1247;
  activeCarriers = 15;
  onTimeRate = 94.5;
  lastUpdated = new Date();
}
```

```html
<!-- shipment-dashboard.component.html -->

<!-- Simple property binding -->
<h1>{{ companyName }}</h1>

<!-- Expressions are evaluated -->
<p>Active Shipments: {{ totalShipments }}</p>
<p>On-Time Rate: {{ onTimeRate }}%</p>

<!-- You can use expressions (but keep them simple!) -->
<p>Delayed Shipments: {{ totalShipments - (totalShipments * onTimeRate / 100) | number:'1.0-0' }}</p>

<!-- Method calls work too -->
<p>Last Updated: {{ lastUpdated.toLocaleString() }}</p>

<!-- String concatenation -->
<p>{{ activeCarriers }} carriers handling {{ totalShipments }} shipments</p>
```

> [!warning] Keep Interpolation Expressions Simple
> Don't put complex logic in `{{ }}`. If you need transformation, use a **pipe** or a **method** in the component class. Same rule as Thymeleaf — keep the template dumb, keep the logic in the class.
>
> - ❌ `{{ shipments.filter(s => s.status === 'delayed').length }}`
> - ✅ `{{ delayedCount }}` (computed in the class)

---

### Type 2: Property Binding — `[property]="expression"`

One-way binding from class to template. Sets an **element property** (not an HTML attribute) to a value from the component.

```typescript
// shipment-form.component.ts
export class ShipmentFormComponent {
  isSubmitting = false;
  shipmentImageUrl = '/assets/images/container.png';
  maxWeight = 25000; // kg
  tooltipText = 'Enter the gross weight in kilograms';
  isExpressShipment = true;
}
```

```html
<!-- shipment-form.component.html -->

<!-- Disable button while submitting (like disabling a form in Spring MVC) -->
<button [disabled]="isSubmitting">Submit Shipment</button>

<!-- Bind image source dynamically -->
<img [src]="shipmentImageUrl" [alt]="'Container image'" />

<!-- Bind input attributes -->
<input type="number" [max]="maxWeight" [title]="tooltipText" />

<!-- Bind CSS class conditionally -->
<div [class.express]="isExpressShipment">Shipment Details</div>

<!-- Bind inline style -->
<div [style.background-color]="isExpressShipment ? '#fff3e0' : '#fff'">
  Express shipments get a highlighted background
</div>
```

### Interpolation vs Property Binding

```html
<!-- These two are equivalent for string values: -->
<p title="{{ tooltipText }}">Hover me</p>
<p [title]="tooltipText">Hover me</p>

<!-- But for non-string values, you MUST use property binding: -->
<button [disabled]="isSubmitting">Submit</button>  <!-- ✅ Correct -->
<button disabled="{{ isSubmitting }}">Submit</button>  <!-- ❌ Wrong — always disabled -->
```

> [!tip] When to Use Which
> - **Interpolation** `{{ }}` — when you need to insert text content
> - **Property binding** `[]` — when you need to set an element property (especially booleans, objects, arrays)

---

### Type 3: Event Binding — `(event)="handler()"`

One-way binding from template to class. Listens for DOM events and calls a method in the component.

```typescript
// shipment-actions.component.ts
export class ShipmentActionsComponent {
  searchQuery = '';
  selectedShipment: Shipment | null = null;

  // Called when user clicks the track button
  onTrackShipment(trackingId: string): void {
    console.log('Tracking:', trackingId);
    // In real app: call ShipmentService.track(trackingId)
  }

  // Called on every keystroke in the search input
  onSearchInput(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.searchQuery = input.value;
    console.log('Searching:', this.searchQuery);
  }

  // Called when a shipment card is clicked
  onSelectShipment(shipment: Shipment): void {
    this.selectedShipment = shipment;
  }

  // Called when user presses Enter
  onKeyEnter(trackingId: string): void {
    this.onTrackShipment(trackingId);
  }

  // Called on mouse hover
  onHoverShipment(trackingId: string): void {
    console.log('Hovering over:', trackingId);
  }
}
```

```html
<!-- shipment-actions.component.html -->

<!-- Click event — the most common -->
<button (click)="onTrackShipment('FRT-001')">Track Shipment</button>

<!-- Input event — fires on every keystroke -->
<input type="text"
       placeholder="Search shipments..."
       (input)="onSearchInput($event)" />

<!-- $event is the native DOM event object -->
<!-- Like HttpServletRequest in a Spring controller -->

<!-- Keyboard event -->
<input type="text"
       placeholder="Enter tracking ID"
       (keyup.enter)="onKeyEnter(trackingInput.value)"
       #trackingInput />

<!-- Mouse events -->
<div (mouseenter)="onHoverShipment('FRT-001')"
     (mouseleave)="onHoverShipment('')">
  Hover over me
</div>

<!-- Multiple events on one element -->
<div class="shipment-card"
     (click)="onSelectShipment(shipment)"
     (dblclick)="onTrackShipment(shipment.trackingId)">
  {{ shipment.trackingId }}
</div>
```

### Common DOM Events

| Event | Fires When | Example |
|---|---|---|
| `(click)` | Element is clicked | Button clicks |
| `(dblclick)` | Element is double-clicked | Open detail view |
| `(input)` | Input value changes | Search filtering |
| `(change)` | Input loses focus after change | Form field update |
| `(keyup)` | Key is released | Generic key handling |
| `(keyup.enter)` | Enter key is released | Submit on Enter |
| `(keydown.escape)` | Escape key is pressed | Close modal |
| `(submit)` | Form is submitted | Form submission |
| `(mouseenter)` | Mouse enters element | Show tooltip |
| `(mouseleave)` | Mouse leaves element | Hide tooltip |
| `(focus)` | Element receives focus | Highlight field |
| `(blur)` | Element loses focus | Validate field |

---

### Type 4: Two-Way Binding — `[(ngModel)]="property"`

Combines property binding + event binding. The template and class stay in sync **in both directions**.

```typescript
// shipment-search.component.ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';  // Required for ngModel!

@Component({
  selector: 'app-shipment-search',
  standalone: true,
  imports: [FormsModule],  // ← Must import FormsModule for ngModel
  templateUrl: './shipment-search.component.html'
})
export class ShipmentSearchComponent {
  // These properties automatically sync with the form inputs
  trackingId = '';
  origin = '';
  destination = '';
  carrier = 'any';
  isExpress = false;

  onSearch(): void {
    console.log('Searching with:', {
      trackingId: this.trackingId,
      origin: this.origin,
      destination: this.destination,
      carrier: this.carrier,
      isExpress: this.isExpress
    });
  }

  onReset(): void {
    this.trackingId = '';
    this.origin = '';
    this.destination = '';
    this.carrier = 'any';
    this.isExpress = false;
  }
}
```

```html
<!-- shipment-search.component.html -->
<div class="search-form">
  <h3>Search Shipments</h3>

  <!-- Two-way binding: typing updates trackingId, changing trackingId updates input -->
  <label>Tracking ID:</label>
  <input type="text" [(ngModel)]="trackingId" placeholder="FRT-2024-..." />

  <label>Origin:</label>
  <input type="text" [(ngModel)]="origin" placeholder="Shanghai, CN" />

  <label>Destination:</label>
  <input type="text" [(ngModel)]="destination" placeholder="Sydney, AU" />

  <label>Carrier:</label>
  <select [(ngModel)]="carrier">
    <option value="any">Any Carrier</option>
    <option value="maersk">Maersk</option>
    <option value="msc">MSC</option>
    <option value="cosco">COSCO</option>
  </select>

  <label>
    <input type="checkbox" [(ngModel)]="isExpress" />
    Express Only
  </label>

  <!-- Live preview of current filter values -->
  <div class="preview">
    <p>Current filters: {{ trackingId || 'any' }} | {{ origin || 'any' }}
       → {{ destination || 'any' }} | {{ carrier }} | Express: {{ isExpress }}</p>
  </div>

  <button (click)="onSearch()">Search</button>
  <button (click)="onReset()">Reset</button>
</div>
```

> [!warning] FormsModule Required
> Two-way binding with `[(ngModel)]` requires importing `FormsModule`. Without it, you'll get a cryptic error. This is the most common Angular beginner mistake.
>
> ```typescript
> imports: [FormsModule]  // ← Don't forget this!
> ```

### How Two-Way Binding Works Under the Hood

`[(ngModel)]="trackingId"` is syntactic sugar for:

```html
<!-- This: -->
<input [(ngModel)]="trackingId" />

<!-- Is equivalent to: -->
<input [ngModel]="trackingId" (ngModelChange)="trackingId = $event" />

<!-- Which is: -->
<!-- Property binding IN: [ngModel]="trackingId" -->
<!-- Event binding OUT: (ngModelChange)="trackingId = $event" -->
```

The "banana in a box" syntax `[( )]` = `[]` (property) + `()` (event) combined.

---

## Phase 3: Template Syntax

### Template Reference Variables — `#variableName`

A template reference variable gives you a direct reference to a DOM element or component instance — like `@ViewChild` but inside the template.

```html
<!-- #trackingInput creates a reference to this input element -->
<input type="text" #trackingInput placeholder="Enter tracking ID" />

<!-- Use the reference to read the input's value -->
<button (click)="onTrack(trackingInput.value)">
  Track Shipment
</button>

<!-- You can reference any element -->
<div #shipmentCard class="card">
  <p>Card dimensions: {{ shipmentCard.offsetWidth }}x{{ shipmentCard.offsetHeight }}</p>
</div>
```

### Null-Safe Navigation — `?.`

Prevents errors when a property is `null` or `undefined`. Like Java's `Optional` or null-safe access.

```typescript
// Component
export class ShipmentDetailComponent {
  shipment: Shipment | null = null;  // Could be null while loading
}
```

```html
<!-- Without null-safe: THROWS ERROR if shipment is null -->
<!-- <p>{{ shipment.carrier.name }}</p> -->

<!-- With null-safe: renders nothing if shipment is null -->
<p>{{ shipment?.carrier?.name }}</p>
<p>{{ shipment?.origin }} → {{ shipment?.destination }}</p>
<p>ETA: {{ shipment?.estimatedDelivery?.toLocaleDateString() }}</p>
```

**Java Equivalent:**
```java
// Angular's ?. is like Java's Optional chaining:
Optional.ofNullable(shipment)
    .map(Shipment::getCarrier)
    .map(Carrier::getName)
    .orElse("");
```

### Non-Null Assertion — `!`

Tells TypeScript "I know this value is not null, trust me." Use sparingly.

```html
<!-- When you're CERTAIN the value exists (e.g., after an @if check) -->
@if (shipment) {
  <p>{{ shipment!.trackingId }}</p>
}
```

### Template Expressions — What You Can and Can't Do

```html
<!-- ✅ Allowed in template expressions -->
{{ 1 + 1 }}                          <!-- Arithmetic -->
{{ shipment.status === 'delivered' }} <!-- Comparison -->
{{ isExpress ? 'Express' : 'Standard' }}  <!-- Ternary -->
{{ shipments.length }}               <!-- Property access -->
{{ getStatusLabel(status) }}         <!-- Method calls -->
{{ value | currency:'USD' }}         <!-- Pipe transforms -->

<!-- ❌ NOT allowed in template expressions -->
{{ shipments = [] }}                 <!-- Assignments -->
{{ new Date() }}                     <!-- Object creation -->
{{ console.log('debug') }}           <!-- Side effects -->
{{ i++ }}                            <!-- Increment/decrement -->
{{ typeof value }}                   <!-- typeof operator -->
```

---

## Phase 4: Control Flow (New Syntax — Angular 17+)

Angular 17 introduced a new **built-in control flow** syntax that replaces the older structural directives (`*ngIf`, `*ngFor`, `*ngSwitch`). The new syntax is cleaner, more readable, and better for performance.

### `@if` / `@else if` / `@else` — Conditional Rendering

```typescript
// shipment-status.component.ts
export class ShipmentStatusComponent {
  @Input() status!: string;
  @Input() shipment!: Shipment | null;
}
```

```html
<!-- Simple condition -->
@if (shipment) {
  <div class="shipment-detail">
    <h3>{{ shipment.trackingId }}</h3>
    <p>{{ shipment.origin }} → {{ shipment.destination }}</p>
  </div>
} @else {
  <p class="no-data">No shipment selected. Choose one from the list.</p>
}

<!-- Multiple conditions — like Java if/else if/else -->
@if (status === 'delivered') {
  <span class="badge success">✅ Delivered</span>
} @else if (status === 'in-transit') {
  <span class="badge info">🚚 In Transit</span>
} @else if (status === 'pending') {
  <span class="badge warning">📦 Pending Pickup</span>
} @else if (status === 'delayed') {
  <span class="badge danger">⚠️ Delayed</span>
} @else if (status === 'cancelled') {
  <span class="badge danger">❌ Cancelled</span>
} @else {
  <span class="badge default">Unknown Status</span>
}

<!-- With a loading state -->
@if (isLoading) {
  <div class="spinner">Loading shipments...</div>
} @else if (shipments.length === 0) {
  <div class="empty-state">
    <p>No shipments found for this warehouse.</p>
    <button (click)="onCreateShipment()">Create First Shipment</button>
  </div>
} @else {
  <div class="shipment-list">
    <!-- render shipments -->
  </div>
}
```

### `@for` — List Rendering

```html
<!-- Basic @for loop — track is REQUIRED -->
@for (shipment of shipments; track shipment.id) {
  <app-shipment-card
    [trackingId]="shipment.trackingId"
    [status]="shipment.status"
    [origin]="shipment.origin"
    [destination]="shipment.destination" />
}

<!-- With @empty — shown when the array is empty -->
@for (shipment of shipments; track shipment.id) {
  <tr>
    <td>{{ shipment.trackingId }}</td>
    <td>{{ shipment.origin }}</td>
    <td>{{ shipment.destination }}</td>
    <td>{{ shipment.status }}</td>
    <td>{{ shipment.carrier }}</td>
    <td>{{ shipment.estimatedDelivery | date:'mediumDate' }}</td>
  </tr>
} @empty {
  <tr>
    <td colspan="6" class="empty-message">
      No shipments match your search criteria.
    </td>
  </tr>
}
```

### `@for` Context Variables

The `@for` block provides implicit context variables — like the loop variable in Java's enhanced for:

```html
@for (shipment of shipments; track shipment.id; let i = $index) {
  <div class="shipment-row"
       [class.odd]="$odd"
       [class.even]="$even"
       [class.first]="$first"
       [class.last]="$last">
    <span class="row-number">{{ i + 1 }}.</span>
    <span>{{ shipment.trackingId }}</span>
    <span>{{ shipment.status }}</span>
  </div>
}
```

| Variable | Type | Description |
|---|---|---|
| `$index` | `number` | Current index (0-based) |
| `$first` | `boolean` | Is this the first item? |
| `$last` | `boolean` | Is this the last item? |
| `$even` | `boolean` | Is the index even? |
| `$odd` | `boolean` | Is the index odd? |
| `$count` | `number` | Total number of items |

### `@switch` — Multi-Branch Conditional

```html
<!-- Like Java's switch statement -->
@switch (shipment.transportMode) {
  @case ('ocean') {
    <div class="transport-ocean">
      <span>🚢</span> Ocean Freight — {{ shipment.vessel }}
    </div>
  }
  @case ('air') {
    <div class="transport-air">
      <span>✈️</span> Air Freight — {{ shipment.airline }}
    </div>
  }
  @case ('rail') {
    <div class="transport-rail">
      <span>🚂</span> Rail Freight — {{ shipment.railOperator }}
    </div>
  }
  @case ('road') {
    <div class="transport-road">
      <span>🚛</span> Road Freight — {{ shipment.truckCompany }}
    </div>
  }
  @default {
    <div class="transport-unknown">
      <span>❓</span> Unknown Transport Mode
    </div>
  }
}
```

### Old Syntax vs New Syntax — Comparison

| Old (Angular <17) | New (Angular 17+) | Notes |
|---|---|---|
| `*ngIf="condition"` | `@if (condition) { }` | Cleaner, no microsyntax |
| `*ngIf="x; else tmpl"` | `@if (x) { } @else { }` | No need for `<ng-template>` |
| `*ngFor="let item of items"` | `@for (item of items; track item.id) { }` | `track` is required |
| `*ngFor="let item of items; trackBy: trackFn"` | `@for (item of items; track item.id) { }` | Simpler syntax |
| `[ngSwitch]` / `*ngSwitchCase` | `@switch` / `@case` / `@default` | Much cleaner |

```html
<!-- OLD: Required CommonModule import and ng-template hacks -->
<div *ngIf="shipment; else noShipment">
  <p>{{ shipment.trackingId }}</p>
</div>
<ng-template #noShipment>
  <p>No shipment found.</p>
</ng-template>

<!-- NEW: Clean, readable, no imports needed for control flow -->
@if (shipment) {
  <p>{{ shipment.trackingId }}</p>
} @else {
  <p>No shipment found.</p>
}
```

> [!tip] Use the New Syntax
> If you're starting with Angular 17+, **always use the new `@if`, `@for`, `@switch` syntax**. It's the future of Angular and produces smaller bundles. The old `*ngIf` / `*ngFor` still works but is considered legacy.

---

## Phase 5: Content Projection (`ng-content`)

Content projection is Angular's version of React's `children` prop — it lets you pass HTML content INTO a component from the outside. Think of it like a Spring Thymeleaf layout fragment.

### Single-Slot Projection

```typescript
// card.component.ts — a reusable card wrapper
@Component({
  selector: 'app-card',
  standalone: true,
  template: `
    <div class="card">
      <div class="card-body">
        <!-- ng-content = "put whatever the parent passes here" -->
        <ng-content />
      </div>
    </div>
  `,
  styles: [`
    .card {
      border: 1px solid #ddd;
      border-radius: 8px;
      box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    .card-body { padding: 1rem; }
  `]
})
export class CardComponent {}
```

```html
<!-- Parent template — passes content INTO the card -->
<app-card>
  <h3>Shipment FRT-001</h3>
  <p>Shanghai → Sydney</p>
  <p>Status: In Transit</p>
</app-card>

<!-- Rendered output: -->
<!-- <div class="card"><div class="card-body">
       <h3>Shipment FRT-001</h3>
       <p>Shanghai → Sydney</p>
       <p>Status: In Transit</p>
     </div></div> -->
```

### Multi-Slot Projection — Named Slots

```typescript
// shipment-panel.component.ts
@Component({
  selector: 'app-shipment-panel',
  standalone: true,
  template: `
    <div class="panel">
      <div class="panel-header">
        <ng-content select="[panel-header]" />
      </div>
      <div class="panel-body">
        <ng-content select="[panel-body]" />
      </div>
      <div class="panel-footer">
        <ng-content select="[panel-footer]" />
      </div>
    </div>
  `
})
export class ShipmentPanelComponent {}
```

```html
<!-- Parent: fills named slots with specific content -->
<app-shipment-panel>
  <div panel-header>
    <h2>🚚 Shipment FRT-2024-001</h2>
    <span class="badge">In Transit</span>
  </div>

  <div panel-body>
    <p><strong>Origin:</strong> Shanghai, CN</p>
    <p><strong>Destination:</strong> Sydney, AU</p>
    <p><strong>Carrier:</strong> Maersk</p>
    <p><strong>ETA:</strong> December 20, 2024</p>
  </div>

  <div panel-footer>
    <button>Track</button>
    <button>Update Status</button>
  </div>
</app-shipment-panel>
```

> [!tip] Content Projection vs React Children
> | Angular | React | Purpose |
> |---|---|---|
> | `<ng-content />` | `{children}` | Default slot |
> | `<ng-content select="[name]" />` | Named props | Named slots |
> | Content projection | Render props / slots | Flexible composition |

---

## Phase 6: Component Communication

### The Communication Patterns

```
Parent ──── @Input() ────→ Child        (Data down)
Parent ←─── @Output() ──── Child        (Events up)
Any    ←──→ Service ←───→ Any           (Shared state via DI)
```

This is the same pattern as React:
- **Data flows down** via `@Input()` (React: props)
- **Events flow up** via `@Output()` (React: callback props)

### Parent → Child: `@Input()`

```typescript
// child: shipment-card.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-shipment-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ trackingId }}</h3>
      <p>{{ origin }} → {{ destination }}</p>
      <p>Status: {{ status }}</p>
      <p>Carrier: {{ carrier }}</p>
    </div>
  `
})
export class ShipmentCardComponent {
  // @Input() = "this property comes from my parent"
  // Like a parameter annotated with @RequestParam in Spring
  @Input() trackingId = '';
  @Input() status = 'pending';
  @Input() origin = '';
  @Input() destination = '';
  @Input() carrier = '';

  // You can also use required inputs (Angular 16+)
  @Input({ required: true }) shipmentId!: string;

  // Transform inputs (Angular 16+)
  @Input({ transform: (value: string) => value.toUpperCase() })
  statusDisplay = '';
}
```

```html
<!-- parent template: passes data DOWN to child -->
@for (shipment of shipments; track shipment.id) {
  <app-shipment-card
    [trackingId]="shipment.trackingId"
    [status]="shipment.status"
    [origin]="shipment.origin"
    [destination]="shipment.destination"
    [carrier]="shipment.carrier"
    [shipmentId]="shipment.id" />
}
```

### Child → Parent: `@Output()` + `EventEmitter`

```typescript
// child: shipment-card.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-shipment-card',
  standalone: true,
  template: `
    <div class="card">
      <h3>{{ trackingId }}</h3>
      <p>{{ origin }} → {{ destination }}</p>

      <div class="actions">
        <button (click)="onTrack()">Track</button>
        <button (click)="onStatusChange('in-transit')">Mark In Transit</button>
        <button (click)="onStatusChange('delivered')">Mark Delivered</button>
        <button (click)="onDelete()">Delete</button>
      </div>
    </div>
  `
})
export class ShipmentCardComponent {
  @Input() trackingId = '';
  @Input() status = '';
  @Input() origin = '';
  @Input() destination = '';

  // @Output() = "I emit events that my parent can listen to"
  // EventEmitter<T> = what type of data is emitted
  @Output() tracked = new EventEmitter<string>();
  @Output() statusChanged = new EventEmitter<{ id: string; newStatus: string }>();
  @Output() deleted = new EventEmitter<string>();

  onTrack(): void {
    // Emit the tracking ID up to the parent
    this.tracked.emit(this.trackingId);
  }

  onStatusChange(newStatus: string): void {
    // Emit an object with ID and new status
    this.statusChanged.emit({
      id: this.trackingId,
      newStatus: newStatus
    });
  }

  onDelete(): void {
    this.deleted.emit(this.trackingId);
  }
}
```

```html
<!-- parent template: listens for events FROM child -->
@for (shipment of shipments; track shipment.id) {
  <app-shipment-card
    [trackingId]="shipment.trackingId"
    [status]="shipment.status"
    [origin]="shipment.origin"
    [destination]="shipment.destination"
    (tracked)="handleTrack($event)"
    (statusChanged)="handleStatusChange($event)"
    (deleted)="handleDelete($event)" />
}
```

```typescript
// parent: shipment-list.component.ts
export class ShipmentListComponent {
  shipments: Shipment[] = [/* ... */];

  // $event contains whatever the child emitted
  handleTrack(trackingId: string): void {
    console.log('Tracking shipment:', trackingId);
    // Navigate to tracking page or call API
  }

  handleStatusChange(event: { id: string; newStatus: string }): void {
    console.log('Status change:', event);
    // Call API to update status
  }

  handleDelete(trackingId: string): void {
    this.shipments = this.shipments.filter(s => s.trackingId !== trackingId);
  }
}
```

### Communication Patterns Comparison: Angular vs React

| Pattern | Angular | React |
|---|---|---|
| Parent → Child data | `@Input() name: string` | `props.name` |
| Child → Parent event | `@Output() event = new EventEmitter()` | `props.onEvent()` (callback) |
| Passing HTML content | `<ng-content />` | `{children}` |
| Named slots | `<ng-content select="[name]" />` | Named props |
| Shared state | Service with `BehaviorSubject` | Context API / Zustand / Redux |
| Direct child access | `@ViewChild(ChildComponent)` | `useRef()` + `forwardRef` |

---

## Phase 7: View Queries

### `@ViewChild` — Access a Single Child

`@ViewChild` lets you get a reference to a child component, directive, or DOM element from the parent class. Like using `@Autowired` to inject a specific bean.

```typescript
import { Component, ViewChild, ElementRef, AfterViewInit } from '@angular/core';
import { ShipmentCardComponent } from '../shipment-card/shipment-card.component';

@Component({
  selector: 'app-shipment-dashboard',
  standalone: true,
  imports: [ShipmentCardComponent],
  template: `
    <!-- Reference a DOM element -->
    <input #searchInput type="text" placeholder="Search..." />

    <!-- Reference a child component -->
    <app-shipment-card #featuredShipment
      [trackingId]="'FRT-001'"
      [status]="'in-transit'" />

    <button (click)="focusSearch()">Focus Search</button>
    <button (click)="highlightFeatured()">Highlight Featured</button>
  `
})
export class ShipmentDashboardComponent implements AfterViewInit {
  // Get reference to the input element
  @ViewChild('searchInput') searchInput!: ElementRef<HTMLInputElement>;

  // Get reference to the child component instance
  @ViewChild('featuredShipment') featuredCard!: ShipmentCardComponent;

  // ViewChild is available AFTER the view initializes
  ngAfterViewInit(): void {
    console.log('Search input:', this.searchInput.nativeElement);
    console.log('Featured card tracking ID:', this.featuredCard.trackingId);
  }

  focusSearch(): void {
    this.searchInput.nativeElement.focus();
  }

  highlightFeatured(): void {
    // Access child component's properties directly
    console.log('Featured shipment:', this.featuredCard.trackingId);
  }
}
```

### `@ViewChildren` — Access Multiple Children

```typescript
import { Component, ViewChildren, QueryList, AfterViewInit } from '@angular/core';
import { ShipmentCardComponent } from '../shipment-card/shipment-card.component';

@Component({
  selector: 'app-shipment-list',
  standalone: true,
  imports: [ShipmentCardComponent],
  template: `
    @for (shipment of shipments; track shipment.id) {
      <app-shipment-card
        [trackingId]="shipment.trackingId"
        [status]="shipment.status" />
    }
  `
})
export class ShipmentListComponent implements AfterViewInit {
  shipments = [/* ... */];

  // QueryList<T> — a live collection of child components
  @ViewChildren(ShipmentCardComponent)
  shipmentCards!: QueryList<ShipmentCardComponent>;

  ngAfterViewInit(): void {
    // Iterate over all child components
    this.shipmentCards.forEach(card => {
      console.log('Card:', card.trackingId, card.status);
    });

    // QueryList is observable — reacts to changes
    this.shipmentCards.changes.subscribe(cards => {
      console.log('Cards changed, new count:', cards.length);
    });
  }
}
```

### `@ContentChild` / `@ContentChildren` — Access Projected Content

These work like `@ViewChild` / `@ViewChildren` but for **projected content** (content passed via `<ng-content>`).

```typescript
// tab-group.component.ts
@Component({
  selector: 'app-tab-group',
  standalone: true,
  template: `
    <div class="tab-headers">
      @for (tab of tabs; track tab.label) {
        <button (click)="selectTab(tab)" [class.active]="tab === activeTab">
          {{ tab.label }}
        </button>
      }
    </div>
    <div class="tab-body">
      <ng-content />
    </div>
  `
})
export class TabGroupComponent implements AfterContentInit {
  // Access projected TabComponent children
  @ContentChildren(TabComponent)
  tabs!: QueryList<TabComponent>;

  activeTab!: TabComponent;

  ngAfterContentInit(): void {
    // Set the first tab as active
    this.activeTab = this.tabs.first;
    this.activeTab.isActive = true;
  }

  selectTab(tab: TabComponent): void {
    this.tabs.forEach(t => t.isActive = false);
    tab.isActive = true;
    this.activeTab = tab;
  }
}
```

### View Queries Summary

| Decorator | What It Accesses | When Available |
|---|---|---|
| `@ViewChild` | Single child in component's own template | `ngAfterViewInit` |
| `@ViewChildren` | Multiple children in component's own template | `ngAfterViewInit` |
| `@ContentChild` | Single child in projected content | `ngAfterContentInit` |
| `@ContentChildren` | Multiple children in projected content | `ngAfterContentInit` |

> [!warning] Timing Matters
> View queries are NOT available in the constructor or `ngOnInit`. They are only available after the view initializes:
> - `@ViewChild` → use in `ngAfterViewInit()`
> - `@ContentChild` → use in `ngAfterContentInit()`
>
> This is similar to how `@PostConstruct` in Spring runs AFTER dependency injection is complete — view queries are resolved AFTER the view is rendered.

---

## Quick Reference — Data Binding Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA BINDING SUMMARY                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  {{ expression }}          Interpolation (text content)     │
│  [property]="expression"   Property binding (DOM property)  │
│  (event)="handler()"       Event binding (DOM event)        │
│  [(ngModel)]="property"    Two-way binding (forms)          │
│                                                             │
│  @Input()                  Parent → Child data              │
│  @Output() + EventEmitter  Child → Parent events            │
│  <ng-content />            Content projection (slots)       │
│  @ViewChild / @ViewChildren  Direct child access            │
│                                                             │
│  #ref                      Template reference variable      │
│  ?.                        Null-safe navigation              │
│  @if / @for / @switch      Built-in control flow (v17+)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## What's Next?

- **[[Directives and Pipes]]** — Custom directives for DOM behavior + pipes for data transformation
- **[[Angular Lifecycle Hooks]]** — `ngOnInit`, `ngOnDestroy`, etc. (like `@PostConstruct` / `@PreDestroy`)
- **[[Angular Services and DI]]** — Dependency injection (feels exactly like Spring DI)
- **[[Angular Routing]]** — Client-side navigation (like `@RequestMapping`)
- **[[Angular Forms]]** — Template-driven and reactive forms

---

## See Also

- [[Angular Fundamentals]]
- [[Directives and Pipes]]
- [[Angular Lifecycle Hooks]]
- [[Components and Props]]
- [[React vs Angular Comparison]]
