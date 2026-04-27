---
title: Angular Lifecycle Hooks
phase: fundamentals
topic: angular-lifecycle-hooks
tags:
  - angular
  - lifecycle-hooks
  - ngOnInit
  - ngOnDestroy
  - ngOnChanges
  - change-detection
  - frontend
  - spring-boot-parallel
difficulty: beginner-to-intermediate
prerequisites:
  - "[[Angular Fundamentals]]"
  - "[[Components and Templates]]"
  - "[[Services and Dependency Injection]]"
created: 2025-07-17
updated: 2025-07-17
---

# Angular Lifecycle Hooks

> *"Every shipment has a lifecycle — created, picked up, in transit, delivered. Every Angular component has one too."*

---

## Why This Matters

In Spring Boot, you use `@PostConstruct` to run setup code after a bean is initialized
and `@PreDestroy` to clean up before a bean is destroyed. You don't create beans manually — the container manages the lifecycle.

Angular is the **same idea**. You don't create components manually — Angular creates them,
updates them, and destroys them. Lifecycle hooks let you tap into those moments.

If you've ever wondered: *"When does my component actually have its data?"*
or *"How do I clean up subscriptions?"* — lifecycle hooks are the answer.

---

## Mental Model — The Shipment Lifecycle Analogy

```
A shipment's lifecycle:

  ORDER PLACED  →  PICKED UP  →  IN TRANSIT  →  OUT FOR DELIVERY  →  DELIVERED  →  ARCHIVED
       ↑              ↑              ↑                ↑                  ↑            ↑
    (created)    (initialized)  (updated)        (checked)          (completed)  (destroyed)


An Angular component's lifecycle:

  constructor()  →  ngOnInit()  →  ngOnChanges()  →  ngDoCheck()  →  ngOnDestroy()
       ↑               ↑              ↑                 ↑                ↑
    (created)    (initialized)   (inputs change)   (change detected) (removed)


Spring bean lifecycle:

  constructor  →  @PostConstruct  →  (bean in use)  →  @PreDestroy  →  (garbage collected)
       ↑               ↑                                    ↑
    (created)    (initialized)                          (destroyed)
```

---

## Phase 1: What are Lifecycle Hooks?

> [!info] Definition
> Lifecycle hooks are **methods** that Angular calls automatically at specific moments
> in a component's existence — from creation to destruction.

### The Complete Lifecycle — Execution Order

```
┌──────────────────────────────────────────────────────────────────┐
│                ANGULAR COMPONENT LIFECYCLE                        │
│                (in exact execution order)                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. constructor()                                                │
│     └─ Class instantiated. DI happens here.                      │
│        NO @Input values yet. Don't fetch data here.              │
│                                                                  │
│  2. ngOnChanges(changes: SimpleChanges)                          │
│     └─ Called BEFORE ngOnInit and every time @Input changes.     │
│        Receives old and new values.                              │
│                                                                  │
│  3. ngOnInit()                          ★ MOST USED              │
│     └─ Component initialized. @Input values ARE available.       │
│        Fetch data here. Like @PostConstruct in Spring.           │
│                                                                  │
│  4. ngDoCheck()                                                  │
│     └─ Custom change detection. Runs on EVERY change detection   │
│        cycle. Use sparingly — performance impact.                │
│                                                                  │
│  5. ngAfterContentInit()                                         │
│     └─ Projected content (<ng-content>) is initialized.          │
│        Called once after first ngDoCheck.                         │
│                                                                  │
│  6. ngAfterContentChecked()                                      │
│     └─ Projected content checked. Runs after every ngDoCheck.    │
│                                                                  │
│  7. ngAfterViewInit()                   ★ COMMONLY USED          │
│     └─ Component's view (template + children) is initialized.    │
│        Access @ViewChild here. Initialize DOM-dependent libs.    │
│                                                                  │
│  8. ngAfterViewChecked()                                         │
│     └─ View checked. Runs after every change detection.          │
│        Careful: don't modify state here → infinite loop.         │
│                                                                  │
│  ─── Component is alive, handling events, receiving changes ───  │
│                                                                  │
│  9. ngOnDestroy()                       ★ CRITICAL               │
│     └─ Component about to be removed from DOM.                   │
│        Unsubscribe. Clear timers. Release resources.             │
│        Like @PreDestroy in Spring.                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Spring Bean Lifecycle Comparison

| Angular Hook | Spring Equivalent | When Called |
|-------------|-------------------|------------|
| `constructor()` | Constructor | Object creation, DI injection |
| `ngOnChanges()` | *(no direct equivalent)* | When `@Input` properties change |
| `ngOnInit()` | `@PostConstruct` | After initialization, inputs ready |
| `ngDoCheck()` | *(no direct equivalent)* | Every change detection cycle |
| `ngAfterContentInit()` | *(no direct equivalent)* | After projected content init |
| `ngAfterViewInit()` | *(no direct equivalent)* | After view/template init |
| `ngOnDestroy()` | `@PreDestroy` | Before component removal |

> [!note] Why Angular Has More Hooks Than Spring
> Spring beans don't have templates, child components, or projected content.
> Angular components are **UI elements** — they need hooks for when the DOM is ready,
> when children are rendered, when projected content is available, etc.

### How to Use a Hook — Implement the Interface

```typescript
import { Component, OnInit, OnDestroy, OnChanges, SimpleChanges } from '@angular/core';

