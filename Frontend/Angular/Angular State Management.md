---
title: Angular State Management
tags:
  - angular
  - state-management
  - ngrx
  - signals
  - rxjs
  - frontend
created: 2025-07-14
status: in-progress
related:
  - "[[RxJS and Reactive Programming]]"
  - "[[Services and Dependency Injection]]"
  - "[[React State Management]]"
  - "[[React vs Angular Comparison]]"
---

# Angular State Management

> [!quote] "State management in Angular is like inventory management in a warehouse — you need to know what's where, keep it consistent, and make sure everyone sees the same data."

You already manage state in backend services — `@Service` beans with fields, caches, database state.
Frontend state management is the same concept, applied to **UI data that multiple components need**.

---

## Phase 1: What is State Management?

### 1.1 — The Problem

Every Angular app deals with **data that needs to be shared, synchronized, and persisted** across
components. As apps grow, passing data through `@Input()` / `@Output()` becomes unmanageable.

```
Without State Management:

┌──────────────┐          ┌──────────────┐
│  Header      │          │  Sidebar     │
│  (needs      │ ← ??? → │  (needs      │
│  user info)  │          │  user info)  │
└──────────────┘          └──────────────┘
       ↑                         ↑
       │     Where does the      │
       │     user info live?     │
       ↓                         ↓
┌──────────────┐          ┌──────────────┐
│  Dashboard   │          │  Settings    │
│  (needs      │          │  (updates    │
│  user info)  │          │  user info)  │
└──────────────┘          └──────────────┘

Problem: 4 components need the same data. 
Who owns it? How do updates propagate?
```

### 1.2 — Types of State

| State Type | Description | Example | Backend Analogy |
|-----------|-------------|---------|-----------------|
| **Component State** | Local to one component | Form inputs, toggle expanded/collapsed | Method-local variables |
| **Shared State** | Shared across components | Current user, selected shipment | `@Service` singleton fields |
| **Server State** | Data from API, cached locally | Shipment list from REST API | Database + Cache layer |
| **URL State** | Encoded in the URL | `/shipments?status=in-transit&page=2` | Request parameters |
| **Form State** | Form values + validation status | Booking form with validation errors | DTO validation state |

### 1.3 — The State Management Spectrum

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STATE MANAGEMENT SPECTRUM                                │
│                                                                             │
│  Simple ◄──────────────────────────────────────────────────────► Complex   │
│                                                                             │
│  Component      Signals      Service +       NgRx            NgRx          │
│  Properties                  BehaviorSubject Component       Store          │
│                                              Store                          │
│                                                                             │
│  ┌─────┐       ┌─────┐      ┌─────┐        ┌─────┐        ┌─────┐        │
│  │ 👤  │       │ ⚡  │      │ 📦  │        │ 📋  │        │ 🏗️  │        │
│  │Local│       │React│      │Share│        │Light│        │Full │        │
│  │State│       │ive  │      │d    │        │     │        │Redux│        │
│  └─────┘       └─────┘      └─────┘        └─────┘        └─────┘        │
│                                                                             │
│  Solo page     Dashboard    Multi-page      Feature-       Enterprise       │
│  CRUD          filters      workflows       level          apps with        │
│                              with shared     complex        audit, undo,     │
│                              data            state          time-travel      │
│                                                                             │
│  Spring MVC    @Cacheable   @Service        CQRS           Event            │
│  Controller    method       singleton       pattern        Sourcing         │
│  (analogy)     (analogy)    (analogy)       (analogy)      (analogy)        │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!tip] Start simple. Add complexity only when the simpler approach causes pain.
> "Premature state management is the root of all evil" — adapted from Knuth

### 1.4 — Backend Developer Analogy

Think of frontend state management layers like backend architecture layers:

| Backend Layer | Frontend Equivalent | Purpose |
|--------------|-------------------|---------|
| Controller `@RequestMapping` | Component | Handles user interaction |
| Service `@Service` | Angular Service / Store | Business logic + state |
| Repository `@Repository` | HttpClient calls | Data access |
| Entity / DTO | Interface / Model | Data shape |
| Spring Cache `@Cacheable` | BehaviorSubject / Signal cache | Avoid redundant lookups |
| Event Bus (Spring Events) | Subject / NgRx Actions | Decouple producers and consumers |

---

## Phase 2: Component State (Simplest)

> [!info] Component state = data that only ONE component cares about.
> No sharing needed. Keep it local.

### 2.1 — Plain Class Properties

