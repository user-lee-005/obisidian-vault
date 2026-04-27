---
title: Angular Testing
tags:
  - angular
  - testing
  - jasmine
  - karma
  - frontend
  - unit-testing
  - e2e
related:
  - "[[Angular Fundamentals]]"
  - "[[Services and Dependency Injection]]"
  - "[[Angular Forms]]"
  - "[[React Testing]]"
difficulty: beginner-to-intermediate
created: 2025-07-18
---

# 🧪 Angular Testing

> [!quote] "You wouldn't ship a container without inspecting its contents — don't ship components without testing them."

Testing in Angular is **first-class**. Unlike many frontend frameworks where testing is an afterthought, Angular ships with a full testing toolkit out of the box — much like how Spring Boot ships with `spring-boot-starter-test`.

> [!tip] Backend Developer Mental Model
> If you've written JUnit tests with Mockito in Spring Boot, Angular testing will feel remarkably similar. The names change, the concepts stay.

---

## Navigation

| Phase | Topic | Spring Parallel |
|-------|-------|-----------------|
| [[#Phase 1 — Testing Overview]] | Angular testing stack | Spring Boot Test |
| [[#Phase 2 — Jasmine Basics]] | Test syntax & matchers | JUnit 5 |
| [[#Phase 3 — TestBed]] | Angular's testing utility | `@SpringBootTest` |
| [[#Phase 4 — Component Testing]] | Testing UI components | Controller tests |
| [[#Phase 5 — Service Testing]] | Testing HTTP services | MockMvc / WebTestClient |
| [[#Phase 6 — Mocking Dependencies]] | Spies and mock providers | Mockito |
| [[#Phase 7 — Testing Reactive Code]] | Async & Observable testing | Reactor StepVerifier |
| [[#Phase 8 — Testing Forms]] | Form validation testing | DTO validation tests |
| [[#Phase 9 — E2E Testing]] | End-to-end with Playwright | Integration tests |
| [[#Phase 10 — Angular vs React Testing]] | Framework comparison | — |
| [[#Phase 11 — Best Practices]] | Do's and Don'ts | — |

---

## Phase 1 — Testing Overview

> [!quote] "A warehouse without quality checks ships broken goods. Code without tests ships broken features."

### Angular's Testing Stack

Angular comes with testing **built-in** — when you run `ng new my-app`, it already includes:

```
+------------------------------------------------------+
|                    ng test                            |
|  (CLI command — like `mvn test` or `gradle test`)    |
+------------------------------------------------------+
|                                                      |
|   +------------+    +-----------+    +------------+  |
|   |  Jasmine   |    |   Karma   |    |  TestBed   |  |
|   | (framework)|    | (runner)  |    | (utilities)|  |
|   |            |    |           |    |            |  |
|   | describe() |    | Launches  |    | Configures |  |
|   | it()       |    | browser   |    | test       |  |
|   | expect()   |    | Runs specs|    | modules    |  |
|   +------------+    +-----------+    +------------+  |
|                                                      |
+------------------------------------------------------+
```

### The Rosetta Stone: Java Testing → Angular Testing

| Java / Spring Boot | Angular | Role |
|---|---|---|
| JUnit 5 | Jasmine | Test framework (describe, it, expect) |
| `@SpringBootTest` | `TestBed.configureTestingModule()` | Sets up application context for tests |
| Mockito | Jasmine spies / `jasmine.createSpyObj` | Mocking dependencies |
| `mvn test` / `gradle test` | `ng test` | CLI command to run tests |
| `@MockBean` | `providers: [{ provide: Svc, useValue: mock }]` | Injecting mocks into DI container |
| `@BeforeEach` | `beforeEach()` | Per-test setup |
| `@Nested` | Nested `describe()` | Grouping related tests |
| `MockMvc` | `HttpTestingController` | Mock HTTP layer |
| `assertEquals()` | `expect().toBe()` | Assertions |
| Hamcrest matchers | Jasmine matchers | Fluent assertion API |

> [!info] Logistics Analogy
> Think of **Jasmine** as the inspection checklist, **Karma** as the warehouse inspector who walks through the facility running checks, and **TestBed** as the staging area where you set up mock shipments to inspect.

### Running Tests

```bash
# Run all tests (opens Karma in browser with live reload)
ng test

# Run tests once (CI mode — like `mvn test` in pipeline)
ng test --watch=false --browsers=ChromeHeadless

# Run tests for a specific file pattern
ng test --include='**/shipment*.spec.ts'
```

> [!note] File Convention
> Angular test files live **next to** the source file:
> ```
> src/app/shipment/
> ├── shipment.component.ts          ← Source
> ├── shipment.component.spec.ts     ← Test (*.spec.ts)
> ├── shipment.service.ts            ← Source
> └── shipment.service.spec.ts       ← Test
> ```
> This is like putting `ShipmentServiceTest.java` right next to `ShipmentService.java` — except Angular enforces this convention.

---

## Phase 2 — Jasmine Basics

> [!quote] "Before you inspect a shipment, you need to know how to fill out the inspection form."

### Test Structure: `describe` → `it` → `expect`

```typescript
// ========================================
// Jasmine structure → JUnit 5 parallel
// ========================================

// describe() = Test class / @Nested group
describe('ShipmentCalculator', () => {

  // beforeEach() = @BeforeEach
  let calculator: ShipmentCalculator;

  beforeEach(() => {
    calculator = new ShipmentCalculator();  // Fresh instance per test
  });

  // afterEach() = @AfterEach
  afterEach(() => {
    // Cleanup if needed
  });

  // it() = @Test method
  it('should calculate weight-based cost', () => {
    const cost = calculator.calculateCost(100, 'kg');

    // expect().toBe() = assertEquals()
    expect(cost).toBe(250);
  });

  // Nested describe = @Nested class in JUnit
  describe('when shipment is overweight', () => {
    it('should apply surcharge', () => {
      const cost = calculator.calculateCost(5000, 'kg');
      expect(cost).toBeGreaterThan(10000);
    });
  });

  // xit() = @Disabled — skips this test
  xit('should handle international rates', () => {
    // TODO: implement later
  });

  // fit() = focus — runs ONLY this test (like running single test in IDE)
  // fit('should ...', () => { ... });
  // ⚠️ Never commit fit() — it silently skips all other tests!
});
```

### Side-by-Side: JUnit 5 vs Jasmine

```
┌─────────────────────────────────┬─────────────────────────────────┐
│        JUnit 5 (Java)           │        Jasmine (TypeScript)     │
├─────────────────────────────────┼─────────────────────────────────┤
│ @Nested                         │ describe()                      │
│ class ShipmentCalcTest {        │ describe('ShipmentCalc', () => {│
│                                 │                                 │
│   @BeforeEach                   │   beforeEach(() => {            │
│   void setUp() { ... }         │     ...                         │
│                                 │   });                           │
│   @Test                         │                                 │
│   void shouldCalcCost() {       │   it('should calc cost', () => {│
│     assertEquals(250, cost);    │     expect(cost).toBe(250);     │
│   }                             │   });                           │
│                                 │                                 │
│   @Disabled                     │   xit('disabled test', () => {  │
│   void disabledTest() { ... }  │     ...                         │
│ }                               │   });                           │
│                                 │ });                             │
└─────────────────────────────────┴─────────────────────────────────┘
```

### Jasmine Matchers (Assertion Methods)

| Jasmine Matcher | JUnit Equivalent | Example |
|---|---|---|
| `toBe(value)` | `assertEquals(value, actual)` | `expect(status).toBe('delivered')` |
| `toEqual(obj)` | `assertEquals` (deep equality) | `expect(shipment).toEqual({id: '1', status: 'pending'})` |
| `toBeTruthy()` | `assertTrue()` | `expect(component).toBeTruthy()` |
| `toBeFalsy()` | `assertFalse()` | `expect(error).toBeFalsy()` |
| `toBeNull()` | `assertNull()` | `expect(result).toBeNull()` |
| `toContain(item)` | `assertTrue(list.contains(item))` | `expect(statuses).toContain('shipped')` |
| `toThrow()` | `assertThrows()` | `expect(() => parse(bad)).toThrow()` |
| `toHaveBeenCalled()` | `verify(mock).method()` | `expect(spy).toHaveBeenCalled()` |
| `toHaveBeenCalledWith(arg)` | `verify(mock).method(arg)` | `expect(spy).toHaveBeenCalledWith('SHIP-001')` |
| `toHaveBeenCalledTimes(n)` | `verify(mock, times(n))` | `expect(spy).toHaveBeenCalledTimes(1)` |
| `toBeGreaterThan(n)` | `assertTrue(actual > n)` | `expect(weight).toBeGreaterThan(0)` |
| `toMatch(regex)` | Pattern matching | `expect(trackingId).toMatch(/^SHIP-\d+$/)` |

> [!tip] Remember
> `toBe` checks **reference equality** (like `==` in Java for primitives).
> `toEqual` checks **deep equality** (like comparing object fields recursively).
> When in doubt, use `toEqual` for objects and arrays.

---

## Phase 3 — TestBed

> [!quote] "TestBed is your staging warehouse — you set up a controlled environment to test components in isolation."

### What is TestBed?

`TestBed` is Angular's **testing utility** that creates a mini Angular module for your tests. It's the equivalent of `@SpringBootTest` — it sets up the dependency injection container, compiles templates, and wires everything together.

```
┌──────────────────────────────────────────────┐
│              TestBed (Test Module)            │
│                                              │
│  ┌─────────────┐  ┌──────────────────────┐   │
│  │  Component   │  │    Mock Providers    │   │
│  │  Under Test  │  │                      │   │
│  │             ├──►│ ShipmentService mock │   │
│  │  Shipment    │  │ Router mock          │   │
│  │  CardComponent│ │ AuthService mock     │   │
│  └─────────────┘  └──────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │          Test Assertions             │    │
│  │  expect(component).toBeTruthy()      │    │
│  │  expect(element.textContent)...      │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

### TestBed Setup Pattern

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ShipmentCardComponent } from './shipment-card.component';

// ================================================
// This is the standard TestBed pattern.
// Think of it as your @SpringBootTest setup.
// ================================================

describe('ShipmentCardComponent', () => {
  let component: ShipmentCardComponent;
  let fixture: ComponentFixture<ShipmentCardComponent>;

  // beforeEach(async) → like @BeforeEach but async because
  // Angular needs to compile templates (HTML → JS)
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      // For standalone components — just import the component
      imports: [ShipmentCardComponent]

      // For module-based components (older style):
      // declarations: [ShipmentCardComponent],
      // imports: [CommonModule]
    }).compileComponents();  // Compiles template + CSS

    // Create component instance — like `new ShipmentCardComponent()`
    // but with DI wired up
    fixture = TestBed.createComponent(ShipmentCardComponent);
    component = fixture.componentInstance;
  });

  // The "smoke test" — does the component even create?
  // Like testing your Spring Bean loads in the context
  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should display tracking ID in the template', () => {
    // Set @Input property — like calling a setter
    component.trackingId = 'SHIP-001';

    // Trigger change detection — Angular doesn't auto-detect in tests!
    // This is like calling the template rendering manually
    fixture.detectChanges();

    // Query the rendered DOM — like parsing HTML response in MockMvc
    const element = fixture.nativeElement.querySelector('.tracking-id');
    expect(element.textContent).toContain('SHIP-001');
  });
});
```

### Key TestBed Concepts

| Concept | What It Does | Spring Parallel |
|---|---|---|
| `TestBed.configureTestingModule()` | Creates test Angular module | `@SpringBootTest` context |
| `fixture` | Wrapper around component + its DOM | — |
| `fixture.componentInstance` | The actual component object | The bean under test |
| `fixture.nativeElement` | The rendered HTML DOM | Response body in MockMvc |
| `fixture.detectChanges()` | Triggers Angular change detection | — (Spring has no UI) |
| `fixture.debugElement` | Debug wrapper with query utilities | — |
| `TestBed.inject(Service)` | Gets service from DI container | `@Autowired` in test |

> [!warning] Don't Forget `fixture.detectChanges()`!
> Angular does **not** automatically run change detection in tests. If you set an `@Input` or change component state, you MUST call `fixture.detectChanges()` before checking the DOM. This is the #1 gotcha for Angular testing beginners.

---

## Phase 4 — Component Testing

> [!quote] "Testing a component is like inspecting a shipping label — verify it shows the right info and responds correctly when handled."

### Testing @Input Properties

```typescript
describe('ShipmentCardComponent', () => {
  let component: ShipmentCardComponent;
  let fixture: ComponentFixture<ShipmentCardComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ShipmentCardComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(ShipmentCardComponent);
    component = fixture.componentInstance;
  });

  it('should render shipment status with correct CSS class', () => {
    // Arrange — set inputs (like calling setters in a POJO)
    component.shipment = {
      id: '1',
      trackingId: 'SHIP-001',
      status: 'in-transit',
      origin: 'Sydney',
      destination: 'Melbourne'
    };

    // Act — trigger rendering
    fixture.detectChanges();

    // Assert — check rendered output
    const badge = fixture.nativeElement.querySelector('.status-badge');
    expect(badge.textContent.trim()).toBe('In Transit');
    expect(badge.classList).toContain('status-in-transit');
  });

  it('should show origin → destination route', () => {
    component.shipment = {
      id: '1', trackingId: 'SHIP-001', status: 'pending',
      origin: 'Sydney', destination: 'Melbourne'
    };
    fixture.detectChanges();

    const route = fixture.nativeElement.querySelector('.route');
    expect(route.textContent).toContain('Sydney');
    expect(route.textContent).toContain('Melbourne');
  });
});
```

### Testing @Output Events

```typescript
// @Output is like a callback / event listener
// In Java terms: it's like an ApplicationEventPublisher

it('should emit statusChanged when deliver button is clicked', () => {
  component.shipment = {
    id: '1', trackingId: 'SHIP-001', status: 'in-transit',
    origin: 'Sydney', destination: 'Melbourne'
  };
  fixture.detectChanges();

  // spyOn = like Mockito.verify() — watches for calls
  spyOn(component.statusChanged, 'emit');

  // Simulate user click — like calling an endpoint
  const button = fixture.nativeElement.querySelector('.deliver-btn');
  button.click();

  // Verify the event was emitted with correct payload
  // Like: verify(eventPublisher).publishEvent(new StatusChangedEvent('delivered'))
  expect(component.statusChanged.emit).toHaveBeenCalledWith('delivered');
});
```

### Testing User Interactions

```typescript
it('should filter shipments when user types in search box', () => {
  component.shipments = [
    { id: '1', trackingId: 'SHIP-001', status: 'pending' },
    { id: '2', trackingId: 'SHIP-002', status: 'delivered' },
    { id: '3', trackingId: 'SHIP-003', status: 'pending' }
  ];
  fixture.detectChanges();

  // Simulate typing in an input — like sending form data
  const input = fixture.nativeElement.querySelector('.search-input');
  input.value = 'SHIP-001';
  input.dispatchEvent(new Event('input'));  // Fire the DOM event
  fixture.detectChanges();

  const rows = fixture.nativeElement.querySelectorAll('.shipment-row');
  expect(rows.length).toBe(1);
  expect(rows[0].textContent).toContain('SHIP-001');
});
```

### Testing with Mock Child Components

```typescript
// When your component uses child components, you can mock them
// to isolate the unit under test — like @MockBean for nested beans

// Create a mock/stub component
@Component({
  selector: 'app-tracking-map',  // Same selector as real component
  template: '<div class="mock-map"></div>',
  standalone: true
})
class MockTrackingMapComponent {
  @Input() coordinates: any;  // Same @Input interface
}

describe('ShipmentDetailComponent', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ShipmentDetailComponent]
    })
    // Override the real child component with the mock
    .overrideComponent(ShipmentDetailComponent, {
      remove: { imports: [TrackingMapComponent] },
      add: { imports: [MockTrackingMapComponent] }
    })
    .compileComponents();
  });
});
```

### `fixture.detectChanges()` Explained

```
┌──────────────────────────────────────────────────────┐
│               Change Detection Flow                   │
│                                                      │
│  1. You set: component.status = 'delivered'          │
│                     │                                │
│                     ▼                                │
│  2. Call: fixture.detectChanges()                    │
│                     │                                │
│                     ▼                                │
│  3. Angular processes bindings:                      │
│     {{ status }} → 'delivered'                       │
│     [class.active]="isActive" → true                 │
│     *ngIf="status === 'delivered'" → show element    │
│                     │                                │
│                     ▼                                │
│  4. DOM is updated — now you can query it            │
│     fixture.nativeElement.querySelector(...)          │
└──────────────────────────────────────────────────────┘
```

> [!warning] Common Mistake
> ❌ Setting `component.name = 'test'` then immediately querying the DOM
> ✅ Setting `component.name = 'test'` → `fixture.detectChanges()` → then querying

---

## Phase 5 — Service Testing

> [!quote] "Testing a service is like verifying your logistics API — mock the network, assert the business logic."

### Testing HTTP Services with `HttpTestingController`

This is the Angular equivalent of Spring's `MockMvc` — it intercepts HTTP calls and lets you return mock responses.

```typescript
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { ShipmentService } from './shipment.service';
import { Shipment } from '../models/shipment.model';

describe('ShipmentService', () => {
  let service: ShipmentService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      // HttpClientTestingModule replaces HttpClient with a mock
      // Like @MockBean for RestTemplate/WebClient
      imports: [HttpClientTestingModule],
      providers: [ShipmentService]
    });

    service = TestBed.inject(ShipmentService);        // Get service from DI
    httpMock = TestBed.inject(HttpTestingController);  // Get HTTP mock controller
  });

  afterEach(() => {
    // CRITICAL: Verify no unexpected HTTP calls were made
    // Like verifyNoMoreInteractions(mockRestTemplate) in Mockito
    httpMock.verify();
  });

  it('should fetch all shipments via GET', () => {
    // Arrange — mock response data
    const mockShipments: Shipment[] = [
      { id: '1', trackingId: 'SHIP-001', status: 'pending', origin: 'Sydney', destination: 'Melbourne' },
      { id: '2', trackingId: 'SHIP-002', status: 'delivered', origin: 'Brisbane', destination: 'Perth' }
    ];

    // Act — call the service method (subscribes to Observable)
    service.getShipments().subscribe(data => {
      // Assert — verify response
      expect(data.length).toBe(2);
      expect(data[0].trackingId).toBe('SHIP-001');
      expect(data[1].status).toBe('delivered');
    });

    // Expect exactly one GET request to this URL
    const req = httpMock.expectOne('/api/shipments');
    expect(req.request.method).toBe('GET');

    // Flush (respond with) mock data — this triggers the subscribe callback
    req.flush(mockShipments);
  });

  it('should create a shipment via POST', () => {
    const newShipment: Partial<Shipment> = {
      trackingId: 'SHIP-003',
      origin: 'Adelaide',
      destination: 'Darwin'
    };

    const createdShipment: Shipment = {
      id: '3', trackingId: 'SHIP-003', status: 'pending',
      origin: 'Adelaide', destination: 'Darwin'
    };

    service.createShipment(newShipment).subscribe(data => {
      expect(data.id).toBe('3');
      expect(data.status).toBe('pending');
    });

    const req = httpMock.expectOne('/api/shipments');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newShipment);  // Verify request body
    req.flush(createdShipment);
  });

  it('should handle HTTP errors gracefully', () => {
    service.getShipments().subscribe({
      next: () => fail('should have failed with 500'),
      error: (error) => {
        expect(error.status).toBe(500);
      }
    });

    const req = httpMock.expectOne('/api/shipments');
    // Simulate server error — like mockMvc returning 500
    req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
  });
});
```

### Spring MockMvc vs Angular HttpTestingController

```
┌─────────────────────────────────┬──────────────────────────────────────┐
│      Spring MockMvc (Java)      │   Angular HttpTestingController (TS) │
├─────────────────────────────────┼──────────────────────────────────────┤
│ mockMvc.perform(                │ service.getShipments().subscribe()   │
│   get("/api/shipments")         │                                      │
│ )                               │ const req = httpMock                 │
│                                 │   .expectOne('/api/shipments');       │
│ .andExpect(status().isOk())     │ expect(req.request.method)           │
│                                 │   .toBe('GET');                      │
│ .andExpect(jsonPath("$[0].id")  │                                      │
│   .value("1"))                  │ req.flush(mockData);                 │
│                                 │ // assertions in subscribe callback  │
│ @MockBean                       │ providers: [{ provide: Svc,          │
│ RestTemplate restTemplate;      │   useValue: mockSvc }]               │
│                                 │                                      │
│ verify(restTemplate)            │ httpMock.verify()                    │
│   .getForObject(...)            │ // ensures no unmatched requests     │
└─────────────────────────────────┴──────────────────────────────────────┘
```

| Spring MockMvc | Angular HttpTestingController | Purpose |
|---|---|---|
| `mockMvc.perform(get("/api/..."))` | `httpMock.expectOne(url)` | Intercept HTTP request |
| `.andExpect(status().isOk())` | `expect(req.request.method).toBe('GET')` | Verify request details |
| `.andReturn().getResponse()` | `req.flush(data)` | Return mock response |
| `@MockBean RestTemplate` | `HttpClientTestingModule` | Replace HTTP client |
| `verifyNoMoreInteractions()` | `httpMock.verify()` | No unexpected calls |

---

## Phase 6 — Mocking Dependencies

> [!quote] "In logistics, you don't need a real ship to test the booking system — you need a mock ship that behaves predictably."

### `jasmine.createSpyObj` — The Mockito of Angular

```typescript
// ============================================
// Mockito (Java)
// ============================================
// ShipmentService mockService = mock(ShipmentService.class);
// when(mockService.getShipments()).thenReturn(List.of(shipment));

// ============================================
// Jasmine (TypeScript) — EXACT same concept
// ============================================
const mockShipmentService = jasmine.createSpyObj('ShipmentService', [
  'getShipments',
  'getById',
  'createShipment',
  'updateStatus'
]);

// .and.returnValue() = when().thenReturn()
mockShipmentService.getShipments.and.returnValue(of([
  { id: '1', trackingId: 'SHIP-001', status: 'pending' }
]));

mockShipmentService.getById.and.returnValue(of({
  id: '1', trackingId: 'SHIP-001', status: 'in-transit'
}));
```

### Injecting Mocks into TestBed

```typescript
describe('ShipmentListComponent', () => {
  let component: ShipmentListComponent;
  let fixture: ComponentFixture<ShipmentListComponent>;
  let mockShipmentService: jasmine.SpyObj<ShipmentService>;

  beforeEach(async () => {
    // Create the spy object — like Mockito.mock()
    mockShipmentService = jasmine.createSpyObj('ShipmentService', [
      'getShipments', 'deleteShipment'
    ]);
    mockShipmentService.getShipments.and.returnValue(of([]));

    await TestBed.configureTestingModule({
      imports: [ShipmentListComponent],
      providers: [
        // Override the real service with our mock
        // This is EXACTLY like @MockBean in Spring:
        //   @MockBean ShipmentService shipmentService;
        { provide: ShipmentService, useValue: mockShipmentService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ShipmentListComponent);
    component = fixture.componentInstance;
  });

  it('should load shipments on init', () => {
    const mockData = [
      { id: '1', trackingId: 'SHIP-001', status: 'pending' }
    ];
    mockShipmentService.getShipments.and.returnValue(of(mockData));

    fixture.detectChanges();  // Triggers ngOnInit

    expect(mockShipmentService.getShipments).toHaveBeenCalled();
    expect(component.shipments.length).toBe(1);
  });

  it('should call deleteShipment when remove button clicked', () => {
    mockShipmentService.getShipments.and.returnValue(of([
      { id: '1', trackingId: 'SHIP-001', status: 'pending' }
    ]));
    mockShipmentService.deleteShipment.and.returnValue(of(void 0));

    fixture.detectChanges();

    const deleteBtn = fixture.nativeElement.querySelector('.delete-btn');
    deleteBtn.click();

    // verify(mockService).deleteShipment("1")  — Mockito equivalent
    expect(mockShipmentService.deleteShipment).toHaveBeenCalledWith('1');
  });
});
```

### Mocking Comparison Table

| Mockito (Java) | Jasmine (Angular) | Purpose |
|---|---|---|
| `mock(Service.class)` | `jasmine.createSpyObj('Svc', [...])` | Create mock object |
| `when(m.method()).thenReturn(val)` | `spy.method.and.returnValue(val)` | Stub return value |
| `verify(m).method()` | `expect(spy.method).toHaveBeenCalled()` | Verify method called |
| `verify(m, times(2))` | `expect(spy).toHaveBeenCalledTimes(2)` | Verify call count |
| `verify(m).method(eq(arg))` | `expect(spy).toHaveBeenCalledWith(arg)` | Verify with args |
| `doThrow().when(m).method()` | `spy.method.and.throwError('err')` | Stub to throw |
| `@MockBean` | `{ provide: Svc, useValue: spy }` | Inject into DI |
| `@SpyBean` | `spyOn(realService, 'method')` | Partial mock |

### Partial Mocks with `spyOn`

```typescript
// Like @SpyBean — spy on a real service, override specific methods
it('should use real calculation but mock the API call', () => {
  const realService = TestBed.inject(ShipmentService);

  // Only mock this one method — rest stays real
  // Like Mockito.spy() + doReturn()
  spyOn(realService, 'fetchRates').and.returnValue(of({ rate: 15.5 }));

  const cost = realService.calculateShippingCost(100);
  expect(cost).toBe(1550);
  expect(realService.fetchRates).toHaveBeenCalled();
});
```

---

## Phase 7 — Testing Reactive Code

> [!quote] "Async code is like tracking a shipment — you don't know when the update arrives, but you need to handle it when it does."

### Testing Observables Directly

```typescript
it('should return filtered shipments', (done) => {
  // The `done` callback tells Jasmine: "this test is async —
  // don't finish until I call done()"
  // Like using CountDownLatch in Java async tests

  service.getShipmentsByStatus('pending').subscribe(shipments => {
    expect(shipments.length).toBe(2);
    expect(shipments.every(s => s.status === 'pending')).toBeTrue();
    done();  // Signal test completion
  });

  const req = httpMock.expectOne('/api/shipments?status=pending');
  req.flush(mockPendingShipments);
});
```

### `fakeAsync` + `tick` — Time Travel for Tests

```typescript
import { fakeAsync, tick } from '@angular/core/testing';

// fakeAsync creates a synthetic time zone where YOU control time
// Like Mockito's ArgumentCaptor but for time itself

it('should debounce search input by 300ms', fakeAsync(() => {
  // Arrange
  mockShipmentService.search.and.returnValue(of([]));

  // Act — user types in search box
  component.searchControl.setValue('SHIP');

  // At this point, debounceTime(300) is waiting...
  // The search should NOT have been called yet
  expect(mockShipmentService.search).not.toHaveBeenCalled();

  // Fast-forward time by 300ms
  tick(300);
  fixture.detectChanges();

  // NOW the debounce has elapsed — search should fire
  expect(mockShipmentService.search).toHaveBeenCalledWith('SHIP');
}));

it('should auto-refresh shipment status every 30 seconds', fakeAsync(() => {
  mockShipmentService.getStatus.and.returnValue(of('in-transit'));

  component.startAutoRefresh();

  // Fast-forward 30 seconds
  tick(30_000);
  expect(mockShipmentService.getStatus).toHaveBeenCalledTimes(1);

  // Fast-forward another 30 seconds
  tick(30_000);
  expect(mockShipmentService.getStatus).toHaveBeenCalledTimes(2);

  component.stopAutoRefresh();  // Clean up interval

  // Flush any remaining timers to avoid "pending timers" error
  // discardPeriodicTasks();
}));
```

### `waitForAsync` — For Real Async Operations

```typescript
import { waitForAsync } from '@angular/core/testing';

// waitForAsync waits for all Promises and Observables to settle
// Use when you can't easily control timing

it('should load data after component init', waitForAsync(() => {
  mockShipmentService.getShipments.and.returnValue(of(mockData));

  fixture.detectChanges();  // Triggers ngOnInit

  // waitForAsync ensures the Observable has completed
  fixture.whenStable().then(() => {
    expect(component.shipments.length).toBe(3);
  });
}));
```

### Async Testing Comparison

```
┌──────────────────────────────────────────────────────┐
│              When to Use What?                        │
│                                                      │
│  fakeAsync + tick                                    │
│  ├── debounceTime, delay, interval                   │
│  ├── setTimeout, setInterval                         │
│  └── You need PRECISE time control                   │
│                                                      │
│  waitForAsync                                        │
│  ├── Promise-based operations                        │
│  ├── Complex async chains                            │
│  └── When timing doesn't matter, just completion     │
│                                                      │
│  done callback                                       │
│  ├── Simple Observable assertions                    │
│  ├── One-shot async operations                       │
│  └── When you need manual completion control         │
└──────────────────────────────────────────────────────┘
```

---

## Phase 8 — Testing Forms

> [!quote] "Validating a shipment form is like validating a bill of lading — every field matters."

See also: [[Angular Forms]]

### Testing Reactive Forms (Recommended)

Reactive forms are **easier to test** because the form model lives in TypeScript, not the template. You test the model directly — no DOM needed!

```typescript
describe('ShipmentFormComponent', () => {
  let component: ShipmentFormComponent;
  let fixture: ComponentFixture<ShipmentFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ShipmentFormComponent, ReactiveFormsModule]
    }).compileComponents();

    fixture = TestBed.createComponent(ShipmentFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create form with required fields', () => {
    // Test the form model directly — no DOM interaction needed!
    // Like testing a DTO's validation annotations
    expect(component.shipmentForm).toBeTruthy();
    expect(component.shipmentForm.get('trackingId')).toBeTruthy();
    expect(component.shipmentForm.get('origin')).toBeTruthy();
    expect(component.shipmentForm.get('destination')).toBeTruthy();
    expect(component.shipmentForm.get('weight')).toBeTruthy();
  });

  it('should be invalid when required fields are empty', () => {
    // Like: assertFalse(validator.validate(emptyDto).isEmpty())
    expect(component.shipmentForm.valid).toBeFalse();
  });

  it('should be valid when all required fields are filled', () => {
    component.shipmentForm.patchValue({
      trackingId: 'SHIP-001',
      origin: 'Sydney',
      destination: 'Melbourne',
      weight: 150
    });

    expect(component.shipmentForm.valid).toBeTrue();
  });

  it('should validate weight is positive', () => {
    const weightControl = component.shipmentForm.get('weight')!;

    weightControl.setValue(-5);
    expect(weightControl.hasError('min')).toBeTrue();

    weightControl.setValue(100);
    expect(weightControl.hasError('min')).toBeFalse();
  });

  it('should show validation error in template', () => {
    const trackingIdControl = component.shipmentForm.get('trackingId')!;

    // Mark as touched (user interacted then left)
    trackingIdControl.markAsTouched();
    trackingIdControl.setValue('');
    fixture.detectChanges();

    const errorEl = fixture.nativeElement.querySelector('.error-message');
    expect(errorEl.textContent).toContain('Tracking ID is required');
  });
});
```

### Testing Template-Driven Forms

```typescript
// Template-driven forms require more DOM interaction
// because the form model lives in the template

it('should bind input value to model', async () => {
  fixture.detectChanges();
  await fixture.whenStable();  // Wait for ngModel to initialize

  const input = fixture.nativeElement.querySelector('#origin-input');
  input.value = 'Brisbane';
  input.dispatchEvent(new Event('input'));

  fixture.detectChanges();
  await fixture.whenStable();

  expect(component.shipment.origin).toBe('Brisbane');
});
```

> [!tip] Prefer Reactive Forms for Testability
> Reactive forms are **significantly easier** to test because the form model is a plain TypeScript object. You don't need to interact with the DOM at all for validation logic — just like testing a Java DTO with Bean Validation annotations.

---

## Phase 9 — E2E Testing

> [!quote] "Unit tests verify each cog in the machine. E2E tests verify the whole conveyor belt moves packages from start to finish."

### What is E2E Testing?

End-to-end tests run your **entire application** in a real browser and simulate real user interactions. It's the frontend equivalent of Spring's `@SpringBootTest` with a real embedded server + REST client.

```
┌─────────────────────────────────────────────────────────┐
│                    E2E Test Flow                         │
│                                                         │
│  Playwright/Cypress        Angular App        Mock API   │
│  ┌──────────┐            ┌──────────┐      ┌─────────┐ │
│  │ Open URL │───────────►│ Renders  │─────►│ /api/.. │ │
│  │ Click    │            │ in real  │      │ returns  │ │
│  │ Type     │            │ browser  │      │ mock     │ │
│  │ Assert   │◄───────────│          │◄─────│ data     │ │
│  └──────────┘            └──────────┘      └─────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Modern E2E: Playwright (Recommended)

```bash
# Add Playwright to your Angular project
ng add @playwright/test
# or
npm init playwright@latest
```

```typescript
// e2e/shipment-list.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Shipment List Page', () => {

  test('should display list of shipments', async ({ page }) => {
    await page.goto('http://localhost:4200/shipments');

    // Wait for data to load
    await page.waitForSelector('.shipment-row');

    const rows = await page.locator('.shipment-row').count();
    expect(rows).toBeGreaterThan(0);
  });

  test('should navigate to detail when clicking a shipment', async ({ page }) => {
    await page.goto('http://localhost:4200/shipments');
    await page.click('.shipment-row:first-child');

    await expect(page).toHaveURL(/\/shipments\/\d+/);
    await expect(page.locator('.shipment-detail')).toBeVisible();
  });

  test('should filter shipments by status', async ({ page }) => {
    await page.goto('http://localhost:4200/shipments');

    await page.selectOption('#status-filter', 'delivered');
    await page.waitForTimeout(500);  // Wait for filter to apply

    const statuses = await page.locator('.status-badge').allTextContents();
    statuses.forEach(status => {
      expect(status.trim().toLowerCase()).toBe('delivered');
    });
  });
});
```

### E2E Testing Tools Comparison

| Tool | Pros | Cons |
|---|---|---|
| **Playwright** | Fast, cross-browser, auto-wait, great DX | Newer, smaller community |
| **Cypress** | Excellent debug UI, time-travel | Single-tab only, no cross-browser (free) |
| **Protractor** | ❌ **Deprecated** — do NOT use | — |

> [!warning] Protractor is Dead
> Angular's original E2E tool Protractor was deprecated in Angular 12. Use **Playwright** or **Cypress** for new projects.

### When to Use What

```
Unit Tests (Jasmine + Karma)     ← Fast, isolated, 80% of tests
  └── Component, Service, Pipe tests

Integration Tests (TestBed)      ← Component + template + DI
  └── Component with real child components

E2E Tests (Playwright/Cypress)   ← Slow, full app, 10-20% of tests
  └── Critical user journeys only
```

---

## Phase 10 — Angular vs React Testing

> [!info] Cross-Reference
> See [[React Testing]] for the React equivalent of this note.

| Aspect | Angular | React |
|---|---|---|
| **Test Framework** | Jasmine (built-in) | Jest (standard) |
| **Test Runner** | Karma (browser-based) | Jest (Node-based) |
| **Component Setup** | `TestBed.configureTestingModule()` | `render()` from React Testing Library |
| **Trigger Render** | `fixture.detectChanges()` | Automatic on state change |
| **Query DOM** | `fixture.nativeElement.querySelector()` | `screen.getByText()`, `screen.getByRole()` |
| **Mock HTTP** | `HttpTestingController` | MSW (Mock Service Worker) / `jest.mock()` |
| **Mock Dependencies** | `jasmine.createSpyObj` + providers | `jest.fn()` + module mocking |
| **Async Testing** | `fakeAsync` / `tick` | `waitFor()` / `act()` |
| **Form Testing** | Reactive form model or DOM | Controlled component state |
| **E2E** | Playwright / Cypress | Playwright / Cypress |
| **Philosophy** | Test the component as a unit | Test as the user sees it |

### Key Difference: Testing Philosophy

```
Angular Testing                          React Testing (RTL)
┌────────────────────────┐              ┌────────────────────────┐
│ Tests component class  │              │ Tests component output  │
│ + template rendering   │              │ from user perspective   │
│                        │              │                         │
│ component.name = 'X';  │              │ render(<Component />);  │
│ fixture.detectChanges();│              │ screen.getByText('X');  │
│ el.querySelector(...)  │              │                         │
│                        │              │ // No direct component  │
│ // Access internals    │              │ // access — only DOM    │
│ // freely              │              │ // queries              │
└────────────────────────┘              └────────────────────────┘
```

> [!note] Angular Trend
> Angular is moving toward a more "user-centric" testing approach with `@angular/cdk/testing` harnesses and the emerging `@testing-library/angular` package — making it more similar to React Testing Library's philosophy.

---

## Phase 11 — Best Practices

> [!quote] "A well-tested codebase is like a well-organized warehouse — everything is inspected, labeled, and tracked."

### ✅ Do's

| Practice | Why | Example |
|---|---|---|
| ✅ Test behavior, not implementation | Tests survive refactoring | Test "displays name" not "calls renderName()" |
| ✅ One assertion per test (when reasonable) | Clear failure messages | Separate tests for each expected output |
| ✅ Use meaningful test names | Acts as documentation | `'should show error when tracking ID is invalid'` |
| ✅ Mock external dependencies | Isolation + speed | Use `jasmine.createSpyObj` for services |
| ✅ Always call `httpMock.verify()` | Catches unexpected API calls | In `afterEach()` block |
| ✅ Use `fakeAsync` for time-dependent tests | Deterministic results | `tick(300)` for debounce |
| ✅ Test edge cases | Catches bugs early | Empty lists, null inputs, error responses |
| ✅ Keep test files next to source | Easy to find | `component.spec.ts` next to `component.ts` |

### ❌ Don'ts

| Anti-Pattern | Why It's Bad | Better Approach |
|---|---|---|
| ❌ Testing private methods directly | Couples to implementation | Test through public API |
| ❌ Using `fit()` / `fdescribe()` in commits | Silently skips other tests | Use CI lint rule to catch these |
| ❌ Testing Angular framework code | Angular is already tested | Don't test `ngIf` works — test YOUR logic |
| ❌ Deep DOM structure assertions | Breaks on template changes | Use semantic queries (class names, roles) |
| ❌ Ignoring `afterEach` cleanup | Memory leaks, flaky tests | Always clean up subscriptions and mocks |
| ❌ Using `setTimeout` in tests | Non-deterministic | Use `fakeAsync` + `tick` instead |
| ❌ Huge TestBed configurations | Slow tests | Only import what's needed for the test |

### The Testing Pyramid for Angular

```
         ╱╲
        ╱  ╲         E2E Tests (Playwright)
       ╱    ╲        → Critical user journeys
      ╱──────╲       → 10-15% of tests
     ╱        ╲      → Slow but high confidence
    ╱──────────╲
   ╱            ╲    Integration Tests (TestBed)
  ╱              ╲   → Component + children + services
 ╱────────────────╲  → 20-30% of tests
╱                  ╲
╱────────────────────╲  Unit Tests (Jasmine)
╱                      ╲ → Services, pipes, utils
╱────────────────────────╲→ 60-70% of tests
                           → Fast, isolated, reliable
```

### Test Organization Checklist

```
src/app/shipment/
├── shipment.component.ts
├── shipment.component.spec.ts          ← Component tests
├── shipment.component.html
├── shipment.service.ts
├── shipment.service.spec.ts            ← Service tests
├── shipment.pipe.ts
├── shipment.pipe.spec.ts               ← Pipe tests
├── models/
│   └── shipment.model.ts
└── guards/
    ├── shipment.guard.ts
    └── shipment.guard.spec.ts          ← Guard tests
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                 Angular Testing Cheat Sheet                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RUN TESTS:     ng test                                     │
│  CI MODE:       ng test --watch=false --browsers=Headless   │
│                                                             │
│  TEST STRUCTURE:                                            │
│    describe('Suite', () => {                                │
│      beforeEach(() => { /* setup */ });                     │
│      it('should do X', () => { expect(a).toBe(b); });      │
│    });                                                      │
│                                                             │
│  TESTBED:                                                   │
│    TestBed.configureTestingModule({ imports: [Comp] })      │
│    fixture = TestBed.createComponent(Comp)                  │
│    component = fixture.componentInstance                    │
│    fixture.detectChanges()  // ← DON'T FORGET!             │
│                                                             │
│  MOCK SERVICE:                                              │
│    const mock = jasmine.createSpyObj('Svc', ['method'])     │
│    mock.method.and.returnValue(of(data))                    │
│    providers: [{ provide: Svc, useValue: mock }]            │
│                                                             │
│  HTTP TESTING:                                              │
│    imports: [HttpClientTestingModule]                        │
│    httpMock = TestBed.inject(HttpTestingController)          │
│    const req = httpMock.expectOne(url)                      │
│    req.flush(mockData)                                      │
│    httpMock.verify()  // ← in afterEach!                    │
│                                                             │
│  ASYNC:                                                     │
│    fakeAsync(() => { tick(ms); })  // Time control          │
│    waitForAsync(() => { })         // Auto-wait             │
│    (done) => { ...; done(); }      // Manual signal         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Notes

- [[Angular Fundamentals]] — Core Angular concepts
- [[Services and Dependency Injection]] — How DI works (relevant for mock injection)
- [[Angular Forms]] — Reactive & template-driven forms (tested in Phase 8)
- [[React Testing]] — Compare with React's testing approach
- [[RxJS and Observables]] — Understanding Observables for Phase 7

---

> [!success] Key Takeaway
> If you can write JUnit tests with Mockito, you can write Angular tests with Jasmine.
> The patterns are nearly identical — `describe` ≈ test class, `it` ≈ `@Test`, `createSpyObj` ≈ `mock()`, `TestBed` ≈ `@SpringBootTest`. The biggest Angular-specific concept is `fixture.detectChanges()` — always remember to trigger change detection after modifying component state!