// Implement the interface(s) you need
@Component({
  selector: 'app-shipment-tracker',
  template: `<div>{{ shipment?.trackingNumber }}</div>`
})
export class ShipmentTrackerComponent implements OnInit, OnDestroy, OnChanges {

  @Input() shipmentId!: string;
  shipment?: Shipment;
  private pollingSubscription?: Subscription;

  constructor(private shipmentService: ShipmentService) {
    // ✅ DI happens here — only assign injected services
    // ❌ Don't fetch data, don't access @Input values
  }

  ngOnChanges(changes: SimpleChanges): void {
    // Called when @Input properties change
    console.log('Input changed:', changes);
  }

  ngOnInit(): void {
    // ✅ @Input values are available — fetch data here
    this.loadShipment();
    this.startPolling();
  }

  ngOnDestroy(): void {
    // ✅ Clean up — unsubscribe, clear timers
    this.pollingSubscription?.unsubscribe();
  }

  private loadShipment(): void {
    this.shipmentService.getById(this.shipmentId)
      .subscribe(data => this.shipment = data);
  }

  private startPolling(): void {
    this.pollingSubscription = interval(30000).pipe(
      switchMap(() => this.shipmentService.getById(this.shipmentId))
    ).subscribe(data => this.shipment = data);
  }
}
```

---

## Phase 2: ngOnInit — The Most Used Hook

> [!tip] Rule of Thumb
> **Constructor** = receive dependencies (DI only)
> **ngOnInit** = initialize the component (fetch data, set up subscriptions)
>
> Same as Spring: constructor receives beans, `@PostConstruct` does setup.

### Why NOT the Constructor?

```typescript
// ❌ WRONG — @Input is NOT available in constructor
@Component({ selector: 'app-shipment-detail', template: '...' })
export class ShipmentDetailComponent {
  @Input() shipmentId!: string;

  constructor(private shipmentService: ShipmentService) {
    // this.shipmentId is UNDEFINED here!
    console.log(this.shipmentId);  // → undefined
    this.shipmentService.getById(this.shipmentId);  // → ERROR: undefined id
  }
}

// ✅ CORRECT — @Input IS available in ngOnInit
@Component({ selector: 'app-shipment-detail', template: '...' })
export class ShipmentDetailComponent implements OnInit {
  @Input() shipmentId!: string;
  shipment?: Shipment;
  loading = true;
  error?: string;