```typescript
// Simplest form — just class properties
// Like local variables in a Spring controller method
@Component({
  selector: 'app-shipment-list',
  template: `
    <button (click)="toggleFilters()">
      {{ showFilters ? 'Hide' : 'Show' }} Filters
    </button>
    
    @if (showFilters) {
      <app-filter-panel />
    }
    
    @if (loading) {
      <div class="spinner">Loading...</div>
    }
  `
})
export class ShipmentListComponent {
  showFilters = false;  // UI toggle — no one else needs this
  loading = false;       // Loading indicator
  shipments: Shipment[] = [];
  
  toggleFilters(): void {
    this.showFilters = !this.showFilters;
  }
}
```

### 2.2 — Signals for Reactive Local State (Angular 16+)

```typescript
import { signal, computed } from '@angular/core';

// Signals make local state reactive — computed values auto-update
@Component({
  selector: 'app-shipment-list',
  template: `
    <div class="toolbar">
      <input (input)="searchTerm.set($any($event.target).value)" 
             placeholder="Search..." />
      <select (change)="selectedStatus.set($any($event.target).value)">
        <option value="all">All Statuses</option>
        <option value="in-transit">In Transit</option>
        <option value="delivered">Delivered</option>
        <option value="delayed">Delayed</option>
      </select>
      <span class="count">
        Showing {{ filteredShipments().length }} of {{ shipments().length }}
      </span>
    </div>
    
    @for (s of filteredShipments(); track s.id) {
      <app-shipment-card [shipment]="s" />
    }
  `
})
export class ShipmentListComponent {
  // ─── Writable Signals (state) ───
  shipments = signal<Shipment[]>([]);
  loading = signal(false);
  selectedStatus = signal<string>('all');
  searchTerm = signal('');
  
  // ─── Computed Signals (derived state — auto-recalculates) ───
  // Like @Cacheable that auto-invalidates when inputs change
  filteredShipments = computed(() => {
    let result = this.shipments();
    
    // Filter by status
    if (this.selectedStatus() !== 'all') {
      result = result.filter(s => s.status === this.selectedStatus());
    }
    
    // Filter by search term
    const term = this.searchTerm().toLowerCase();
    if (term) {
      result = result.filter(s => 
        s.trackingId.toLowerCase().includes(term) ||
        s.origin.toLowerCase().includes(term) ||
        s.destination.toLowerCase().includes(term)
      );
    }
    
    return result;
  });
  
  // ─── Stats derived from filtered data ───
  totalWeight = computed(() => 
    this.filteredShipments().reduce((sum, s) => sum + s.weight, 0)
  );
  
  delayedCount = computed(() => 
    this.filteredShipments().filter(s => s.status === 'delayed').length
  );
  
  // ─── Load data from API ───
  constructor(private shipmentService: ShipmentService) {
    this.loadShipments();
  }
  
  loadShipments(): void {
    this.loading.set(true);
    this.shipmentService.getAll().subscribe({
      next: (data) => {
        this.shipments.set(data);
        this.loading.set(false);
      },
      error: () => this.loading.set(false)
    });
  }
}
```

> [!success] When to use Component State
> ✅ UI toggles (show/hide, expanded/collapsed)
> ✅ Form input values
> ✅ Local loading/error states
> ✅ Pagination state
> ✅ Data that ONLY this component and its children need

---

## Phase 3: Services as State Stores

> [!info] The "Angular Way" — use `@Injectable` services as shared state stores.
> This is like a Spring `@Service` singleton that also holds state.
> Most Angular apps should start here before reaching for NgRx.

### 3.1 — The Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE AS STATE STORE                        │
│                                                                 │
│   Component A ──read──►  ┌──────────────────┐  ◄──read── Component B
│                          │  ShipmentStore    │                  │
│   Component C ──write──► │  (@Injectable)    │  ◄──write── Component D
│                          │                    │                  │
│                          │  state: Signal     │                  │
│                          │  shipments()       │                  │
│                          │  loading()         │                  │
│                          │  selectedId()      │                  │
│                          │                    │                  │
│                          │  loadShipments()   │                  │
│                          │  selectShipment()  │                  │
│                          │  updateStatus()    │                  │
│                          └──────────────────┘                  │
│                                                                 │
│   Like a Spring @Service singleton:                             │
│   - All components get the SAME instance                       │
│   - State is shared automatically                               │
│   - Methods mutate state — consumers react                      │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 — Full Example: ShipmentStore Service

