---
tags:
  - angular
  - directives
  - pipes
  - data-transformation
  - frontend
created: 2025-07-15
---

# Directives and Pipes

> *"Directives are classes that add additional behavior to elements in your Angular applications."* — Angular Documentation

> *Directives are Angular's version of AOP — they let you attach cross-cutting behavior to DOM elements without modifying the element itself. Pipes are Angular's version of Spring Converters — they transform data for display without changing the source.*

---

## Phase 1: What Are Directives?

### The Core Idea

A directive is an instruction that tells Angular to **transform the DOM** in some way. Every time you use `@Component`, you're already using a directive — components are actually a special type of directive with a template.

### The Three Types

```
Directives
├── Component Directives     ← @Component (directive with a template)
│     e.g., <app-shipment-card>
│
├── Structural Directives    ← Change the DOM layout (add/remove elements)
│     e.g., *ngIf, *ngFor, @if, @for
│
└── Attribute Directives     ← Change the appearance or behavior of an element
      e.g., [ngClass], [ngStyle], custom directives
```

### 🔁 The Spring Boot Analogy

| Angular Concept | Spring Boot Equivalent | What It Does |
|---|---|---|
| **Component Directive** | `@Controller` | A full handler with a view |
| **Structural Directive** | (no direct equiv) | Adds/removes DOM elements |
| **Attribute Directive** | `@Aspect` (AOP) | Adds behavior to existing elements |
| **Pipe** | `Converter<S,T>` / `Formatter<T>` | Transforms data for display |

> [!tip] AOP Analogy
> Think of attribute directives like **Spring AOP aspects**. An `@Around` advice doesn't change the method itself — it wraps behavior around it. Similarly, an Angular directive doesn't change the element — it adds behavior (highlighting, validation, auto-focus) around it.

---

## Phase 2: Structural Directives (Legacy Syntax)

> [!warning] Legacy but Still Important
> Angular 17+ introduced `@if`, `@for`, `@switch` (see [[Components and Templates]]). The `*ngIf`, `*ngFor`, `*ngSwitch` syntax still works and you'll encounter it in older codebases. Know both.

### `*ngIf` — Conditional Rendering

```typescript
// shipment-detail.component.ts
export class ShipmentDetailComponent {
  shipment: Shipment | null = null;
  isLoading = true;
  error: string | null = null;
}
```

```html
<!-- Simple condition -->
<div *ngIf="shipment">
  <h3>{{ shipment.trackingId }}</h3>
</div>

<!-- With else template -->
<div *ngIf="shipment; else noShipment">
  <h3>{{ shipment.trackingId }}</h3>
  <p>{{ shipment.origin }} → {{ shipment.destination }}</p>
</div>
<ng-template #noShipment>
  <p>No shipment found. Try a different tracking ID.</p>
</ng-template>

<!-- With loading + error + data states -->
<div *ngIf="isLoading; else loaded">
  <p>Loading shipment data...</p>
</div>
<ng-template #loaded>
  <div *ngIf="error; else showData">
    <p class="error">Error: {{ error }}</p>
  </div>
  <ng-template #showData>
    <div *ngIf="shipment">
      <h3>{{ shipment.trackingId }}</h3>
    </div>
  </ng-template>
</ng-template>
```

> [!warning] Why the New Syntax is Better
> Notice how the old syntax uses nested `<ng-template>` blocks? That's exactly why Angular 17+ introduced `@if` / `@else` — the new syntax is far more readable. See [[Components and Templates#Phase 4 Control Flow New Syntax — Angular 17]].

### `*ngFor` — List Rendering

```html
<!-- Basic loop -->
<div *ngFor="let shipment of shipments">
  <p>{{ shipment.trackingId }} — {{ shipment.status }}</p>
</div>

<!-- With index and other context variables -->
<table>
  <tr *ngFor="let shipment of shipments; let i = index;
              let isFirst = first; let isLast = last;
              let isEven = even; let isOdd = odd;
              trackBy: trackByFn">
    <td>{{ i + 1 }}</td>
    <td>{{ shipment.trackingId }}</td>
    <td>{{ shipment.origin }} → {{ shipment.destination }}</td>
    <td [class.highlight]="isFirst">{{ shipment.status }}</td>
  </tr>
</table>
```