  constructor(private shipmentService: ShipmentService) {
    // Only DI — nothing else
  }

  ngOnInit(): void {
    // shipmentId is now set by Angular from the parent component
    this.loading = true;
    this.shipmentService.getById(this.shipmentId).pipe(
      finalize(() => this.loading = false)
    ).subscribe({
      next: (data) => this.shipment = data,
      error: (err) => this.error = err.message
    });
  }
}
```

### Execution Timeline

```
Parent template:  <app-shipment-detail [shipmentId]="'SHP-001'" />

Timeline:
  1. Angular creates ShipmentDetailComponent
  2. constructor() runs           → shipmentId = undefined
  3. Angular sets @Input values   → shipmentId = 'SHP-001'
  4. ngOnChanges() runs           → changes = { shipmentId: { current: 'SHP-001' } }
  5. ngOnInit() runs              → shipmentId = 'SHP-001' ✅ SAFE to use
```

### Spring Boot Comparison

```java
// Spring Boot — same pattern
@Component
public class ShipmentProcessor {
    private final ShipmentRepository repo;
    private Cache<String, Shipment> cache;

    // Constructor — only receive dependencies
    public ShipmentProcessor(ShipmentRepository repo) {
        this.repo = repo;
        // DON'T initialize cache here — dependencies might not be fully wired
    }

    @PostConstruct  // ← Angular's ngOnInit equivalent
    public void init() {
        // Safe to use injected dependencies
        this.cache = buildCache(repo.findFrequentlyAccessed());
        log.info("ShipmentProcessor initialized with {} cached items", cache.size());
    }
}
```

```
Constructor vs ngOnInit / @PostConstruct:

┌─────────────────┬─────────────────────────────────────────────┐
│   Constructor    │  ngOnInit / @PostConstruct                  │
├─────────────────┼─────────────────────────────────────────────┤
│ DI injection     │ Initialization logic                        │
│ Assign services  │ Fetch data                                  │
│ Simple defaults  │ Set up subscriptions                        │
│ NO side effects  │ Access @Input values                        │
│ NO async work    │ Call services                               │
│ Runs BEFORE      │ Runs AFTER inputs are set                   │
│ inputs are set   │                                             │
└─────────────────┴─────────────────────────────────────────────┘
```

---

## Phase 3: ngOnChanges — Reacting to Input Changes

> [!info] When to Use
> When your component receives `@Input` properties that can change over time,
> and you need to **react** to those changes (re-fetch data, recalculate, etc.).

### How It Works

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-shipment-detail',
  template: `
    <div *ngIf="loading">Loading shipment {{ shipmentId }}...</div>
    <div *ngIf="shipment">
      <h2>{{ shipment.trackingNumber }}</h2>
      <p>Status: {{ shipment.status }}</p>
      <p>{{ shipment.origin }} → {{ shipment.destination }}</p>
    </div>
    <div *ngIf="error" class="error">{{ error }}</div>
  `
})
export class ShipmentDetailComponent implements OnChanges {
  @Input() shipmentId!: string;
  shipment?: Shipment;
  loading = false;
  error?: string;

  constructor(private shipmentService: ShipmentService) {}

  // Called EVERY TIME shipmentId changes (including the first time)
  ngOnChanges(changes: SimpleChanges): void {
    // changes is an object with one entry per changed @Input
    const idChange = changes['shipmentId'];

    if (idChange) {
      console.log('Previous ID:', idChange.previousValue);  // e.g., 'SHP-001'
      console.log('Current ID:', idChange.currentValue);     // e.g., 'SHP-002'
      console.log('First change?', idChange.firstChange);    // true on first set

      // Re-fetch data when the ID changes
      this.loadShipment(idChange.currentValue);
    }
  }

  private loadShipment(id: string): void {
    this.loading = true;
    this.error = undefined;

    this.shipmentService.getById(id).pipe(
      finalize(() => this.loading = false)
    ).subscribe({
      next: (data) => this.shipment = data,
      error: (err) => this.error = `Failed to load shipment ${id}: ${err.message}`
    });
  }
}
```