```typescript
// ─── State Interface ───
interface ShipmentState {
  shipments: Shipment[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
  filters: {
    status: string;
    search: string;
  };
}

// ─── Initial State ───
const initialState: ShipmentState = {
  shipments: [],
  loading: false,
  error: null,
  selectedId: null,
  filters: { status: 'all', search: '' }
};

// ─── Store Service ───
@Injectable({ providedIn: 'root' })
export class ShipmentStore {
  // Private mutable state — only this service can modify
  private state = signal<ShipmentState>(initialState);
  
  // ══════════════════════════════════════════════════════
  // SELECTORS — computed signals for reading state
  // Like Spring @Cacheable getters that auto-invalidate
  // ══════════════════════════════════════════════════════
  
  shipments = computed(() => this.state().shipments);
  loading = computed(() => this.state().loading);
  error = computed(() => this.state().error);
  selectedId = computed(() => this.state().selectedId);
  
  selectedShipment = computed(() => 
    this.state().shipments.find(s => s.id === this.state().selectedId) ?? null
  );
  
  filteredShipments = computed(() => {
    const { shipments, filters } = this.state();
    let result = shipments;
    
    if (filters.status !== 'all') {
      result = result.filter(s => s.status === filters.status);
    }
    if (filters.search) {
      const term = filters.search.toLowerCase();
      result = result.filter(s => s.trackingId.toLowerCase().includes(term));
    }
    return result;
  });
  
  shipmentCount = computed(() => this.filteredShipments().length);
  
  // ══════════════════════════════════════════════════════
  // ACTIONS — methods that modify state
  // Like Spring @Service methods
  // ══════════════════════════════════════════════════════
  
  constructor(private shipmentService: ShipmentService) {}
  
  loadShipments(): void {
    this.state.update(s => ({ ...s, loading: true, error: null }));
    
    this.shipmentService.getAll().subscribe({
      next: (shipments) => {
        this.state.update(s => ({ ...s, shipments, loading: false }));
      },
      error: (err) => {
        this.state.update(s => ({ 
          ...s, 
          loading: false, 
          error: err.message || 'Failed to load shipments' 
        }));
      }
    });
  }
  
  selectShipment(id: string | null): void {
    this.state.update(s => ({ ...s, selectedId: id }));
  }
  
  setFilter(key: 'status' | 'search', value: string): void {
    this.state.update(s => ({
      ...s,
      filters: { ...s.filters, [key]: value }
    }));
  }
  
  updateShipmentStatus(id: string, status: string): void {
    // Optimistic update (update UI immediately, revert on error)
    const previousShipments = this.state().shipments;
    
    this.state.update(s => ({
      ...s,
      shipments: s.shipments.map(ship => 
        ship.id === id ? { ...ship, status } : ship
      )
    }));
    
    this.shipmentService.updateStatus(id, status).subscribe({
      error: () => {
        // Revert on error — like a database transaction rollback
        this.state.update(s => ({ ...s, shipments: previousShipments }));
      }
    });
  }
  
  removeShipment(id: string): void {
    this.state.update(s => ({
      ...s,
      shipments: s.shipments.filter(ship => ship.id !== id),
      selectedId: s.selectedId === id ? null : s.selectedId
    }));
  }
}
```

### 3.3 — Using the Store in Components

```typescript
// ─── List Component — reads shipments ───
@Component({
  selector: 'app-shipment-list',
  template: `
    <div class="toolbar">
      <input (input)="store.setFilter('search', $any($event.target).value)" 
             placeholder="Search..." />
      <select (change)="store.setFilter('status', $any($event.target).value)">
        <option value="all">All</option>
        <option value="in-transit">In Transit</option>
        <option value="delivered">Delivered</option>
      </select>
      <span>{{ store.shipmentCount() }} shipments</span>
    </div>
    
    @if (store.loading()) {
      <app-loading-spinner />
    } @else if (store.error()) {
      <app-error-banner [message]="store.error()!" (retry)="store.loadShipments()" />
    } @else {
      @for (s of store.filteredShipments(); track s.id) {
        <app-shipment-card 
          [shipment]="s" 
          [selected]="s.id === store.selectedId()"
          (click)="store.selectShipment(s.id)" 
        />
      }
    }
  `
})
export class ShipmentListComponent implements OnInit {
  constructor(protected store: ShipmentStore) {}
  
  ngOnInit(): void {
    this.store.loadShipments();
  }
}

// ─── Detail Component — reads selected shipment ───
@Component({
  selector: 'app-shipment-detail',
  template: `
    @if (store.selectedShipment(); as shipment) {
      <h2>{{ shipment.trackingId }}</h2>
      <p>{{ shipment.origin }} → {{ shipment.destination }}</p>
      <p>Status: {{ shipment.status }}</p>
      <button (click)="store.updateShipmentStatus(shipment.id, 'delivered')">
        Mark Delivered
      </button>
    } @else {
      <p class="placeholder">Select a shipment to view details</p>
    }
  `
})
export class ShipmentDetailComponent {
  constructor(protected store: ShipmentStore) {}
}
```