```typescript
// trackBy function — critical for performance
// Like equals()/hashCode() in Java — helps Angular identify items
trackByFn(index: number, shipment: Shipment): string {
  return shipment.trackingId;
}
```

### `*ngSwitch` — Multi-Branch Conditional

```html
<div [ngSwitch]="shipment.transportMode">
  <div *ngSwitchCase="'ocean'">🚢 Ocean Freight</div>
  <div *ngSwitchCase="'air'">✈️ Air Freight</div>
  <div *ngSwitchCase="'rail'">🚂 Rail Freight</div>
  <div *ngSwitchCase="'road'">🚛 Road Freight</div>
  <div *ngSwitchDefault>❓ Unknown</div>
</div>
```

### Legacy vs Modern — Quick Comparison

| Old (Structural Directive) | New (Built-in Control Flow) |
|---|---|
| `*ngIf="x"` | `@if (x) { }` |
| `*ngIf="x; else tmpl"` + `<ng-template>` | `@if (x) { } @else { }` |
| `*ngFor="let item of items; trackBy: fn"` | `@for (item of items; track item.id) { }` |
| `[ngSwitch]` + `*ngSwitchCase` | `@switch` + `@case` |

---

## Phase 3: Attribute Directives

Attribute directives change the **appearance or behavior** of an existing element. They don't add or remove elements — they modify them.

### `[ngClass]` — Conditional CSS Classes

```typescript
// shipment-row.component.ts
export class ShipmentRowComponent {
  @Input() status!: string;
  @Input() isUrgent = false;
  @Input() isSelected = false;
}
```

```html
<!-- String syntax — apply a single class -->
<div [ngClass]="'shipment-row'">Static class</div>

<!-- Object syntax — conditionally apply classes -->
<div [ngClass]="{
  'shipment-row': true,
  'status-delivered': status === 'delivered',
  'status-delayed': status === 'delayed',
  'status-in-transit': status === 'in-transit',
  'status-pending': status === 'pending',
  'urgent': isUrgent,
  'selected': isSelected
}">
  Shipment Row
</div>

<!-- Array syntax — apply multiple classes -->
<div [ngClass]="['shipment-row', 'card', status]">
  Shipment Row
</div>

<!-- Method syntax — compute classes dynamically -->
<div [ngClass]="getRowClasses()">
  Shipment Row
</div>
```

```typescript
getRowClasses(): Record<string, boolean> {
  return {
    'shipment-row': true,
    [`status-${this.status}`]: true,
    'urgent-highlight': this.isUrgent && this.status !== 'delivered',
    'selected': this.isSelected
  };
}
```

### `[ngStyle]` — Conditional Inline Styles

```html
<!-- Object syntax — set multiple styles -->
<div [ngStyle]="{
  'background-color': status === 'delayed' ? '#ffebee' : '#fff',
  'border-left': '4px solid ' + getStatusColor(status),
  'opacity': status === 'cancelled' ? '0.5' : '1',
  'font-weight': isUrgent ? 'bold' : 'normal'
}">
  {{ trackingId }} — {{ status }}
</div>

<!-- Individual style properties -->
<div [style.background-color]="getStatusColor(status)"
     [style.font-size.px]="isUrgent ? 16 : 14"
     [style.border-left]="'4px solid ' + getStatusColor(status)">
  Shipment Status
</div>
```

> [!tip] Prefer `[ngClass]` Over `[ngStyle]`
> Just like in web development generally — use CSS classes instead of inline styles whenever possible. `[ngClass]` is easier to maintain, test, and override. Use `[ngStyle]` only when you need truly dynamic values (e.g., calculated widths, positions).

### Other Built-in Attribute Directives

| Directive | Purpose | Example |
|---|---|---|
| `[ngClass]` | Conditional CSS classes | `[ngClass]="{'active': isActive}"` |
| `[ngStyle]` | Conditional inline styles | `[ngStyle]="{'color': textColor}"` |
| `[(ngModel)]` | Two-way form binding | `[(ngModel)]="searchQuery"` |
| `[ngTemplateOutlet]` | Render a template dynamically | Advanced pattern |

