---
title: RxJS and Reactive Programming
tags:
  - angular
  - rxjs
  - reactive-programming
  - observables
  - signals
  - frontend
created: 2025-07-14
status: in-progress
related:
  - "[[Angular Fundamentals]]"
  - "[[Services and Dependency Injection]]"
  - "[[Angular State Management]]"
  - "[[Reactive Programming]]"
---

# RxJS and Reactive Programming

> [!quote] "If Project Reactor is the engine of Spring WebFlux, RxJS is the engine of Angular."

You already know [[Reactive Programming]] from Spring WebFlux — `Mono<T>`, `Flux<T>`, backpressure, 
non-blocking streams. **RxJS is the JavaScript equivalent** that Angular uses everywhere.

This note bridges your backend reactive knowledge to the frontend world.

---

## Phase 1: What is RxJS?

> [!info] RxJS = Reactive Extensions for JavaScript
> A library for composing asynchronous and event-based programs using **Observable sequences**.
> Angular uses RxJS **heavily** — HttpClient, Router, Forms, EventEmitter all return Observables.

### 1.1 — Why RxJS in Angular?

In Spring Boot, you chose between:
- **MVC** (blocking, imperative) → `RestController` returns `ResponseEntity<T>`
- **WebFlux** (non-blocking, reactive) → `RestController` returns `Mono<T>` or `Flux<T>`

In Angular, there's no choice — **reactive is the default**:
```
Spring MVC (imperative)          →  Angular without RxJS (impossible in practice)
Spring WebFlux (reactive)        →  Angular with RxJS (the standard approach)
```

> [!tip] You already think in reactive streams. RxJS will feel familiar.

### 1.2 — The Rosetta Stone: Project Reactor ↔ RxJS

Since you already know Project Reactor from [[Reactive Programming]], here's the direct mapping:

| Project Reactor (Java) | RxJS (TypeScript) | Description |
|------------------------|-------------------|-------------|
| `Mono<T>` | `Observable<T>` (single emission) | Single async value |
| `Flux<T>` | `Observable<T>` (multiple emissions) | Stream of values |
| `Subscriber` | `Observer` | Consumes emitted values |
| `subscribe()` | `subscribe()` | Triggers the stream (cold start) |
| `.map()` | `.pipe(map())` | Transform each emitted value |
| `.flatMap()` | `.pipe(switchMap())` | Flatten nested async operations |
| `.filter()` | `.pipe(filter())` | Filter values in the stream |
| `.doOnNext()` | `.pipe(tap())` | Side effects without altering stream |
| `.doOnError()` | `.pipe(catchError())` | Handle errors in the stream |
| `.zipWith()` | `combineLatest()` / `forkJoin()` | Combine multiple streams |
| `.then()` | `.pipe(switchMap())` | Chain async operations |
| `Disposable` | `Subscription` | Cancel / unsubscribe from stream |
| `Sinks.Many` | `Subject` | Programmatic emission (push values) |
| `Schedulers` | `asyncScheduler` / `observeOn` | Control execution context |
| Backpressure | ❌ Not built-in | JS is single-threaded, no backpressure |

> [!warning] Key Difference: No Backpressure in RxJS
> Remember from your [[Reactive Programming]] notes — backpressure lets the subscriber tell 
> the publisher to slow down. JavaScript is **single-threaded** (event loop), so backpressure 
> doesn't apply the same way. Instead, you use operators like `debounceTime`, `throttleTime`, 
> `bufferTime` to manage overwhelming streams.

### 1.3 — Mental Model Shift

```
┌─────────────────────────────────────────────────────────────────┐
│                    REACTIVE PROGRAMMING                         │
│                                                                 │
│   BACKEND (Java)                  FRONTEND (TypeScript)         │
│   ─────────────                   ──────────────────────        │
│                                                                 │
│   Mono<Shipment>                  Observable<Shipment>          │
│       ↓                               ↓                        │
│   .map(s -> s.getStatus())        .pipe(map(s => s.status))    │
│       ↓                               ↓                        │
│   .flatMap(s -> callApi(s))       .pipe(switchMap(s => api(s)))│
│       ↓                               ↓                        │
│   .subscribe(result -> ...)       .subscribe(result => ...)    │
│                                                                 │
│   Same concept. Different syntax. Same reactive mindset.        │
└─────────────────────────────────────────────────────────────────┘
```