### The SimpleChanges Object

```typescript
// When parent changes: <app-detail [shipmentId]="selectedId" [mode]="viewMode" />

ngOnChanges(changes: SimpleChanges): void {
  // changes structure:
  // {
  //   shipmentId: {
  //     previousValue: 'SHP-001',
  //     currentValue: 'SHP-002',
  //     firstChange: false,
  //     isFirstChange(): boolean
  //   },
  //   mode: {
  //     previousValue: 'summary',
  //     currentValue: 'detailed',
  //     firstChange: false,
  //     isFirstChange(): boolean
  //   }
  // }

  if (changes['shipmentId'] && !changes['shipmentId'].firstChange) {
    // Only re-fetch if it's NOT the first change (ngOnInit handles the first one)
    this.loadShipment(changes['shipmentId'].currentValue);
  }
}
```

### ngOnChanges vs ngOnInit

```
Scenario: Parent has a list. Clicking a shipment shows details.

  Parent: <app-detail [shipmentId]="selectedId" />

  User clicks "SHP-001":
    → ngOnChanges({ shipmentId: { current: 'SHP-001', firstChange: true } })
    → ngOnInit()                    ← Runs ONCE after first ngOnChanges
    → Component shows SHP-001

  User clicks "SHP-002":
    → ngOnChanges({ shipmentId: { current: 'SHP-002', firstChange: false } })
    → ngOnInit() does NOT run again ← Only runs once!
    → Component shows SHP-002

  Takeaway:
    ngOnInit  = runs ONCE after component is initialized
    ngOnChanges = runs on EVERY @Input change (including the first)
```

---

## Phase 4: ngOnDestroy — Cleanup (CRITICAL)

> [!warning] Memory Leak Prevention
> If you subscribe to Observables, set up timers, or attach event listeners
> and **don't clean them up**, your app will leak memory.
> `ngOnDestroy` is where you clean up — like `@PreDestroy` in Spring.

### The Problem — Memory Leaks

```typescript
// ❌ MEMORY LEAK — subscription is never cleaned up
@Component({ selector: 'app-live-tracker', template: '...' })
export class LiveTrackerComponent implements OnInit {
  position?: GeoPosition;

  constructor(private trackingService: TrackingService) {}

  ngOnInit(): void {
    // This observable emits every 5 seconds... FOREVER
    // Even after the component is destroyed, the subscription lives on
    this.trackingService.getPositionStream('SHP-001')
      .subscribe(pos => this.position = pos);  // ← LEAK!
  }
}
```

### The Fix — Unsubscribe in ngOnDestroy

```typescript
// ✅ CORRECT — clean up subscriptions
@Component({ selector: 'app-live-tracker', template: '...' })
export class LiveTrackerComponent implements OnInit, OnDestroy {
  position?: GeoPosition;
  private positionSub?: Subscription;

  constructor(private trackingService: TrackingService) {}

  ngOnInit(): void {
    this.positionSub = this.trackingService.getPositionStream('SHP-001')
      .subscribe(pos => this.position = pos);
  }

  ngOnDestroy(): void {
    // Clean up — stop listening when component is removed
    this.positionSub?.unsubscribe();
  }
}
```

### Better Pattern — Subject + takeUntil