### 3.4 — BehaviorSubject Alternative (RxJS-based Store)

> [!info] Before Signals existed (pre-Angular 16), services used BehaviorSubject.
> You'll see this in many existing codebases.

```typescript
// Same pattern, but with BehaviorSubject instead of Signals
// See [[RxJS and Reactive Programming]] Phase 5 for Subject details

@Injectable({ providedIn: 'root' })
export class ShipmentStoreRxJS {
  private state$ = new BehaviorSubject<ShipmentState>(initialState);
  
  // Selectors — derived Observables
  shipments$ = this.state$.pipe(map(s => s.shipments), distinctUntilChanged());
  loading$   = this.state$.pipe(map(s => s.loading), distinctUntilChanged());
  error$     = this.state$.pipe(map(s => s.error), distinctUntilChanged());
  
  selectedShipment$ = this.state$.pipe(
    map(s => s.shipments.find(sh => sh.id === s.selectedId) ?? null),
    distinctUntilChanged()
  );
  
  // Actions
  loadShipments(): void {
    this.updateState({ loading: true, error: null });
    this.shipmentService.getAll().subscribe({
      next: (shipments) => this.updateState({ shipments, loading: false }),
      error: (err) => this.updateState({ loading: false, error: err.message })
    });
  }
  
  private updateState(partial: Partial<ShipmentState>): void {
    this.state$.next({ ...this.state$.getValue(), ...partial });
  }
}

// Usage in template with async pipe
// <div *ngIf="store.loading$ | async">Loading...</div>
// @for (s of store.shipments$ | async; track s.id) { ... }
```

> [!tip] Signals vs BehaviorSubject for State Stores
> **New projects (Angular 16+)** → Prefer Signals — simpler, no async pipe needed
> **Existing projects** → BehaviorSubject works fine, migrate gradually
> **Complex async flows** → BehaviorSubject / Observable may still be better

---

## Phase 4: NgRx — Redux for Angular

> [!warning] NgRx adds significant boilerplate. Only use it when services aren't enough.
> For most apps, Services + Signals is sufficient.

### 4.1 — When Do You Need NgRx?

✅ **Use NgRx when:**
- Team is large (5+ developers) and needs strict patterns
- App has complex state interactions (many cross-cutting concerns)
- You need time-travel debugging (NgRx DevTools)
- Audit trail of every state change (action log)
- You're already using CQRS/Event Sourcing on the backend

❌ **Don't use NgRx when:**
- App is small-medium (CRUD with a few shared states)
- Team is small and communication is easy
- You want quick development velocity
- The boilerplate cost exceeds the debugging benefit

### 4.2 — NgRx Architecture

> [!quote] "NgRx is to Angular what CQRS + Event Sourcing is to backend services."

```
┌────────────────────────────────────────────────────────────────────┐
│                        NgRx Architecture                           │
│                                                                    │
│   ┌───────────┐  dispatch()  ┌───────────┐  updates  ┌─────────┐ │
│   │ Component │ ──────────►  │  Actions   │ ────────► │ Reducer │ │
│   │           │              │ (Events)   │           │ (Pure   │ │
│   │           │              └───────────┘           │  Func)  │ │
│   │           │                    │                  └────┬────┘ │
│   │           │                    │ triggers               │      │
│   │           │                    ▼                        ▼      │
│   │           │              ┌───────────┐           ┌─────────┐ │
│   │           │ ◄─────────── │  Effects   │           │  Store  │ │
│   │           │   select()   │ (Side      │           │ (State) │ │
│   │           │ ◄────────────│  Effects)  │           │         │ │
│   └───────────┘              └───────────┘           └─────────┘ │
│                                                            │      │
│                              ┌───────────┐                │      │
│                              │ Selectors │ ◄──────────────┘      │
│                              │ (Queries) │                        │
│                              └───────────┘                        │
│                                                                    │
│   Backend Analogy:                                                │
│   Actions    = Commands/Events  (CQRS command bus)                │
│   Reducer    = Event handler    (applies event to aggregate)      │
│   Store      = Database         (single source of truth)          │
│   Selectors  = Queries/Views    (read models)                     │
│   Effects    = @EventListener   (side effects on events)          │
└────────────────────────────────────────────────────────────────────┘
```