---

## Phase 4: Custom Directives

### Creating an Attribute Directive

Custom directives let you encapsulate reusable DOM behavior. This is where the AOP analogy to Spring really shines — you're adding cross-cutting behavior to elements.

**Example: Highlight directive for shipment rows on hover**

```typescript
// src/app/directives/highlight.directive.ts
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',  // Attribute selector — used as [appHighlight]
  standalone: true
})
export class HighlightDirective {
  // The highlight color can be customized via input
  // Default: light blue (good for shipment table rows)
  @Input() appHighlight = '#e3f2fd';

  // Optional: set the text color on highlight
  @Input() highlightTextColor = '';

  // Store the original background to restore on mouse leave
  private originalBg = '';
  private originalColor = '';

  // ElementRef gives direct access to the host DOM element
  // Like getting a reference to the HTML element in vanilla JS
  constructor(private el: ElementRef<HTMLElement>) {}

  // @HostListener listens to events on the HOST element
  // Like @EventListener in Spring — but for DOM events
  @HostListener('mouseenter')
  onMouseEnter(): void {
    this.originalBg = this.el.nativeElement.style.backgroundColor;
    this.originalColor = this.el.nativeElement.style.color;
    this.el.nativeElement.style.backgroundColor = this.appHighlight;
    if (this.highlightTextColor) {
      this.el.nativeElement.style.color = this.highlightTextColor;
    }
    this.el.nativeElement.style.cursor = 'pointer';
    this.el.nativeElement.style.transition = 'background-color 0.2s ease';
  }

  @HostListener('mouseleave')
  onMouseLeave(): void {
    this.el.nativeElement.style.backgroundColor = this.originalBg;
    this.el.nativeElement.style.color = this.originalColor;
  }
}
```

**Using the directive:**

```html
<!-- Default highlight (light blue) -->
<tr appHighlight>
  <td>FRT-001</td>
  <td>Shanghai → Sydney</td>
  <td>In Transit</td>
</tr>

<!-- Custom highlight color -->
<tr [appHighlight]="'#fff3e0'">
  <td>FRT-002</td>
  <td>Rotterdam → Mumbai</td>
  <td>Pending</td>
</tr>

<!-- With custom text color too -->
<tr [appHighlight]="'#1565c0'" [highlightTextColor]="'white'">
  <td>FRT-003</td>
  <td>Los Angeles → Tokyo</td>
  <td>Delivered</td>
</tr>
```

### Example: Auto-Focus Directive

Automatically focuses an input element when the component loads — useful for search bars.

```typescript
// src/app/directives/auto-focus.directive.ts
import { Directive, ElementRef, AfterViewInit } from '@angular/core';

@Directive({
  selector: '[appAutoFocus]',
  standalone: true
})
export class AutoFocusDirective implements AfterViewInit {
  constructor(private el: ElementRef<HTMLElement>) {}

  // Focus the element after the view is initialized
  // Like @PostConstruct in Spring — runs after setup is complete
  ngAfterViewInit(): void {
    // Small timeout to ensure the element is fully rendered
    setTimeout(() => {
      this.el.nativeElement.focus();
    }, 0);
  }
}
```

```html
<!-- The search input auto-focuses when the page loads -->
<input type="text"
       appAutoFocus
       placeholder="Search shipments by tracking ID..." />
```

### Example: Shipment Status Color Directive

Dynamically sets the text color based on shipment status:

```typescript
// src/app/directives/status-color.directive.ts
import { Directive, ElementRef, Input, OnChanges, SimpleChanges } from '@angular/core';

@Directive({
  selector: '[appStatusColor]',
  standalone: true
})
export class StatusColorDirective implements OnChanges {
  @Input() appStatusColor = '';

  private colorMap: Record<string, string> = {
    'pending': '#ff9800',      // Orange
    'in-transit': '#2196f3',   // Blue
    'delivered': '#4caf50',    // Green
    'delayed': '#f44336',      // Red
    'cancelled': '#9e9e9e'     // Grey
  };

  constructor(private el: ElementRef<HTMLElement>) {}

  // OnChanges runs whenever an @Input changes
  // Like a Spring @EventListener reacting to state changes
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['appStatusColor']) {
      const color = this.colorMap[this.appStatusColor] || '#000';
      this.el.nativeElement.style.color = color;
      this.el.nativeElement.style.fontWeight = 'bold';
    }
  }
}
```