> [!quote] "If you can read a Reactor chain, you can read an RxJS chain. The operators are named differently, but the marble diagrams are identical."

---

## Phase 2: Observables — The Core Primitive

> [!info] Observable = A lazy push-based collection of multiple values over time.
> Think of it like `Flux<T>` that can emit 0, 1, or many values.

### 2.1 — Cold vs Hot Observables

This maps directly to what you know from Reactor:

| Type | Reactor Equivalent | RxJS Behavior | Analogy |
|------|-------------------|---------------|---------|
| **Cold** | Default `Mono`/`Flux` | Each subscriber gets its own execution | 📦 Each customer gets a **fresh shipment** |
| **Hot** | `Sinks.Many` / `.share()` | All subscribers share one execution | 📻 All warehouses hear the **same radio broadcast** |

```typescript
// ✅ COLD Observable — each subscriber triggers a new HTTP call
// Like: each Mono.subscribe() triggers a new request in WebFlux
const shipment$ = this.http.get<Shipment>('/api/shipments/123');

shipment$.subscribe(s => console.log('Sub 1:', s)); // HTTP call #1
shipment$.subscribe(s => console.log('Sub 2:', s)); // HTTP call #2 (separate!)

// ✅ HOT Observable — all subscribers share the same stream
// Like: Sinks.Many in Reactor
const clicks$ = fromEvent(document, 'click'); // one event source, many listeners
```

> [!warning] Common Gotcha
> Every `.subscribe()` on a cold Observable triggers a **new execution**.
> If you subscribe to an HTTP Observable 3 times, you get 3 HTTP requests!
> Use `shareReplay(1)` to make it hot (cache the result).

### 2.2 — Creating Observables

```typescript
import { Observable, of, from, interval, fromEvent, timer, EMPTY, throwError } from 'rxjs';

// ─── of() — emit fixed values (like Mono.just() / Flux.just()) ───
const single$ = of('SHP-001');                    // emits 'SHP-001', then completes
const multi$  = of('SHP-001', 'SHP-002', 'SHP-003'); // emits 3 values, then completes

// ─── from() — convert iterable/Promise to Observable ───
const fromArray$   = from(['SHP-001', 'SHP-002']);     // like Flux.fromIterable()
const fromPromise$ = from(fetch('/api/shipments'));     // like Mono.fromFuture()

// ─── interval() — emit incremental numbers on a timer ───
const polling$ = interval(5000); // emits 0, 1, 2, ... every 5 seconds
                                  // like Flux.interval(Duration.ofSeconds(5))

// ─── fromEvent() — DOM events as Observable (no backend equivalent) ───
const clicks$ = fromEvent(document.getElementById('btn')!, 'click');

// ─── timer() — emit after delay, optionally repeat ───
const delayed$ = timer(3000);        // emits 0 after 3 seconds
const repeat$  = timer(0, 5000);     // emits immediately, then every 5 seconds

// ─── EMPTY — completes immediately with no emissions ───
const nothing$ = EMPTY; // like Mono.empty()

// ─── throwError — errors immediately ───
const fail$ = throwError(() => new Error('Shipment not found'));
// like Mono.error(new RuntimeException("Shipment not found"))

// ─── new Observable() — full manual control ───
const custom$ = new Observable<Shipment>(subscriber => {
  // This is like implementing a custom Publisher in Reactor
  subscriber.next({ id: 'SHP-001', status: 'in-transit' } as Shipment);
  subscriber.next({ id: 'SHP-002', status: 'delivered' } as Shipment);
  subscriber.complete();
  
  // Teardown logic (like Disposable)
  return () => {
    console.log('Cleanup: connection closed');
  };
});
```

### 2.3 — Subscribing to Observables

```typescript
// The Observer interface (like Subscriber in Reactor)
// ┌────────────────────────────────────────────────────────────┐
// │  next(value)    → called for each emitted value           │
// │  error(err)     → called if stream errors (terminal)      │
// │  complete()     → called when stream finishes (terminal)  │
// └────────────────────────────────────────────────────────────┘

const shipments$ = this.shipmentService.getShipments();

// ✅ Full observer object
const subscription = shipments$.subscribe({
  next: (data) => {
    this.shipments = data;
    console.log('Received shipments:', data.length);
  },
  error: (err) => {
    console.error('Failed to load shipments:', err);
    this.errorMessage = 'Could not load shipments';
  },
  complete: () => {
    console.log('Stream completed');
  }
});

// ✅ Shorthand — just the next callback
shipments$.subscribe(data => this.shipments = data);

// ✅ CRITICAL — Unsubscribe to prevent memory leaks!
// In Reactor, the container manages lifecycle. In Angular, YOU must manage it.
ngOnDestroy(): void {
  subscription.unsubscribe(); // like calling disposable.dispose()
}
```