### 4.3 — NgRx: Complete Shipment Example

#### Step 1: Define State & Actions

```typescript
// ─── shipment.models.ts ───
export interface Shipment {
  id: string;
  trackingId: string;
  origin: string;
  destination: string;
  status: string;
  weight: number;
}

export interface ShipmentState {
  shipments: Shipment[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

export const initialShipmentState: ShipmentState = {
  shipments: [],
  loading: false,
  error: null,
  selectedId: null
};

// ─── shipment.actions.ts ───
import { createActionGroup, emptyProps, props } from '@ngrx/store';

// Actions are like CQRS commands — they describe WHAT happened
export const ShipmentActions = createActionGroup({
  source: 'Shipments',
  events: {
    // Load
    'Load Shipments': emptyProps(),
    'Load Shipments Success': props<{ shipments: Shipment[] }>(),
    'Load Shipments Failure': props<{ error: string }>(),
    
    // Select
    'Select Shipment': props<{ id: string }>(),
    
    // Update
    'Update Status': props<{ id: string; status: string }>(),
    'Update Status Success': props<{ shipment: Shipment }>(),
    'Update Status Failure': props<{ error: string }>(),
  }
});
```

#### Step 2: Create Reducer

```typescript
// ─── shipment.reducer.ts ───
import { createReducer, on } from '@ngrx/store';

// Reducer is a PURE FUNCTION — no side effects
// Like an event handler that applies an event to the aggregate state
export const shipmentReducer = createReducer(
  initialShipmentState,
  
  // Loading
  on(ShipmentActions.loadShipments, (state) => ({
    ...state,
    loading: true,
    error: null
  })),
  
  on(ShipmentActions.loadShipmentsSuccess, (state, { shipments }) => ({
    ...state,
    shipments,
    loading: false
  })),
  
  on(ShipmentActions.loadShipmentsFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error
  })),
  
  // Selection
  on(ShipmentActions.selectShipment, (state, { id }) => ({
    ...state,
    selectedId: id
  })),
  
  // Status Update
  on(ShipmentActions.updateStatusSuccess, (state, { shipment }) => ({
    ...state,
    shipments: state.shipments.map(s => 
      s.id === shipment.id ? shipment : s
    )
  }))
);
```

#### Step 3: Create Selectors

```typescript
// ─── shipment.selectors.ts ───
import { createFeatureSelector, createSelector } from '@ngrx/store';

// Feature selector — grabs the shipment slice from global store
const selectShipmentState = createFeatureSelector<ShipmentState>('shipments');

// Selectors are like database views / read models
export const selectAllShipments = createSelector(
  selectShipmentState,
  (state) => state.shipments
);

export const selectLoading = createSelector(
  selectShipmentState,
  (state) => state.loading
);

export const selectError = createSelector(
  selectShipmentState,
  (state) => state.error
);

export const selectSelectedId = createSelector(
  selectShipmentState,
  (state) => state.selectedId
);

// Composed selector — like a JOIN in SQL
export const selectSelectedShipment = createSelector(
  selectAllShipments,
  selectSelectedId,
  (shipments, selectedId) => shipments.find(s => s.id === selectedId) ?? null
);

// Parameterized selector
export const selectShipmentsByStatus = (status: string) => createSelector(
  selectAllShipments,
  (shipments) => shipments.filter(s => s.status === status)
);
```

#### Step 4: Create Effects (Side Effects)

```typescript
// ─── shipment.effects.ts ───
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import { catchError, map, switchMap } from 'rxjs/operators';

// Effects handle side effects — like @EventListener in Spring
@Injectable()
export class ShipmentEffects {
  
  // When LoadShipments is dispatched → call API → dispatch Success or Failure
  loadShipments$ = createEffect(() => this.actions$.pipe(
    ofType(ShipmentActions.loadShipments),
    switchMap(() => 
      this.shipmentService.getAll().pipe(
        map(shipments => ShipmentActions.loadShipmentsSuccess({ shipments })),
        catchError(err => of(ShipmentActions.loadShipmentsFailure({ 
          error: err.message 
        })))
      )
    )
  ));
  
  // When UpdateStatus is dispatched → call API → dispatch Success or Failure
  updateStatus$ = createEffect(() => this.actions$.pipe(
    ofType(ShipmentActions.updateStatus),
    switchMap(({ id, status }) =>
      this.shipmentService.updateStatus(id, status).pipe(
        map(shipment => ShipmentActions.updateStatusSuccess({ shipment })),
        catchError(err => of(ShipmentActions.updateStatusFailure({ 
          error: err.message 
        })))
      )
    )
  ));
  
  constructor(
    private actions$: Actions,
    private shipmentService: ShipmentService
  ) {}
}
```