```typescript
// ✅ BETTER — one destroy$ subject handles ALL subscriptions
@Component({ selector: 'app-dashboard', template: '...' })
export class DashboardComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();  // Emits once on destroy

  shipments: Shipment[] = [];
  alerts: Alert[] = [];
  metrics?: DashboardMetrics;

  constructor(
    private shipmentService: ShipmentService,
    private alertService: AlertService,
    private metricsService: MetricsService
  ) {}

  ngOnInit(): void {
    // All subscriptions automatically unsubscribe when destroy$ emits
    this.shipmentService.getShipments().pipe(
      takeUntil(this.destroy$)               // ← Auto-unsub on destroy
    ).subscribe(data => this.shipments = data);

    this.alertService.getAlerts().pipe(
      takeUntil(this.destroy$)               // ← Auto-unsub on destroy
    ).subscribe(data => this.alerts = data);

    this.metricsService.getLiveMetrics().pipe(
      takeUntil(this.destroy$)               // ← Auto-unsub on destroy
    ).subscribe(data => this.metrics = data);
  }

  ngOnDestroy(): void {
    this.destroy$.next();    // Signal all subscriptions to stop
    this.destroy$.complete(); // Complete the subject
  }
}
```

### Best Pattern — takeUntilDestroyed (Angular 16+)

```typescript
// ✅ BEST — Angular 16+ built-in helper, no manual Subject needed
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ selector: 'app-dashboard', template: '...' })
export class DashboardComponent implements OnInit {
  private destroyRef = inject(DestroyRef);  // Angular manages this

  shipments: Shipment[] = [];

  constructor(private shipmentService: ShipmentService) {}

  ngOnInit(): void {
    this.shipmentService.getShipments().pipe(
      takeUntilDestroyed(this.destroyRef)    // ← Zero boilerplate cleanup!
    ).subscribe(data => this.shipments = data);
  }

  // No ngOnDestroy needed — Angular handles it!
}
```

### Spring Boot Comparison

```java
// Spring Boot — @PreDestroy for cleanup
@Service
public class ShipmentPollingService {
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    @PostConstruct
    public void startPolling() {
        scheduler.scheduleAtFixedRate(this::pollShipments, 0, 30, TimeUnit.SECONDS);
    }

    @PreDestroy  // ← Angular's ngOnDestroy equivalent
    public void stopPolling() {
        scheduler.shutdown();  // Clean up resources
    }
}
```

```
Cleanup comparison:

Angular ngOnDestroy:           Spring @PreDestroy:
├─ Unsubscribe Observables     ├─ Shutdown thread pools
├─ Clear setInterval/Timeout   ├─ Close connections
├─ Remove event listeners      ├─ Flush caches
├─ Disconnect WebSockets       ├─ Disconnect WebSockets
└─ Release resources           └─ Release resources
```

---

## Phase 5: ngAfterViewInit — DOM is Ready

> [!info] When to Use
> When you need to interact with the **rendered DOM** — access child components,
> initialize third-party libraries (charts, maps), or measure element sizes.

### Accessing Child Elements with @ViewChild

```typescript
@Component({
  selector: 'app-shipment-map',
  template: `
    <div #mapContainer class="map-container"></div>
    <app-shipment-list #shipmentList [shipments]="shipments" />
  `
})
export class ShipmentMapComponent implements AfterViewInit, OnDestroy {
  @ViewChild('mapContainer') mapContainer!: ElementRef<HTMLDivElement>;
  @ViewChild('shipmentList') shipmentList!: ShipmentListComponent;

  private map?: MapLibInstance;

  constructor(private shipmentService: ShipmentService) {}

  // ❌ In ngOnInit — @ViewChild references are NOT available yet
  // ✅ In ngAfterViewInit — @ViewChild references ARE available

  ngAfterViewInit(): void {
    // DOM element is ready — initialize the map library
    this.map = new MapLibInstance(this.mapContainer.nativeElement, {
      center: [0, 0],
      zoom: 2
    });

    // Access child component methods
    console.log('Shipment list has', this.shipmentList.shipments.length, 'items');
  }

  ngOnDestroy(): void {
    this.map?.destroy();  // Clean up the map
  }
}
```

### Timeline — When @ViewChild Becomes Available