### 2.4 — The Lifecycle of an Observable

```
  Observable Created         Subscribed          Values Emitted       Completed/Error
       │                        │                     │                     │
       ▼                        ▼                     ▼                     ▼
  ┌──────────┐   .subscribe()  ┌──────────┐   next()  ┌──────────┐   complete()  ┌──────────┐
  │  COLD    │ ──────────────► │ RUNNING  │ ────────► │ EMITTING │ ────────────► │  DONE    │
  │ (inert)  │                 │          │           │          │    error()    │          │
  └──────────┘                 └──────────┘           └──────────┘ ────────────► └──────────┘
                                                                                      │
                                                                          .unsubscribe()
                                                                          (cleanup runs)
```

> [!danger] Memory Leak Warning ⚠️
> **In Spring WebFlux**, the framework manages subscriptions for you (request lifecycle).
> **In Angular**, if you `.subscribe()` manually, YOU must `.unsubscribe()` in `ngOnDestroy()`.
> Forgetting this is the #1 RxJS bug in Angular apps.

---

## Phase 3: The pipe() Operator — Composing Streams

> [!info] `.pipe()` is how you chain operators in RxJS.
> It's the equivalent of chaining `.map().filter().flatMap()` in Reactor.

### 3.1 — Why pipe()?

In Reactor:
```java
shipmentFlux
    .filter(s -> s.getStatus().equals("IN_TRANSIT"))
    .map(s -> s.getTrackingId())
    .doOnNext(id -> log.info("Processing: {}", id))
    .flatMap(id -> enrichmentService.enrich(id));
```

In RxJS:
```typescript
shipments$.pipe(
  filter(s => s.status === 'in-transit'),
  map(s => s.trackingId),
  tap(id => console.log('Processing:', id)),
  switchMap(id => this.enrichmentService.enrich(id))
);
```

> [!tip] Same chain style. Same mental model. Just wrap operators inside `.pipe()`.

### 3.2 — Full Practical Example

```typescript
// Logistics scenario: Load shipments, filter in-transit, handle errors
this.shipmentService.getShipments().pipe(
  // Step 1: Filter — only process if we have data
  filter(shipments => shipments.length > 0),
  
  // Step 2: Transform — extract in-transit shipments
  map(shipments => shipments.filter(s => s.status === 'in-transit')),
  
  // Step 3: Side effect — log for debugging (like .doOnNext())
  tap(filtered => console.log('In-transit count:', filtered.length)),
  
  // Step 4: Error handling — return fallback on failure
  catchError(err => {
    console.error('API Error:', err);
    this.notificationService.showError('Failed to load shipments');
    return of([]); // Return empty array as fallback (like .onErrorReturn())
  }),
  
  // Step 5: Cleanup — runs on complete OR error (like .doFinally())
  finalize(() => this.loading = false)
  
).subscribe(data => {
  this.inTransitShipments = data;
});
```