#### Step 5: Wire Up in Component

```typescript
// ─── shipment-list.component.ts ───
import { Store } from '@ngrx/store';

@Component({
  selector: 'app-shipment-list',
  template: `
    @if (loading$ | async) {
      <app-loading-spinner />
    }
    
    @if (error$ | async; as error) {
      <app-error-banner [message]="error" />
    }
    
    @for (s of shipments$ | async; track s.id) {
      <app-shipment-card 
        [shipment]="s"
        (click)="selectShipment(s.id)"
        (statusChange)="updateStatus(s.id, $event)" 
      />
    }
  `
})
export class ShipmentListComponent implements OnInit {
  // Read from store using selectors
  shipments$ = this.store.select(selectAllShipments);
  loading$   = this.store.select(selectLoading);
  error$     = this.store.select(selectError);
  
  constructor(private store: Store) {}
  
  ngOnInit(): void {
    // Dispatch action — like sending a command
    this.store.dispatch(ShipmentActions.loadShipments());
  }
  
  selectShipment(id: string): void {
    this.store.dispatch(ShipmentActions.selectShipment({ id }));
  }
  
  updateStatus(id: string, status: string): void {
    this.store.dispatch(ShipmentActions.updateStatus({ id, status }));
  }
}
```

### 4.4 — NgRx Data Flow Visualized

```
User clicks "Load Shipments"
    │
    ▼
Component: store.dispatch(loadShipments())
    │
    ▼
Action: [Shipments] Load Shipments                    ← Like a CQRS Command
    │
    ├──► Reducer: state = { ...state, loading: true }  ← Sync state update
    │
    └──► Effect: API call → shipmentService.getAll()   ← Like @EventListener
              │
              ▼
         API returns data
              │
              ▼
         Effect dispatches: loadShipmentsSuccess({ shipments })
              │
              ▼
         Reducer: state = { ...state, shipments, loading: false }
              │
              ▼
         Selector: selectAllShipments → returns updated list
              │
              ▼
         Component: shipments$ | async → template re-renders
```

### 4.5 — When to Use NgRx vs Services

| Criteria | Services + Signals | NgRx |
|----------|-------------------|------|
| **Team size** | 1-4 developers | 5+ developers |
| **App complexity** | Simple to medium | Large, complex |
| **Boilerplate** | Minimal | Significant (actions, reducers, effects, selectors) |
| **Debugging** | `console.log` / browser DevTools | NgRx DevTools with time-travel |
| **Learning curve** | Low (just services) | High (Redux concepts) |
| **Predictability** | Good (if disciplined) | Excellent (enforced patterns) |
| **Testing** | Standard unit tests | Pure function reducers = easy to test |
| **Undo/Redo** | Manual implementation | Built-in with time-travel |
| **Audit trail** | Not built-in | Every action logged automatically |
| **Backend analogy** | `@Service` with state | CQRS + Event Sourcing |

> [!warning] NgRx Honest Assessment
> NgRx is powerful but heavy. For a 3-person team building a shipment tracker, 
> **Services + Signals will get you 90% of the benefit with 20% of the boilerplate.**
> Only reach for NgRx when you genuinely need strict patterns or DevTools debugging.

---

## Phase 5: NgRx Component Store

> [!info] NgRx Component Store = lightweight NgRx for component-level state.
> Less boilerplate than full NgRx, more structure than plain services.
> Good middle ground.

### 5.1 — When to Use Component Store

```
Services ◄────── NgRx Component Store ──────► Full NgRx
(too loose)      (just right for features)    (too heavy)
```

Use when:
- A feature has complex state but doesn't need global store
- You want structure without full NgRx overhead
- State is scoped to a component subtree (not app-wide)

### 5.2 — Component Store Example