```html
<!-- Automatically colors the text based on status -->
<span [appStatusColor]="shipment.status">
  {{ shipment.status }}
</span>
```

### Directive Best Practices

| ✅ Do | ❌ Don't |
|---|---|
| Keep directives focused on ONE behavior | Cram multiple behaviors into one directive |
| Use `@Input()` for configuration | Hardcode values inside the directive |
| Use `@HostListener` for events | Manually add event listeners (memory leaks) |
| Clean up in `ngOnDestroy` if needed | Leave subscriptions or listeners hanging |
| Prefix selectors with `app` | Use generic names that conflict with HTML |

---

## Phase 5: What Are Pipes?

### The Core Idea

Pipes transform data for **display** in templates. They take an input value, transform it, and return the transformed value — without changing the original data.

```
Raw Data ──→ Pipe ──→ Formatted Display

"2024-12-20T10:30:00Z"  ──→ date pipe ──→ "Dec 20, 2024, 10:30 AM"
25000.5                   ──→ currency pipe ──→ "$25,000.50"
"DELIVERED"              ──→ titlecase pipe ──→ "Delivered"
```

### 🔁 The Spring Boot Analogy

| Angular Pipe | Spring Boot Equivalent | Purpose |
|---|---|---|
| `date` pipe | `@DateTimeFormat` / `DateTimeFormatter` | Format dates |
| `currency` pipe | `NumberFormat.getCurrencyInstance()` | Format currency |
| `number` pipe | `DecimalFormat` | Format numbers |
| Custom pipe | `Converter<S,T>` / `Formatter<T>` | Custom data transformation |

**Java equivalent of a pipe:**
```java
// Spring Converter — same concept as an Angular pipe
@Component
public class ShipmentStatusConverter implements Converter<String, String> {
    @Override
    public String convert(String status) {
        return switch (status) {
            case "pending" -> "📦 Pending";
            case "in-transit" -> "🚚 In Transit";
            case "delivered" -> "✅ Delivered";
            case "cancelled" -> "❌ Cancelled";
            default -> status;
        };
    }
}
```

### Pipe Syntax

```html
<!-- Basic: value | pipeName -->
{{ shipment.createdAt | date }}

<!-- With arguments: value | pipeName:arg1:arg2 -->
{{ shipment.createdAt | date:'medium' }}

<!-- Chained pipes: value | pipe1 | pipe2 -->
{{ shipment.status | lowercase | titlecase }}
```

---

## Phase 6: Built-in Pipes

Angular ships with a rich set of built-in pipes. Here's every one you'll commonly use, with logistics-domain examples.

### `DatePipe` — Format Dates

```typescript
// Component
export class ShipmentComponent {
  createdAt = new Date('2024-12-15T14:30:00Z');
  estimatedDelivery = new Date('2024-12-20T10:00:00Z');
  departureTime = new Date('2024-12-16T08:45:00Z');
}
```

```html
<!-- Date pipe formats -->
{{ createdAt | date:'short' }}         <!-- 12/15/24, 2:30 PM -->
{{ createdAt | date:'medium' }}        <!-- Dec 15, 2024, 2:30:00 PM -->
{{ createdAt | date:'long' }}          <!-- December 15, 2024 at 2:30:00 PM GMT+11 -->
{{ createdAt | date:'full' }}          <!-- Sunday, December 15, 2024 at ... -->

{{ createdAt | date:'shortDate' }}     <!-- 12/15/24 -->
{{ createdAt | date:'mediumDate' }}    <!-- Dec 15, 2024 -->
{{ createdAt | date:'longDate' }}      <!-- December 15, 2024 -->

{{ createdAt | date:'shortTime' }}     <!-- 2:30 PM -->
{{ createdAt | date:'mediumTime' }}    <!-- 2:30:00 PM -->

<!-- Custom format strings (like Java's DateTimeFormatter patterns) -->
{{ createdAt | date:'yyyy-MM-dd' }}           <!-- 2024-12-15 -->
{{ createdAt | date:'dd/MM/yyyy HH:mm' }}     <!-- 15/12/2024 14:30 -->
{{ createdAt | date:'EEEE, MMM d' }}          <!-- Sunday, Dec 15 -->
{{ departureTime | date:'HH:mm z' }}          <!-- 08:45 AEDT -->
```