### 3.3 — Reactor ↔ RxJS Operator Chain Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│ REACTOR (Java)                  │ RxJS (TypeScript)              │
│─────────────────────────────────│────────────────────────────────│
│ flux                            │ observable$.pipe(              │
│   .filter(predicate)            │   filter(predicate),           │
│   .map(transformer)             │   map(transformer),            │
│   .doOnNext(sideEffect)         │   tap(sideEffect),             │
│   .flatMap(asyncOp)             │   switchMap(asyncOp),          │
│   .onErrorReturn(fallback)      │   catchError(() => of(fb)),    │
│   .doFinally(cleanup)           │   finalize(cleanup)            │
│   .subscribe(consumer);         │ ).subscribe(consumer);         │
└──────────────────────────────────────────────────────────────────┘
```

---

## Phase 4: Essential Operators

> [!quote] "Operators are the vocabulary of reactive programming. Master these and you can express any async flow."

### 4.1 — Transformation Operators

#### `map` — Transform Each Value
```typescript
// Like .map() in Reactor — transform each emitted value
// Logistics: Convert shipment to display model
this.shipmentService.getAll().pipe(
  map(shipments => shipments.map(s => ({
    id: s.id,
    label: `${s.trackingId} — ${s.origin} → ${s.destination}`,
    isLate: s.eta < new Date()
  })))
).subscribe(displayModels => this.shipmentCards = displayModels);
```

#### `switchMap` — Cancel Previous, Take Latest ⭐
```typescript
// Like .flatMap() BUT cancels previous inner subscription
// Logistics: User types in search — only care about latest search term
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  switchMap(term => this.shipmentService.search(term))
  // If user types "SHP" then "SHP-00", the "SHP" request is CANCELLED
).subscribe(results => this.searchResults = results);
```

#### `mergeMap` — Run All in Parallel
```typescript
// Like .flatMap() in Reactor — runs all inner subscriptions concurrently
// Logistics: Update status of multiple shipments at once
from(selectedShipmentIds).pipe(
  mergeMap(id => this.shipmentService.updateStatus(id, 'dispatched'))
  // All updates fire in parallel — order of completion not guaranteed
).subscribe(result => console.log('Updated:', result.id));
```

#### `concatMap` — Queue, Run One at a Time
```typescript
// Sequential execution — waits for each to complete before starting next
// Logistics: Process shipment milestones in order
from(milestones).pipe(
  concatMap(milestone => this.milestoneService.record(milestone))
  // Milestone 1 completes → Milestone 2 starts → Milestone 3 starts...
).subscribe();
```

#### `exhaustMap` — Ignore New While Running
```typescript
// Ignores new emissions while current inner observable is active
// Logistics: Prevent double-submit on "Confirm Booking" button
this.confirmClick$.pipe(
  exhaustMap(() => this.bookingService.confirm(this.booking))
  // Click 1 → API call starts. Click 2, 3 during call → IGNORED.
).subscribe(result => this.navigateToConfirmation(result));
```

### 4.2 — The switchMap / mergeMap / concatMap / exhaustMap Decision Table

> [!important] This is the MOST confusing part of RxJS. Master this table.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Operator     │ Previous?   │ Parallel? │ Use Case                       │
│──────────────│─────────────│───────────│────────────────────────────────│
│ switchMap    │ ❌ Cancels   │ No        │ Search, autocomplete, nav      │
│ mergeMap     │ ✅ Keeps     │ Yes       │ Batch ops, parallel calls      │
│ concatMap    │ ✅ Keeps     │ No (queue)│ Sequential writes, ordered ops │
│ exhaustMap   │ ✅ Keeps     │ No (drop) │ Prevent double-submit          │
└──────────────────────────────────────────────────────────────────────────┘

Warehouse Analogy:
  switchMap:   New truck arrives → STOP unloading current truck, switch to new one
  mergeMap:    New truck arrives → Open another dock, unload both simultaneously
  concatMap:   New truck arrives → Wait in line, unload one truck at a time
  exhaustMap:  New truck arrives → "We're busy, come back later" (truck leaves)
```

### 4.3 — Filtering Operators

```typescript
// ─── filter — same as Reactor .filter() ───
shipments$.pipe(
  filter(s => s.weight > 1000) // Only heavy shipments
);

// ─── take — take first N values then complete ───
shipments$.pipe(
  take(5) // Take first 5, then auto-complete. Like .take(5) in Reactor
);

// ─── first — take first value matching predicate ───
shipments$.pipe(
  first(s => s.status === 'delayed') // First delayed shipment
);

// ─── distinctUntilChanged — skip consecutive duplicates ───
this.searchControl.valueChanges.pipe(
  distinctUntilChanged() // "abc" → "abc" → "abcd" becomes "abc" → "abcd"
  // Like .distinctUntilChanged() in Reactor
);

// ─── debounceTime — wait for silence before emitting ───
this.searchControl.valueChanges.pipe(
  debounceTime(300) // Wait 300ms after user stops typing
  // No direct Reactor equivalent — this is an RxJS specialty for UI
);

// ─── throttleTime — emit first, then ignore for duration ───
this.scrollEvents$.pipe(
  throttleTime(200) // At most one event per 200ms
);
```

### 4.4 — Combination Operators