```typescript
import { Injectable } from '@angular/core';
import { ComponentStore, tapResponse } from '@ngrx/component-store';
import { switchMap } from 'rxjs/operators';

interface ShipmentListState {
  shipments: Shipment[];
  loading: boolean;
  error: string | null;
  page: number;
}

@Injectable() // NOT providedIn: 'root' — scoped to component
export class ShipmentListStore extends ComponentStore<ShipmentListState> {
  
  constructor(private shipmentService: ShipmentService) {
    super({
      shipments: [],
      loading: false,
      error: null,
      page: 1
    });
  }
  
  // ─── Selectors ───
  readonly shipments$ = this.select(state => state.shipments);
  readonly loading$ = this.select(state => state.loading);
  readonly vm$ = this.select({
    shipments: this.shipments$,
    loading: this.loading$,
    error: this.select(state => state.error)
  });
  
  // ─── Updaters (synchronous state changes) ───
  readonly setPage = this.updater((state, page: number) => ({
    ...state,
    page
  }));
  
  // ─── Effects (async operations) ───
  readonly loadShipments = this.effect<void>(trigger$ =>
    trigger$.pipe(
      switchMap(() => {
        this.patchState({ loading: true });
        return this.shipmentService.getAll().pipe(
          tapResponse(
            (shipments) => this.patchState({ shipments, loading: false }),
            (error: Error) => this.patchState({ error: error.message, loading: false })
          )
        );
      })
    )
  );
}

// ─── Usage ───
@Component({
  selector: 'app-shipment-list',
  providers: [ShipmentListStore], // Scoped to this component!
  template: `
    @if (store.vm$ | async; as vm) {
      @if (vm.loading) { <spinner /> }
      @for (s of vm.shipments; track s.id) {
        <app-shipment-card [shipment]="s" />
      }
    }
  `
})
export class ShipmentListComponent implements OnInit {
  constructor(protected store: ShipmentListStore) {}
  
  ngOnInit(): void {
    this.store.loadShipments();
  }
}
```

---

## Phase 6: Comparison with React State Management

> [!tip] If you're also learning React, this mapping helps.
> See [[React State Management]] for the React side.

| Angular Approach | React Equivalent | Description |
|-----------------|-----------------|-------------|
| Component properties | `useState` | Local component state |
| Signals | `useState` + `useMemo` | Reactive local state |
| Service + BehaviorSubject | Context API + `useReducer` | Built-in shared state |
| Service + Signals | Zustand / Jotai | Lightweight reactive store |
| NgRx Store | Redux Toolkit | Full-featured global state |
| NgRx Component Store | `useReducer` + Context | Feature-level complex state |
| `@Input()` / `@Output()` | Props / Callbacks | Parent-child communication |
| Service EventBus (Subject) | Custom Events / Zustand | Cross-component events |

### 6.1 — Key Philosophical Difference

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   Angular:  "Services ARE the state management solution.       │
│              @Injectable + providedIn: 'root' = singleton."    │
│                                                                │
│   React:    "Here are 47 state libraries. Choose wisely.       │
│              Context? Redux? Zustand? Jotai? MobX? Recoil?"   │
│                                                                │
│   Angular wins on: Opinionated structure, DI-based sharing     │
│   React wins on:   Flexibility, simpler mental model           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

> [!info] As a Spring Boot developer, Angular's DI-based state management will feel natural.
> Services in Angular work almost identically to `@Service` beans in Spring.

---

## Phase 7: Decision Matrix — Choosing the Right Approach

### 7.1 — Decision Flowchart

```
                    START: "I need to manage state"
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Does only ONE       │
                   │ component need it?  │
                   └─────────┬───────────┘
                        YES  │  NO
                   ┌─────────┘  └──────────────────┐
                   ▼                                ▼
          ┌──────────────┐              ┌─────────────────────┐
          │ Component    │              │ Do multiple         │
          │ State        │              │ FEATURES share it?  │
          │ (Signals or  │              └─────────┬───────────┘
          │  properties) │                   YES  │  NO
          └──────────────┘              ┌─────────┘  └──────────┐
                                        ▼                        ▼
                              ┌──────────────┐        ┌──────────────────┐
                              │ Service +    │        │ NgRx Component   │
                              │ Signals /    │        │ Store            │
                              │ BehaviorSubj │        │ (scoped to       │
                              └──────┬───────┘        │  feature)        │
                                     │                └──────────────────┘
                                     ▼
                          ┌─────────────────────┐
                          │ Is the app LARGE     │
                          │ with complex cross-  │
                          │ cutting state flows? │
                          └─────────┬───────────┘
                               YES  │  NO
                          ┌─────────┘  └──────┐
                          ▼                    ▼
                  ┌──────────────┐    ┌──────────────┐
                  │ NgRx Store   │    │ Service +    │
                  │ (full Redux) │    │ Signals is   │
                  │              │    │ ENOUGH ✅     │
                  └──────────────┘    └──────────────┘
```