```
Component lifecycle with @ViewChild:

  constructor()           → @ViewChild = undefined
       ↓
  ngOnChanges()           → @ViewChild = undefined
       ↓
  ngOnInit()              → @ViewChild = undefined  ← Still not ready!
       ↓
  ngDoCheck()             → @ViewChild = undefined
       ↓
  ngAfterContentInit()    → @ViewChild = undefined
       ↓
  ngAfterContentChecked() → @ViewChild = undefined
       ↓
  ┌─────────────────────────────────────────────────┐
  │ Angular renders the template (creates DOM)       │
  └─────────────────────────────────────────────────┘
       ↓
  ngAfterViewInit()       → @ViewChild = AVAILABLE ✅  ← First time safe
       ↓
  ngAfterViewChecked()    → @ViewChild = available
```

> [!warning] Don't Modify Data in AfterViewInit
> Modifying component state in `ngAfterViewInit` can trigger another change detection
> cycle, which calls `ngAfterViewChecked`, which can loop.
> If you must update state, wrap it in `setTimeout()` or use `ChangeDetectorRef`.

```typescript
// ❌ This can cause ExpressionChangedAfterItHasBeenCheckedError
ngAfterViewInit(): void {
  this.title = 'Updated in AfterViewInit';  // May cause error in dev mode
}

// ✅ Defer the change to the next cycle
ngAfterViewInit(): void {
  setTimeout(() => {
    this.title = 'Updated safely';
  });
}
```

---

## Phase 6: Other Hooks — When You'll Actually Use Them

### ngDoCheck — Custom Change Detection

```typescript
// Rarely needed — only when Angular's default detection misses something
// Example: Deep object comparison (Angular only checks reference changes)

@Component({ selector: 'app-cargo-manifest', template: '...' })
export class CargoManifestComponent implements DoCheck {
  @Input() manifest!: CargoItem[];
  private previousLength = 0;

  ngDoCheck(): void {
    // Angular won't detect mutations inside an array
    // We manually check if items were added/removed
    if (this.manifest.length !== this.previousLength) {
      console.log(`Manifest changed: ${this.previousLength} → ${this.manifest.length} items`);
      this.previousLength = this.manifest.length;
      this.recalculateTotals();
    }
  }

  private recalculateTotals(): void {
    // Recalculate total weight, volume, etc.
  }
}
```

> [!warning] Performance
> `ngDoCheck` runs on **every single change detection cycle** — every click,
> every keypress, every timer. Keep logic minimal or you'll kill performance.

### ngAfterContentInit — Projected Content Ready

```typescript
// When using <ng-content> (content projection — like slots in Vue)
// This hook fires when the projected content is initialized

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-content select="[card-header]"></ng-content>
      </div>
      <div class="card-body">
        <ng-content></ng-content>
      </div>
    </div>
  `
})
export class CardComponent implements AfterContentInit {
  @ContentChild('cardTitle') cardTitle?: ElementRef;

  ngAfterContentInit(): void {
    // Projected content is available here
    if (this.cardTitle) {
      console.log('Card title:', this.cardTitle.nativeElement.textContent);
    }
  }
}

// Usage:
// <app-card>
//   <h2 card-header #cardTitle>Shipment Details</h2>
//   <p>Content goes here</p>
// </app-card>
```

```
@ViewChild  → elements in THIS component's template         → ngAfterViewInit
@ContentChild → elements PROJECTED INTO this component       → ngAfterContentInit

Think of it like:
  @ViewChild    = your own warehouse equipment
  @ContentChild = cargo that arrives from outside