### `CurrencyPipe` — Format Currency

```typescript
export class FreightComponent {
  oceanFreightCost = 2500.00;
  airFreightCost = 8750.50;
  customsDuty = 1234.567;
  localCharges = 450;
}
```

```html
<!-- Default: USD with symbol -->
{{ oceanFreightCost | currency }}               <!-- $2,500.00 -->

<!-- Specific currency -->
{{ oceanFreightCost | currency:'USD' }}          <!-- $2,500.00 -->
{{ oceanFreightCost | currency:'EUR' }}          <!-- €2,500.00 -->
{{ oceanFreightCost | currency:'GBP' }}          <!-- £2,500.00 -->
{{ oceanFreightCost | currency:'AUD' }}          <!-- A$2,500.00 -->
{{ oceanFreightCost | currency:'JPY' }}          <!-- ¥2,500 -->

<!-- Currency code instead of symbol -->
{{ airFreightCost | currency:'USD':'code' }}     <!-- USD8,750.50 -->

<!-- Custom decimal places: minIntegerDigits.minFractionDigits-maxFractionDigits -->
{{ customsDuty | currency:'USD':'symbol':'1.2-2' }}  <!-- $1,234.57 -->
{{ localCharges | currency:'USD':'symbol':'1.0-0' }} <!-- $450 -->
```

### `DecimalPipe` (number) — Format Numbers

```typescript
export class MetricsComponent {
  totalShipments = 1247892;
  onTimeRate = 0.9453;
  avgTransitDays = 14.6789;
  containerWeight = 25000;
}
```

```html
<!-- Format: minIntegerDigits.minFractionDigits-maxFractionDigits -->
{{ totalShipments | number }}                  <!-- 1,247,892 -->
{{ totalShipments | number:'1.0-0' }}          <!-- 1,247,892 -->
{{ avgTransitDays | number:'1.1-1' }}          <!-- 14.7 -->
{{ avgTransitDays | number:'1.2-2' }}          <!-- 14.68 -->
{{ containerWeight | number:'1.0-0' }}         <!-- 25,000 -->
```

### `PercentPipe` — Format Percentages

```html
{{ onTimeRate | percent }}                <!-- 95% -->
{{ onTimeRate | percent:'1.1-1' }}        <!-- 94.5% -->
{{ onTimeRate | percent:'1.2-2' }}        <!-- 94.53% -->
```

### `UpperCasePipe` / `LowerCasePipe` / `TitleCasePipe`

```html
{{ 'maersk line' | uppercase }}       <!-- MAERSK LINE -->
{{ 'SHANGHAI' | lowercase }}          <!-- shanghai -->
{{ 'ocean freight' | titlecase }}     <!-- Ocean Freight -->
{{ shipment.status | titlecase }}     <!-- In-Transit → In-transit -->
```

### `JsonPipe` — Debug Helper

```html
<!-- Dumps the entire object as formatted JSON — GREAT for debugging -->
<pre>{{ shipment | json }}</pre>
<!-- Output:
{
  "trackingId": "FRT-001",
  "origin": "Shanghai, CN",
  "destination": "Sydney, AU",
  "status": "in-transit"
}
-->

<!-- Super useful during development to see what data looks like -->
<pre>{{ shipments | json }}</pre>
```

> [!tip] Debugging Trick
> `{{ data | json }}` inside a `<pre>` tag is Angular's `console.log()` for the template. Use it whenever you're not sure what shape your data has. It's like adding `@ToString` to a Java class and printing the object — instant visibility.

### `AsyncPipe` — Observable Auto-Subscribe