```typescript
import { combineLatest, forkJoin, merge, concat } from 'rxjs';

// ─── forkJoin — wait for ALL to complete, emit last values ───
// Like Mono.zip() in Reactor — parallel execution, wait for all
forkJoin({
  shipment: this.shipmentService.getById('SHP-001'),
  carrier:  this.carrierService.getById('CAR-05'),
  tracking: this.trackingService.getHistory('SHP-001')
}).subscribe(({ shipment, carrier, tracking }) => {
  // All 3 API calls completed — build the detail view
  this.detail = { shipment, carrier, tracking };
});

// ─── combineLatest — emit when ANY source emits (after all have emitted once) ───
// Like Flux.combineLatest() — reacts to latest from each
combineLatest([
  this.statusFilter$,
  this.dateFilter$,
  this.searchTerm$
]).pipe(
  switchMap(([status, date, term]) => 
    this.shipmentService.search({ status, date, term })
  )
).subscribe(results => this.filteredShipments = results);

// ─── merge — interleave emissions from multiple sources ───
// Like Flux.merge() — first-come, first-served
const allUpdates$ = merge(
  this.websocket.onShipmentUpdate(),
  this.polling.getUpdates(),
  this.manualRefresh$
);

// ─── concat — emit from sources sequentially ───
// Like Flux.concat() — complete first, then start second
const loadSequence$ = concat(
  this.loadCriticalShipments(),  // Load priority shipments first
  this.loadRemainingShipments()  // Then load the rest
);
```

### 4.5 — Error Handling Operators

```typescript
// ─── catchError — handle and recover (like .onErrorResume()) ───
this.shipmentService.getById(id).pipe(
  catchError(err => {
    if (err.status === 404) {
      return of(null); // Return null for not found
    }
    return throwError(() => err); // Re-throw other errors
  })
);

// ─── retry — retry N times on error (like .retry() in Reactor) ───
this.shipmentService.getById(id).pipe(
  retry(3) // Retry up to 3 times before erroring
);

// ─── retry with delay — exponential backoff ───
this.shipmentService.getById(id).pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(retryCount * 1000) // 1s, 2s, 3s
  })
);
```

### 4.6 — Utility Operators

```typescript
// ─── tap — side effects (like .doOnNext()) ───
shipments$.pipe(
  tap(data => console.log('Received:', data)),     // doOnNext
  tap({ error: err => console.error('Err:', err) }), // doOnError
  tap({ complete: () => console.log('Done') })      // doOnComplete
);

// ─── finalize — cleanup on complete OR error (like .doFinally()) ───
this.shipmentService.getAll().pipe(
  tap(() => this.loading = true),
  finalize(() => this.loading = false) // Always runs
);

// ─── delay — delay emissions ───
of('notification').pipe(
  delay(2000) // Emit after 2 seconds
);

// ─── timeout — error if no emission within duration ───
this.shipmentService.getById(id).pipe(
  timeout(5000) // Error if API doesn't respond within 5 seconds
);
```

---

## Phase 5: Subjects — Multicast Observables

> [!info] Subject = both Observable AND Observer. It can emit values AND be subscribed to.
> Think of it like `Sinks.Many` from Reactor — you can programmatically push values.

### 5.1 — Types of Subjects

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Subject Type      │ Behavior                        │ Reactor Equivalent  │
│───────────────────│─────────────────────────────────│─────────────────────│
│ Subject           │ No initial value, no replay     │ Sinks.many()        │
│ BehaviorSubject   │ Has current value, emits latest │ (no direct match)   │
│                   │ to new subscribers              │                     │
│ ReplaySubject     │ Replays last N values to new    │ Sinks.many().replay │
│                   │ subscribers                     │                     │
│ AsyncSubject      │ Emits ONLY last value on        │ Mono (sort of)      │
│                   │ complete                        │                     │
└───────────────────────────────────────────────────────────────────────────┘
```

### 5.2 — BehaviorSubject — The Most Used in Angular ⭐

```typescript
import { BehaviorSubject, Observable } from 'rxjs';

// Logistics: Shared state for selected shipment across components
@Injectable({ providedIn: 'root' })
export class ShipmentStateService {
  // Private BehaviorSubject — only this service can push values
  private selectedShipment$ = new BehaviorSubject<Shipment | null>(null);
  
  // Public Observable — consumers can only read, not write
  // Like exposing a Flux from a Sinks.Many — read-only view
  getSelected(): Observable<Shipment | null> {
    return this.selectedShipment$.asObservable();
  }
  
  // Get current value synchronously (BehaviorSubject exclusive feature!)
  getCurrentSelected(): Shipment | null {
    return this.selectedShipment$.getValue();
  }
  