```

---

## Phase 7: Hooks Comparison Table — The Complete Reference

### All Hooks at a Glance

| Hook | When Called | Frequency | Common Use | Spring Parallel |
|------|-----------|-----------|------------|-----------------|
| `constructor()` | Class instantiation | Once | DI only | Constructor |
| `ngOnChanges()` | `@Input` changes | Many times | React to prop changes | — |
| `ngOnInit()` | After first change detection | Once | Fetch data, setup | `@PostConstruct` |
| `ngDoCheck()` | Every change detection | Many times | Custom deep checks | — |
| `ngAfterContentInit()` | After `<ng-content>` init | Once | Access projected content | — |
| `ngAfterContentChecked()` | After `<ng-content>` check | Many times | Verify projected content | — |
| `ngAfterViewInit()` | After view rendered | Once | DOM access, chart init | — |
| `ngAfterViewChecked()` | After view checked | Many times | React to view changes | — |
| `ngOnDestroy()` | Before component removal | Once | Cleanup subscriptions | `@PreDestroy` |

### React Equivalent Mapping

| Angular Hook | React Equivalent | Notes |
|-------------|-----------------|-------|
| `ngOnInit()` | `useEffect(() => {}, [])` | Empty deps = run once on mount |
| `ngOnChanges()` | `useEffect(() => {}, [prop])` | Specific deps = run on change |
| `ngOnDestroy()` | `useEffect` return cleanup | `return () => { cleanup }` |
| `ngAfterViewInit()` | `useEffect` + `useRef` | Access DOM after render |
| `ngDoCheck()` | `useMemo` / custom hook | Custom comparison logic |
| Component-level provider | `useContext` + Provider | Scoped state |

> See also: [[Hooks]] (React hooks) and [[State and Lifecycle]] (React class lifecycle)

### Frequency Guide

```
Hooks that run ONCE:
  ├── constructor()          → Component created
  ├── ngOnInit()             → Initialized
  ├── ngAfterContentInit()   → Projected content ready
  ├── ngAfterViewInit()      → View ready
  └── ngOnDestroy()          → About to be removed

Hooks that run MANY TIMES (every change detection cycle):
  ├── ngOnChanges()          → Only when @Input actually changes
  ├── ngDoCheck()            → EVERY cycle (expensive!)
  ├── ngAfterContentChecked()→ EVERY cycle
  └── ngAfterViewChecked()   → EVERY cycle

Hooks you'll use 90% of the time:
  ★ ngOnInit()      → Setup, fetch data
  ★ ngOnDestroy()   → Cleanup
  ★ ngOnChanges()   → React to input changes (sometimes)
  ★ ngAfterViewInit()→ DOM access (sometimes)
```

---

## Phase 8: Best Practices

### ✅ Do

| Practice | Why |
|----------|-----|
| Use `ngOnInit` over constructor for initialization | Inputs are available, clearer intent |
| Always implement `OnDestroy` if you subscribe | Prevents memory leaks — **non-negotiable** |
| Use `takeUntilDestroyed()` (Angular 16+) | Cleanest subscription cleanup pattern |
| Keep `ngDoCheck` logic minimal | Runs on EVERY change detection cycle |
| Implement the interface (`implements OnInit`) | Makes intent clear, gets type checking |
| Use `ngOnChanges` for input-driven re-fetching | Cleaner than watching for changes manually |

### ❌ Don't

| Anti-Pattern | Why It's Bad |
|-------------|-------------|
| Fetch data in constructor | `@Input` values not set yet |
| Skip `ngOnDestroy` with subscriptions | Memory leaks accumulate silently |
| Heavy logic in `AfterViewChecked` | Runs every cycle — performance killer |
| Modify state in `AfterViewInit` | Causes `ExpressionChangedAfterItHasBeenCheckedError` |
| Use `ngDoCheck` without careful thought | Most expensive hook — runs constantly |
| Ignore `firstChange` in `ngOnChanges` | May double-fetch on initialization |

### Decision Flowchart — Which Hook Do I Use?

```
  Need to initialize the component?
    ├─ YES → ngOnInit() ★
    └─ NO
         │
  Need to react when @Input changes?
    ├─ YES → ngOnChanges()
    └─ NO
         │
  Need to access DOM / @ViewChild?
    ├─ YES → ngAfterViewInit()
    └─ NO
         │
  Need to clean up subscriptions/timers?
    ├─ YES → ngOnDestroy() ★
    └─ NO
         │
  Need to access projected <ng-content>?
    ├─ YES → ngAfterContentInit()
    └─ NO
         │
  Need custom change detection?
    ├─ YES → ngDoCheck() (rare)
    └─ NO → You probably don't need a hook