The `async` pipe automatically subscribes to an Observable or Promise and returns the latest value. It also **auto-unsubscribes** when the component is destroyed — preventing memory leaks.

```typescript
import { Component } from '@angular/core';
import { Observable } from 'rxjs';
import { AsyncPipe } from '@angular/common';
import { ShipmentService } from '../services/shipment.service';

@Component({
  selector: 'app-shipment-list',
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <!-- async pipe subscribes and unsubscribes automatically -->
    @if (shipments$ | async; as shipments) {
      @for (shipment of shipments; track shipment.id) {
        <div>{{ shipment.trackingId }} — {{ shipment.status }}</div>
      }
    } @else {
      <p>Loading...</p>
    }

    <!-- Also works for single values -->
    <p>Total count: {{ shipmentCount$ | async }}</p>
  `
})
export class ShipmentListComponent {
  // The $ suffix is a convention for Observable variables
  // Like naming a Future<T> with a "future" suffix in Java
  shipments$: Observable<Shipment[]>;
  shipmentCount$: Observable<number>;

  constructor(private shipmentService: ShipmentService) {
    this.shipments$ = this.shipmentService.getAll();
    this.shipmentCount$ = this.shipmentService.getCount();
  }
}
```

> [!warning] Always Use `async` Pipe for Observables in Templates
> Don't manually subscribe in `ngOnInit` and store the result — use the `async` pipe instead. It prevents memory leaks by auto-unsubscribing. Manual subscribes in component classes are the #1 source of memory leaks in Angular apps.

### `SlicePipe` — Array/String Slicing

```html
<!-- Show first 5 shipments only -->
@for (shipment of (shipments | slice:0:5); track shipment.id) {
  <div>{{ shipment.trackingId }}</div>
}

<!-- Truncate long strings -->
<p>{{ longDescription | slice:0:100 }}...</p>
```

### `KeyValuePipe` — Iterate Over Objects

```typescript
export class ShipmentSummaryComponent {
  statusCounts: Record<string, number> = {
    'pending': 45,
    'in-transit': 128,
    'delivered': 892,
    'delayed': 23,
    'cancelled': 7
  };
}
```

```html
<!-- Iterate over key-value pairs of an object -->
@for (entry of statusCounts | keyvalue; track entry.key) {
  <div>
    <strong>{{ entry.key | titlecase }}:</strong> {{ entry.value }} shipments
  </div>
}
```

---

## Phase 7: Custom Pipes

### Creating a Shipment Status Pipe

```typescript
// src/app/pipes/shipment-status.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'shipmentStatus',   // Used in template: {{ value | shipmentStatus }}
  standalone: true           // Self-contained, no module needed
})
export class ShipmentStatusPipe implements PipeTransform {
  // transform() is the core method — like convert() in Spring's Converter<S,T>
  // value = input, ...args = optional parameters
  transform(status: string, showEmoji: boolean = true): string {
    const statusMap: Record<string, { emoji: string; label: string }> = {
      'pending':    { emoji: '📦', label: 'Pending Pickup' },
      'in-transit': { emoji: '🚚', label: 'In Transit' },
      'delivered':  { emoji: '✅', label: 'Delivered' },
      'delayed':    { emoji: '⚠️', label: 'Delayed' },
      'cancelled':  { emoji: '❌', label: 'Cancelled' },
      'customs':    { emoji: '🛃', label: 'Held at Customs' }
    };

    const info = statusMap[status];
    if (!info) return status;

    return showEmoji ? `${info.emoji} ${info.label}` : info.label;
  }
}
```

```html
<!-- Usage in templates -->
{{ shipment.status | shipmentStatus }}          <!-- 🚚 In Transit -->
{{ shipment.status | shipmentStatus:false }}    <!-- In Transit (no emoji) -->