  // Push a new value
  select(shipment: Shipment): void {
    this.selectedShipment$.next(shipment);
  }
  
  // Clear selection
  clearSelection(): void {
    this.selectedShipment$.next(null);
  }
}

// ─── Usage in Component ───
@Component({ /* ... */ })
export class ShipmentDetailComponent implements OnInit, OnDestroy {
  shipment: Shipment | null = null;
  private destroy$ = new Subject<void>();
  
  constructor(private shipmentState: ShipmentStateService) {}
  
  ngOnInit(): void {
    this.shipmentState.getSelected().pipe(
      takeUntil(this.destroy$) // Auto-unsubscribe pattern
    ).subscribe(s => this.shipment = s);
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 5.3 — Subject as Event Bus

```typescript
// Logistics: Notification service using Subject as event bus
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notifications$ = new Subject<Notification>();
  
  // Components subscribe to get notified
  getNotifications(): Observable<Notification> {
    return this.notifications$.asObservable();
  }
  
  // Any service/component can push notifications
  notify(message: string, type: 'success' | 'error' | 'info'): void {
    this.notifications$.next({ message, type, timestamp: new Date() });
  }
}
```

### 5.4 — ReplaySubject — Late Subscribers Get History

```typescript
// Logistics: Audit log — new subscribers get last 10 events
const auditLog$ = new ReplaySubject<AuditEvent>(10); // Buffer last 10

auditLog$.next({ action: 'SHIPMENT_CREATED', id: 'SHP-001' });
auditLog$.next({ action: 'STATUS_UPDATED', id: 'SHP-001' });
auditLog$.next({ action: 'CARRIER_ASSIGNED', id: 'SHP-001' });

// Late subscriber immediately receives all 3 events above
auditLog$.subscribe(event => console.log('Audit:', event));
```

---

## Phase 6: The async Pipe — Angular's Killer Feature

> [!tip] The `async` pipe is why Angular and RxJS work so well together.
> It **auto-subscribes** in the template and **auto-unsubscribes** on component destroy.
> No manual subscribe. No memory leaks. No `ngOnDestroy` cleanup.

### 6.1 — Basic Usage

```typescript
// ✅ Component — just expose the Observable, don't subscribe!
@Component({
  selector: 'app-shipment-list',
  template: `
    @if (shipments$ | async; as shipments) {
      <h2>Shipments ({{ shipments.length }})</h2>
      @for (s of shipments; track s.id) {
        <app-shipment-card [data]="s" />
      }
    } @else {
      <div class="loading-spinner">Loading shipments...</div>
    }
  `
})
export class ShipmentListComponent {
  // Just declare the Observable — the template handles subscription
  shipments$ = this.shipmentService.getAll();
  
  constructor(private shipmentService: ShipmentService) {}
  
  // No ngOnDestroy needed! async pipe handles unsubscription ✅
}
```

### 6.2 — Combining Multiple Observables in Template

```typescript
@Component({
  template: `
    @if (vm$ | async; as vm) {
      <h1>{{ vm.shipment.trackingId }}</h1>
      <p>Carrier: {{ vm.carrier.name }}</p>
      <app-tracking-timeline [events]="vm.tracking" />
    }
  `
})
export class ShipmentDetailComponent {
  // Combine multiple streams into a single view model Observable
  vm$ = combineLatest({
    shipment: this.shipmentService.getById(this.id),
    carrier:  this.carrierService.getForShipment(this.id),
    tracking: this.trackingService.getEvents(this.id)
  });
}
```

### 6.3 — Manual Subscribe vs async Pipe

```typescript
// ❌ BAD — Manual subscribe (imperative, error-prone)
export class ShipmentListComponent implements OnInit, OnDestroy {
  shipments: Shipment[] = [];
  private sub!: Subscription;
  
  ngOnInit() {
    this.sub = this.shipmentService.getAll()
      .subscribe(data => this.shipments = data);
  }
  
  ngOnDestroy() {
    this.sub.unsubscribe(); // Easy to forget!
  }
}

// ✅ GOOD — async pipe (declarative, leak-proof)
export class ShipmentListComponent {
  shipments$ = this.shipmentService.getAll();
  constructor(private shipmentService: ShipmentService) {}
  // That's it. No subscribe. No unsubscribe. No lifecycle hooks.
}
```

> [!success] Rule of Thumb
> **Prefer `async` pipe for 90% of cases.** Only use manual `.subscribe()` when you 
> need to trigger side effects (navigation, notifications) that can't be expressed in template.

---

## Phase 7: Angular Signals (Angular 16+)

> [!info] Signals are Angular's NEW reactive primitive — simpler than Observables for **synchronous component state**.
> Think of them as reactive variables that automatically notify consumers when they change.

### 7.1 — Signal Basics

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({ /* ... */ })
export class ShipmentDashboardComponent {
  // ─── signal() — reactive state (like a reactive variable) ───
  shipments = signal<Shipment[]>([]);
  searchTerm = signal('');
  loading = signal(false);
  
  // ─── computed() — derived state (auto-recalculates when deps change) ───
  // Like a @Cacheable method that auto-invalidates
  filteredShipments = computed(() => 
    this.shipments().filter(s => 
      s.trackingId.toLowerCase().includes(this.searchTerm().toLowerCase())
    )
  );
  
  totalWeight = computed(() => 
    this.filteredShipments().reduce((sum, s) => sum + s.weight, 0)
  );
  
  // ─── effect() — side effects when signals change ───
  constructor() {
    effect(() => {
      // Runs whenever filteredShipments changes
      console.log('Filtered shipments:', this.filteredShipments().length);
      // Angular tracks which signals are read inside this callback
    });
  }
  
  // ─── Updating signals ───
  onSearch(term: string): void {
    this.searchTerm.set(term);  // .set() replaces the value
  }
  
  addShipment(shipment: Shipment): void {
    this.shipments.update(current => [...current, shipment]); // .update() transforms
  }
}
```

### 7.2 — Signals vs Observables: When to Use Each

| Criteria | Signals | Observables (RxJS) |
|----------|---------|-------------------|
| **Nature** | Synchronous, always has a value | Async, may not have a value yet |
| **Use case** | Component state, UI state | HTTP calls, events, WebSockets |
| **Complexity** | Simple get/set | Complex transformations, timing |
| **Template** | Direct read: `{{ signal() }}` | Needs `async` pipe: `{{ obs$ \| async }}` |
| **Cleanup** | Automatic | Manual (unless async pipe) |
| **Operators** | `computed()`, `effect()` | 100+ operators (map, filter, switchMap...) |
| **Learning curve** | Low | High |

> [!tip] Decision Rule
> - **Simple component state** (form values, toggles, counters) → **Signals** ✅
> - **Async streams** (HTTP, WebSocket, events, timers) → **Observables** ✅
> - **Complex transformations** (debounce, retry, combine) → **Observables** ✅
> - **Shared state across components** → Either works, but **Signals** are simpler

### 7.3 — Converting Between Signals and Observables

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

// Observable → Signal
// Like converting a Flux<T> to a cached value
const shipments = toSignal(this.shipmentService.getAll(), { initialValue: [] });

// Signal → Observable
// Like converting a variable to a Flux that emits on change
const searchTerm$ = toObservable(this.searchTerm);

// Practical: Use signal for display, observable for complex processing
const search = signal('');
const searchResults = toSignal(
  toObservable(search).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.shipmentService.search(term))
  ),
  { initialValue: [] }
);
```

---

## Phase 8: Common Patterns

### 8.1 — Search with Debounce ⭐

```typescript
// The classic RxJS pattern — search-as-you-type with debounce
@Component({
  template: `
    <input [formControl]="searchControl" placeholder="Search shipments..." />
    @for (result of results$ | async; track result.id) {
      <app-shipment-row [shipment]="result" />
    }
  `
})
export class ShipmentSearchComponent {
  searchControl = new FormControl('');
  
  results$ = this.searchControl.valueChanges.pipe(
    debounceTime(300),          // Wait 300ms after typing stops
    distinctUntilChanged(),     // Don't search if term didn't change
    filter(term => term!.length >= 2), // Min 2 characters
    switchMap(term =>           // Cancel previous search, start new one
      this.shipmentService.search(term!).pipe(
        catchError(() => of([])) // Return empty on error
      )
    )
  );
  
  constructor(private shipmentService: ShipmentService) {}
}
```

### 8.2 — Parallel API Calls with forkJoin

```typescript
// Load a shipment detail page — need data from 3 APIs
// Like Mono.zip() in Reactor
loadShipmentDetail(id: string): void {
  this.loading = true;
  
  forkJoin({
    shipment:  this.shipmentService.getById(id),
    carrier:   this.carrierService.getForShipment(id),
    milestones: this.milestoneService.getForShipment(id)
  }).pipe(
    finalize(() => this.loading = false)
  ).subscribe({
    next: ({ shipment, carrier, milestones }) => {
      this.shipment = shipment;
      this.carrier = carrier;
      this.milestones = milestones;
    },
    error: (err) => this.handleError(err)
  });
}
```

### 8.3 — Polling with interval + switchMap

```typescript
// Logistics: Poll for shipment status updates every 30 seconds
// Like Flux.interval() in Reactor
shipmentStatus$ = interval(30_000).pipe(
  startWith(0), // Emit immediately on subscribe, then every 30s
  switchMap(() => this.shipmentService.getStatus(this.shipmentId)),
  distinctUntilChanged((prev, curr) => prev.status === curr.status),
  tap(status => {
    if (status.status === 'delivered') {
      this.notificationService.notify('Shipment delivered!', 'success');
    }
  })
);
```

### 8.4 — The takeUntil Pattern for Unsubscription

```typescript
// When you MUST use manual subscribe, use takeUntil for clean unsubscription
@Component({ /* ... */ })
export class ShipmentComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  
  ngOnInit(): void {
    // All subscriptions auto-cancel when destroy$ emits
    this.shipmentService.getUpdates().pipe(
      takeUntil(this.destroy$)
    ).subscribe(update => this.handleUpdate(update));
    
    this.notificationService.getAlerts().pipe(
      takeUntil(this.destroy$)
    ).subscribe(alert => this.showAlert(alert));
  }
  
  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### 8.5 — The DestroyRef Pattern (Angular 16+ — Even Cleaner)

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class ShipmentComponent implements OnInit {
  private destroyRef = inject(DestroyRef);
  
  ngOnInit(): void {
    // Auto-unsubscribes when component is destroyed
    this.shipmentService.getUpdates().pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(update => this.handleUpdate(update));
  }
  
  // No ngOnDestroy needed! ✅
}
```

---

## Quick Reference: Operator Cheat Sheet

| Category | Operator | Reactor Equivalent | What It Does |
|----------|----------|--------------------|-------------|
| Transform | `map` | `.map()` | Transform each value |
| Transform | `switchMap` | `.flatMap()` (cancel) | Flatten + cancel previous |
| Transform | `mergeMap` | `.flatMap()` | Flatten + parallel |
| Transform | `concatMap` | `.concatMap()` | Flatten + sequential |
| Filter | `filter` | `.filter()` | Keep matching values |
| Filter | `take` | `.take()` | Take first N values |
| Filter | `debounceTime` | — | Wait for silence |
| Filter | `distinctUntilChanged` | `.distinctUntilChanged()` | Skip consecutive dupes |
| Combine | `forkJoin` | `Mono.zip()` | Wait for all, emit once |
| Combine | `combineLatest` | `Flux.combineLatest()` | Latest from each source |
| Combine | `merge` | `Flux.merge()` | Interleave streams |
| Error | `catchError` | `.onErrorResume()` | Handle and recover |
| Error | `retry` | `.retry()` | Retry on error |
| Utility | `tap` | `.doOnNext()` | Side effects |
| Utility | `finalize` | `.doFinally()` | Cleanup on end |
| Utility | `delay` | `.delayElements()` | Delay emissions |
| Utility | `timeout` | `.timeout()` | Error if too slow |

---

## Common Mistakes ❌

| Mistake | Fix |
|---------|-----|
| Forgetting to unsubscribe | Use `async` pipe, `takeUntil`, or `takeUntilDestroyed` |
| Nesting subscribes | Use `switchMap` / `mergeMap` instead |
| Using `mergeMap` for search | Use `switchMap` to cancel previous |
| Not handling errors | Always add `catchError` in the pipe |
| Subscribing multiple times to HTTP | Use `shareReplay(1)` to cache |
| Using Observables for simple state | Use Signals (Angular 16+) |

---

## Related Notes

- [[Angular Fundamentals]] — Core Angular concepts
- [[Services and Dependency Injection]] — Where RxJS patterns live
- [[Angular State Management]] — State patterns using RxJS and Signals
- [[Reactive Programming]] — Your Spring WebFlux reactive foundations
- [[React vs Angular Comparison]] — How React handles similar patterns

---

> [!quote] "Once you've mastered Project Reactor chains, RxJS is just a syntax change. The reactive mindset is the hard part — and you already have that."