```

### Real-World Component — All Hooks Together

```typescript
@Component({
  selector: 'app-shipment-tracker',
  template: `
    <div #mapContainer class="map"></div>
    <h2>{{ shipment?.trackingNumber }}</h2>
    <p>Status: {{ shipment?.status }}</p>
    <p>Last updated: {{ lastUpdated | date:'medium' }}</p>
  `
})
export class ShipmentTrackerComponent
  implements OnInit, OnChanges, AfterViewInit, OnDestroy {

  @Input() shipmentId!: string;
  @ViewChild('mapContainer') mapContainer!: ElementRef;

  shipment?: Shipment;
  lastUpdated?: Date;
  private destroy$ = new Subject<void>();
  private map?: MapInstance;

  // 1. Constructor — DI only
  constructor(
    private shipmentService: ShipmentService,
    private trackingService: TrackingService
  ) {}

  // 2. ngOnChanges — react to shipmentId changes
  ngOnChanges(changes: SimpleChanges): void {
    if (changes['shipmentId'] && !changes['shipmentId'].firstChange) {
      // Re-fetch when parent changes the ID (not on first load — ngOnInit handles that)
      this.loadShipment();
    }
  }

  // 3. ngOnInit — initial setup
  ngOnInit(): void {
    this.loadShipment();
    this.startLiveUpdates();
  }

  // 4. ngAfterViewInit — DOM ready, init map
  ngAfterViewInit(): void {
    this.map = new MapInstance(this.mapContainer.nativeElement);
  }

  // 5. ngOnDestroy — clean everything up
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.map?.destroy();
  }

  private loadShipment(): void {
    this.shipmentService.getById(this.shipmentId).pipe(
      takeUntil(this.destroy$)
    ).subscribe(data => {
      this.shipment = data;
      this.lastUpdated = new Date();
      this.map?.setPosition(data.currentLocation);
    });
  }

  private startLiveUpdates(): void {
    this.trackingService.getPositionStream(this.shipmentId).pipe(
      takeUntil(this.destroy$)
    ).subscribe(position => {
      this.map?.updateMarker(position);
      this.lastUpdated = new Date();
    });
  }
}
```

---

## Quick Reference — Lifecycle Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│             ANGULAR LIFECYCLE CHEAT SHEET                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SETUP (fetch data, subscriptions):                          │
│    ngOnInit()                                                │
│                                                              │
│  REACT TO INPUT CHANGES:                                     │
│    ngOnChanges(changes: SimpleChanges)                       │
│                                                              │
│  ACCESS DOM / @ViewChild:                                    │
│    ngAfterViewInit()                                         │
│                                                              │
│  CLEANUP (ALWAYS if you subscribe):                          │
│    ngOnDestroy()                                             │
│    └─ Use takeUntil(destroy$) or takeUntilDestroyed()        │
│                                                              │
│  SPRING PARALLELS:                                           │
│    ngOnInit()    = @PostConstruct                             │
│    ngOnDestroy() = @PreDestroy                                │
│    constructor   = constructor (DI only)                      │
│                                                              │
│  GOLDEN RULE:                                                │
│    If you subscribe → you MUST unsubscribe                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Related Notes

- [[Components and Templates]] — Components that use these hooks
- [[Angular Fundamentals]] — Core concepts and project structure
- [[Services and Dependency Injection]] — Services consumed in `ngOnInit`
- [[RxJS and Reactive Programming]] — Observables and subscription management
- [[Hooks]] — React hooks comparison
- [[State and Lifecycle]] — React class component lifecycle comparison

---

> [!quote] Final Thought
> *"In Spring, you trust the container to manage your beans' lifecycle. In Angular, you trust the framework to manage your components' lifecycle. Same trust. Same pattern. Just learn the hook names."*