<!-- In a table -->
@for (shipment of shipments; track shipment.id) {
  <tr>
    <td>{{ shipment.trackingId }}</td>
    <td>{{ shipment.status | shipmentStatus }}</td>
    <td>{{ shipment.origin }} → {{ shipment.destination }}</td>
  </tr>
}
```

### Creating a Freight Weight Pipe

```typescript
// src/app/pipes/weight.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'weight',
  standalone: true
})
export class WeightPipe implements PipeTransform {
  // Converts weight between units: kg, lbs, tonnes
  transform(valueInKg: number, targetUnit: 'kg' | 'lbs' | 'tonnes' = 'kg'): string {
    if (valueInKg == null) return '—';

    switch (targetUnit) {
      case 'lbs':
        return `${(valueInKg * 2.20462).toFixed(1)} lbs`;
      case 'tonnes':
        return `${(valueInKg / 1000).toFixed(2)} t`;
      case 'kg':
      default:
        return `${valueInKg.toLocaleString()} kg`;
    }
  }
}
```

```html
<!-- Usage -->
{{ 25000 | weight }}             <!-- 25,000 kg -->
{{ 25000 | weight:'lbs' }}       <!-- 55,115.5 lbs -->
{{ 25000 | weight:'tonnes' }}    <!-- 25.00 t -->
```

### Creating a Time Ago Pipe

```typescript
// src/app/pipes/time-ago.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'timeAgo',
  standalone: true
})
export class TimeAgoPipe implements PipeTransform {
  transform(date: Date | string): string {
    const now = new Date();
    const past = new Date(date);
    const diffMs = now.getTime() - past.getTime();
    const diffSeconds = Math.floor(diffMs / 1000);
    const diffMinutes = Math.floor(diffSeconds / 60);
    const diffHours = Math.floor(diffMinutes / 60);
    const diffDays = Math.floor(diffHours / 24);

    if (diffSeconds < 60) return 'just now';
    if (diffMinutes < 60) return `${diffMinutes}m ago`;
    if (diffHours < 24) return `${diffHours}h ago`;
    if (diffDays < 7) return `${diffDays}d ago`;
    if (diffDays < 30) return `${Math.floor(diffDays / 7)}w ago`;
    return `${Math.floor(diffDays / 30)}mo ago`;
  }
}
```

```html
<!-- Shows relative time -->
<p>Last updated: {{ shipment.updatedAt | timeAgo }}</p>
<!-- "Last updated: 3h ago" -->
```

### Pure vs Impure Pipes

This is a critical performance concept.

| Aspect | Pure Pipe (default) | Impure Pipe |
|---|---|---|
| **Recalculation** | Only when input value **reference** changes | Every change detection cycle |
| **Performance** | ✅ Excellent (memoized) | ❌ Can be slow |
| **Declaration** | `@Pipe({ pure: true })` (default) | `@Pipe({ pure: false })` |
| **Use case** | Formatting, conversion | Filtering arrays, async data |

```typescript
// PURE pipe (default) — only recalculates when input reference changes
@Pipe({ name: 'shipmentStatus', standalone: true, pure: true })
export class ShipmentStatusPipe implements PipeTransform {
  transform(status: string): string { /* ... */ }
}

// IMPURE pipe — recalculates on EVERY change detection cycle
// Use VERY sparingly — can destroy performance
@Pipe({ name: 'filterByStatus', standalone: true, pure: false })
export class FilterByStatusPipe implements PipeTransform {
  transform(shipments: Shipment[], status: string): Shipment[] {
    return shipments.filter(s => s.status === status);
  }
}
```

> [!warning] Avoid Impure Pipes for Filtering
> Don't use impure pipes to filter arrays — it runs on EVERY change detection cycle (could be hundreds of times per second). Instead, compute the filtered list in the component class:
>
> ```typescript
> // ✅ Do this instead
> get filteredShipments(): Shipment[] {
>   return this.shipments.filter(s => s.status === this.selectedStatus);
> }
> ```

---

## Phase 8: Pipes vs React

Angular pipes and React's approach to data transformation are fundamentally different:

### Syntax Comparison

```html
<!-- Angular: Pipe syntax built into templates -->
<p>{{ shipment.createdAt | date:'medium' }}</p>
<p>{{ freightCost | currency:'USD' }}</p>
<p>{{ shipment.status | shipmentStatus }}</p>