### 7.2 — Quick Decision Table

| Scenario | Recommended Approach | Why |
|----------|---------------------|-----|
| Toggle sidebar open/close | Component property | Purely local UI state |
| Filter + sort a table | Component Signals | Local derived state with `computed()` |
| Share selected shipment across views | Service + Signal | Simple shared state |
| Complex form wizard (multi-step) | NgRx Component Store | Complex local state with effects |
| Enterprise app with 50+ features | NgRx Store | Need strict patterns + DevTools |
| Authentication state (user, tokens) | Service + Signal | Shared but simple |
| Real-time dashboard with WebSocket | Service + BehaviorSubject | Async streams need RxJS |
| Offline-first with sync queue | NgRx Store + Effects | Complex async state machine |

### 7.3 — Best Practices

> [!success] State Management Rules of Thumb

**1. Start Simple, Scale Up**
```
Component State → Service + Signals → NgRx Component Store → NgRx Store
     Start here ─────────────────────────────── Move right ONLY when needed
```

**2. Single Source of Truth**
- Each piece of state should live in ONE place
- Like a database — one authoritative source, many readers

**3. Immutable Updates**
```typescript
// ❌ BAD — mutating state directly
this.shipments.push(newShipment);

// ✅ GOOD — create new reference (like functional Reactor chains)
this.shipments.set([...this.shipments(), newShipment]);
```

**4. Separate Read from Write**
```typescript
// ✅ GOOD — clear separation (like CQRS)
// Reads: computed signals / selectors (derived, cached)
filteredShipments = computed(() => this.state().shipments.filter(...));

// Writes: explicit methods (commands)
addShipment(s: Shipment): void {
  this.state.update(st => ({ ...st, shipments: [...st.shipments, s] }));
}
```

**5. Keep State Normalized**
```typescript
// ❌ BAD — duplicated shipment data
state = {
  inTransitShipments: [...],
  deliveredShipments: [...],
  allShipments: [...]  // Overlap!
};

// ✅ GOOD — single list, derive views
state = {
  shipments: [...],  // Single source
  filters: { status: 'in-transit' }
};
// Compute: filteredShipments = shipments.filter(s => s.status === filters.status)
```

**6. Signals for Sync, Observables for Async**
```typescript
// Synchronous state → Signals
count = signal(0);
name = signal('');
items = signal<Item[]>([]);

// Asynchronous streams → Observables  
// (HTTP calls, WebSocket, timer, user events with complex timing)
results$ = this.http.get('/api/data');
updates$ = this.websocket.listen('shipment-updates');
search$  = this.searchInput.valueChanges.pipe(debounceTime(300), switchMap(...));
```

---

## Summary: State Management at a Glance

```
┌────────────────────────────────────────────────────────────────────────┐
│ Approach              │ Complexity │ Boilerplate │ Best For            │
│───────────────────────│────────────│─────────────│─────────────────────│
│ Component Properties  │ ⭐          │ None        │ Local UI state      │
│ Signals               │ ⭐⭐        │ Minimal     │ Reactive local state│
│ Service + Signals     │ ⭐⭐⭐      │ Low         │ Shared state (most) │
│ Service + BehaviorSubj│ ⭐⭐⭐      │ Low-Medium  │ Async shared state  │
│ NgRx Component Store  │ ⭐⭐⭐⭐    │ Medium      │ Feature-level state │
│ NgRx Store            │ ⭐⭐⭐⭐⭐  │ High        │ Enterprise apps     │
└────────────────────────────────────────────────────────────────────────┘
```

> [!quote] "The best state management solution is the simplest one that solves your problem."

---

## Related Notes

- [[RxJS and Reactive Programming]] — The reactive foundation (Observables, Subjects, operators)
- [[Services and Dependency Injection]] — Where state stores live in Angular
- [[Angular Fundamentals]] — Component basics, lifecycle hooks
- [[React State Management]] — Compare with React's approach
- [[React vs Angular Comparison]] — Broader framework comparison

---

> [!quote] "In the backend, you'd never use Event Sourcing for a simple CRUD app. Same rule applies here — don't use NgRx when a service will do."