<!-- Chaining is natural -->
<p>{{ shipment.status | lowercase | titlecase }}</p>
```

```jsx
{/* React: Just function calls — no special syntax */}
<p>{formatDate(shipment.createdAt, 'medium')}</p>
<p>{formatCurrency(freightCost, 'USD')}</p>
<p>{formatStatus(shipment.status)}</p>

{/* Chaining requires nesting */}
<p>{titleCase(shipment.status.toLowerCase())}</p>
```

### Feature Comparison

| Feature | Angular Pipes | React Functions |
|---|---|---|
| **Syntax** | `{{ value \| pipeName }}` | `{formatFn(value)}` |
| **Memoization** | Automatic (pure pipes) | Manual (`useMemo`, `React.memo`) |
| **Chaining** | `value \| pipe1 \| pipe2` | `pipe2(pipe1(value))` |
| **Template integration** | First-class syntax | Just function calls |
| **Async support** | `{{ obs$ \| async }}` | `useEffect` + `useState` |
| **Built-in library** | Rich (date, currency, etc.) | None (use libraries) |
| **Learning curve** | Must learn pipe syntax | Just JavaScript functions |
| **Reusability** | Declarative in any template | Import and call anywhere |

### The Angular Advantage

```html
<!-- Angular: async pipe handles subscribe/unsubscribe automatically -->
@if (shipments$ | async; as shipments) {
  <p>{{ shipments.length }} shipments loaded</p>
}
```

```jsx
// React: You manage the subscription lifecycle manually
const [shipments, setShipments] = useState([]);
useEffect(() => {
  const sub = shipmentService.getAll().subscribe(setShipments);
  return () => sub.unsubscribe();  // Don't forget this!
}, []);
```

> [!tip] Key Takeaway
> Angular pipes are **declarative, memoized, and composable** — they're a cleaner syntax for data transformation in templates. React uses plain functions, which are more flexible but require manual optimization. Both approaches work well — Angular's is just more structured.

---

## Quick Reference — Directives & Pipes Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│                DIRECTIVES & PIPES SUMMARY                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  STRUCTURAL DIRECTIVES (Legacy)                             │
│    *ngIf="condition"           Conditional render           │
│    *ngFor="let x of list"      Loop render                  │
│    [ngSwitch] / *ngSwitchCase  Multi-branch                 │
│                                                             │
│  ATTRIBUTE DIRECTIVES                                       │
│    [ngClass]="{ 'cls': bool }"  Conditional classes         │
│    [ngStyle]="{ 'prop': val }"  Conditional styles          │
│    [(ngModel)]="property"       Two-way binding             │
│                                                             │
│  CUSTOM DIRECTIVES                                          │
│    @Directive({ selector: '[appName]' })                    │
│    @HostListener('event') for DOM events                    │
│    @Input() for configuration                               │
│                                                             │
│  BUILT-IN PIPES                                             │
│    {{ val | date:'medium' }}         Format dates           │
│    {{ val | currency:'USD' }}        Format currency        │
│    {{ val | number:'1.2-2' }}        Format numbers         │
│    {{ val | percent }}               Format percentages     │
│    {{ val | uppercase/lowercase }}   Change case            │
│    {{ val | json }}                  Debug dump             │
│    {{ obs$ | async }}                Auto-subscribe         │
│    {{ arr | slice:0:5 }}             Array slicing          │
│    {{ obj | keyvalue }}              Object iteration       │
│                                                             │
│  CUSTOM PIPES                                               │
│    @Pipe({ name: 'myPipe', standalone: true })              │
│    implements PipeTransform                                  │
│    transform(value, ...args): result                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## What's Next?

- **[[Angular Lifecycle Hooks]]** — `ngOnInit`, `ngOnChanges`, `ngOnDestroy` (like `@PostConstruct` / `@PreDestroy`)
- **[[Angular Services and DI]]** — Dependency injection deep dive
- **[[Angular Routing]]** — Client-side navigation with guards
- **[[Angular Forms]]** — Template-driven and reactive forms with validation

---

## See Also

- [[Components and Templates]]
- [[Angular Fundamentals]]
- [[React vs Angular Comparison]]
- [[Angular Lifecycle Hooks]]
