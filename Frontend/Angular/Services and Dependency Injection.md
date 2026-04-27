---
title: Services and Dependency Injection
phase: fundamentals
topic: angular-services-di
tags:
  - angular
  - services
  - dependency-injection
  - httpclient
  - providers
  - interceptors
  - frontend
  - spring-boot-parallel
difficulty: beginner-to-intermediate
prerequisites:
  - "[[Angular Fundamentals]]"
  - "[[Components and Templates]]"
created: 2025-07-17
updated: 2025-07-17
---

# Services and Dependency Injection

> *"Don't put logic in your controllers. Put it in services."*
> — Every Spring Boot mentor you've ever had. Angular says the exact same thing.

---

## Why This Matters

If you've built Spring Boot microservices, you already know the pattern:
**Controllers handle HTTP, Services handle logic, Repositories handle data.**

Angular follows the **exact same separation of concerns**:
**Components handle UI, Services handle logic, HttpClient handles API calls.**

Angular's DI system is the **most Spring-like feature** in any frontend framework.
React doesn't have it. Vue doesn't have it. Angular does — and it will feel like home.

---

## Mental Model — The Logistics Warehouse Analogy

```
Think of your Angular app as a logistics warehouse:

┌─────────────────────────────────────────────────────────┐
│                    WAREHOUSE (App)                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Loading Dock │  │  Sorting Bay │  │  Dispatch    │  │
│  │  (Component)  │  │  (Component) │  │  (Component) │  │
│  │              │  │              │  │              │  │
│  │  Displays UI  │  │  Displays UI │  │  Displays UI │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                 │           │
│         └────────┬────────┴────────┬────────┘           │
│                  │                 │                     │
│         ┌───────▼───────┐ ┌───────▼───────┐            │
│         │ ShipmentService│ │ CarrierService │            │
│         │  (Service)     │ │  (Service)     │            │
│         │                │ │                │            │
│         │ Business Logic │ │ Business Logic │            │
│         │ API Calls      │ │ API Calls      │            │
│         └───────┬───────┘ └───────┬───────┘            │
│                 │                 │                     │
│                 └────────┬────────┘                     │
│                          │                              │
│                  ┌───────▼───────┐                      │
│                  │  HttpClient   │                      │
│                  │  (Built-in)   │                      │
│                  │               │                      │
│                  │ Talks to API  │                      │
│                  └───────────────┘                      │
└─────────────────────────────────────────────────────────┘

Components NEVER call APIs directly.
Services are the middlemen — just like in Spring Boot.
```

---

## Phase 1: What are Services?

> [!info] Spring Boot Parallel
> Angular `@Injectable()` = Spring `@Service`. Same concept. Same purpose.
> Classes that handle business logic, data access, API calls — NOT UI.

### The Problem Without Services

❌ **Bad — Logic inside the component (like putting SQL in a Controller):**

```typescript
// ❌ DON'T DO THIS — logic inside component
@Component({
  selector: 'app-shipment-list',
  template: `
    <div *ngFor="let s of shipments">{{ s.trackingNumber }}</div>
  `
})
export class ShipmentListComponent implements OnInit {
  shipments: Shipment[] = [];

  // HttpClient directly in component = bad separation
  constructor(private http: HttpClient) {}

  ngOnInit(): void {
    // Business logic in component — Spring devs would cringe
    this.http.get<Shipment[]>('/api/shipments')
      .pipe(
        map(shipments => shipments.filter(s => s.status === 'IN_TRANSIT')),
        catchError(err => {
          console.error('Failed to load shipments', err);
          return of([]);
        })
      )
      .subscribe(data => this.shipments = data);
  }
}
```

> [!warning] Why this is bad
> - Component knows about HTTP, URLs, error handling, filtering
> - Can't reuse the API call in another component
> - Can't unit test the logic without rendering the component
> - Same problems as putting business logic in a Spring `@Controller`

### The Solution — Extract a Service

✅ **Good — Service handles logic, Component handles UI:**

```typescript
// ✅ shipment.service.ts — handles ALL shipment logic
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })  // Singleton — like Spring @Service
export class ShipmentService {
  private apiUrl = '/api/shipments';

  // Constructor injection — EXACTLY like Spring Boot
  constructor(private http: HttpClient) {}

  getShipments(): Observable<Shipment[]> {
    return this.http.get<Shipment[]>(this.apiUrl);
  }

  getById(id: string): Observable<Shipment> {
    return this.http.get<Shipment>(`${this.apiUrl}/${id}`);
  }

  create(shipment: CreateShipmentDto): Observable<Shipment> {
    return this.http.post<Shipment>(this.apiUrl, shipment);
  }

  update(id: string, shipment: UpdateShipmentDto): Observable<Shipment> {
    return this.http.put<Shipment>(`${this.apiUrl}/${id}`, shipment);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }

  getInTransit(): Observable<Shipment[]> {
    return this.getShipments().pipe(
      map(shipments => shipments.filter(s => s.status === 'IN_TRANSIT'))
    );
  }
}
```

```typescript
// ✅ shipment-list.component.ts — only handles UI
@Component({
  selector: 'app-shipment-list',
  template: `
    <div *ngFor="let s of shipments">{{ s.trackingNumber }}</div>
  `
})
export class ShipmentListComponent implements OnInit {
  shipments: Shipment[] = [];

  // Service injected — component doesn't know about HTTP
  constructor(private shipmentService: ShipmentService) {}

  ngOnInit(): void {
    this.shipmentService.getInTransit()
      .subscribe(data => this.shipments = data);
  }
}
```

### Spring Boot Side-by-Side

```java
// Spring Boot — you already write code like this
@Service
public class ShipmentService {
    private final ShipmentRepository repository;

    // Constructor injection (recommended in Spring)
    public ShipmentService(ShipmentRepository repository) {
        this.repository = repository;
    }

    public List<Shipment> getInTransit() {
        return repository.findByStatus(ShipmentStatus.IN_TRANSIT);
    }
}

@RestController
@RequestMapping("/api/shipments")
public class ShipmentController {
    private final ShipmentService shipmentService;

    public ShipmentController(ShipmentService shipmentService) {
        this.shipmentService = shipmentService;
    }

    @GetMapping("/in-transit")
    public List<Shipment> getInTransit() {
        return shipmentService.getInTransit();  // Controller delegates to Service
    }
}
```

```
Spring Boot:    Controller  →  Service  →  Repository  →  Database
Angular:        Component   →  Service  →  HttpClient  →  Spring Boot API

Same layers. Same separation. Different runtime.
```

---

## Phase 2: Dependency Injection — Angular's Most Spring-Like Feature

> [!tip] Key Insight
> Angular has a **full DI container** — just like Spring's `ApplicationContext`.
> It resolves dependencies, manages lifecycles, and supports scopes.
> No other major frontend framework has this.

### Constructor Injection — Identical Pattern

```typescript
// Angular — constructor injection (the ONLY way)
@Component({ selector: 'app-shipment-list', template: '...' })
export class ShipmentListComponent {
  constructor(
    private shipmentService: ShipmentService,    // Injected automatically
    private carrierService: CarrierService,      // Injected automatically
    private router: Router                       // Even framework services are injected
  ) {}
}
```

```java
// Spring Boot — constructor injection (recommended way)
@RestController
public class ShipmentController {
    private final ShipmentService shipmentService;
    private final CarrierService carrierService;

    // Spring auto-injects via constructor
    public ShipmentController(
        ShipmentService shipmentService,
        CarrierService carrierService
    ) {
        this.shipmentService = shipmentService;
        this.carrierService = carrierService;
    }
}
```

> [!note] The Only Difference
> - Spring supports field injection (`@Autowired` on fields) — discouraged but works
> - Angular **only** supports constructor injection — no field injection
> - This is actually a GOOD thing — forces proper design

### The DI Comparison Table

| Concept | Spring Boot | Angular | Notes |
|---------|-------------|---------|-------|
| Service annotation | `@Service` / `@Component` | `@Injectable()` | Both mark a class as injectable |
| Constructor injection | Auto-detected or `@Autowired` | Constructor parameter with type | Identical pattern |
| Singleton scope | Default (singleton) | `providedIn: 'root'` | Both default to singleton |
| Prototype scope | `@Scope("prototype")` | Component-level `providers` | New instance per injection |
| Bean configuration | `@Configuration` + `@Bean` | `providers` array | Custom creation logic |
| Property injection | `@Value("${key}")` | `InjectionToken` + `@Inject()` | Non-class values |
| Container | `ApplicationContext` | `Injector` | The DI container itself |
| Qualifiers | `@Qualifier("name")` | `InjectionToken` | When multiple implementations exist |
| Optional deps | `@Autowired(required=false)` | `@Optional()` | Graceful absence handling |
| Profiles | `@Profile("dev")` | `environment.ts` + providers | Environment-specific beans |
| Lazy loading | `@Lazy` | Lazy-loaded modules | Deferred initialization |

### How Angular Resolves Dependencies

```
When Angular sees:  constructor(private shipmentService: ShipmentService)

It does this:
                                                              
  1. Look at the type: ShipmentService                        
  2. Search the injector tree (bottom → top):                 
                                                              
     Component Injector  →  has ShipmentService?  NO          
            ↓                                                 
     Module Injector     →  has ShipmentService?  NO          
            ↓                                                 
     Root Injector       →  has ShipmentService?  YES ✅       
            ↓                                                 
     Return the singleton instance                            

  Same concept as Spring's ApplicationContext.getBean(ShipmentService.class)
```

---

## Phase 3: Providers and Injection Tokens

### Provider Scopes — Where You Register Matters

#### 1. Root Level — Application-wide Singleton (Default)

```typescript
// Most common — available everywhere, one instance for entire app
@Injectable({ providedIn: 'root' })  // ← This is the key
export class ShipmentService {
  constructor(private http: HttpClient) {}
}
```

```
Spring equivalent:
@Service  // Default scope = singleton
public class ShipmentService { }

Both create ONE instance shared across the entire application.
```

✅ Use for: API services, auth services, state management, caching

#### 2. Component Level — New Instance Per Component

```typescript
@Component({
  selector: 'app-shipment-form',
  template: '...',
  // Each ShipmentFormComponent gets its OWN FormStateService
  providers: [FormStateService]  // ← Component-level provider
})
export class ShipmentFormComponent {
  constructor(private formState: FormStateService) {}
}
```

```
Spring equivalent:
@Service
@Scope("prototype")  // New instance each time
public class FormStateService { }
```

✅ Use for: Form state, component-specific state, wizards

#### 3. Module Level — Shared Within a Feature Module

```typescript
@NgModule({
  providers: [
    WarehouseService  // Available to all components in this module
  ]
})
export class WarehouseModule {}
```

```
Spring equivalent:
// Like having a child ApplicationContext for a module
// Components in this module share one WarehouseService
// Components outside this module can't access it
```

### InjectionToken — For Non-Class Dependencies

> [!info] Spring Parallel
> `InjectionToken` is like `@Value("${property}")` in Spring.
> Used when you need to inject a string, number, or configuration object — not a class.

```typescript
import { InjectionToken } from '@angular/core';

// Define the token (like a property key in Spring)
export const API_BASE_URL = new InjectionToken<string>('API_BASE_URL');
export const MAX_RETRIES = new InjectionToken<number>('MAX_RETRIES');
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// Interface for complex config
export interface AppConfig {
  apiUrl: string;
  environment: 'dev' | 'staging' | 'prod';
  features: {
    enableTracking: boolean;
    enableNotifications: boolean;
  };
}
```

```typescript
// Provide the values (like application.yml in Spring)
// In app.config.ts or app.module.ts
providers: [
  { provide: API_BASE_URL, useValue: 'https://api.logistics.com' },
  { provide: MAX_RETRIES, useValue: 3 },
  {
    provide: APP_CONFIG,
    useValue: {
      apiUrl: 'https://api.logistics.com',
      environment: 'prod',
      features: {
        enableTracking: true,
        enableNotifications: false
      }
    }
  }
]
```

```typescript
// Inject the values (like @Value in Spring)
@Injectable({ providedIn: 'root' })
export class ShipmentService {
  constructor(
    private http: HttpClient,
    @Inject(API_BASE_URL) private apiUrl: string,     // @Inject required for tokens
    @Inject(MAX_RETRIES) private maxRetries: number
  ) {}

  getShipments(): Observable<Shipment[]> {
    return this.http.get<Shipment[]>(`${this.apiUrl}/shipments`).pipe(
      retry(this.maxRetries)
    );
  }
}
```

```java
// Spring Boot equivalent
@Service
public class ShipmentService {
    @Value("${api.base-url}")
    private String apiUrl;

    @Value("${api.max-retries}")
    private int maxRetries;
}
```

---

## Phase 4: Hierarchical Injectors — The Injector Tree

> [!tip] Spring Parallel
> Spring has parent-child `ApplicationContext` hierarchies.
> Angular has a similar **injector tree** that mirrors the component tree.

### The Injector Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│                     Root Injector                            │
│  (providedIn: 'root' services live here)                    │
│                                                              │
│  Services: ShipmentService, AuthService, HttpClient          │
│  Scope: Singleton — one instance for entire app              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │             Module Injector (Lazy)                    │   │
│  │  (Services from lazy-loaded modules)                  │   │
│  │                                                       │   │
│  │  Services: WarehouseService (only in this module)     │   │
│  │                                                       │   │
│  │  ┌─────────────────────┐  ┌─────────────────────┐   │   │
│  │  │  Component Injector │  │  Component Injector │   │   │
│  │  │  (Warehouse List)   │  │  (Warehouse Detail) │   │   │
│  │  │                     │  │                     │   │   │
│  │  │  providers: [       │  │  (inherits parent   │   │   │
│  │  │    FormState        │  │   providers)         │   │   │
│  │  │  ]                  │  │                     │   │   │
│  │  │                     │  │                     │   │   │
│  │  │  ┌───────────────┐  │  │                     │   │   │
│  │  │  │ Child Comp    │  │  │                     │   │   │
│  │  │  │ (inherits     │  │  │                     │   │   │
│  │  │  │  FormState    │  │  │                     │   │   │
│  │  │  │  from parent) │  │  │                     │   │   │
│  │  │  └───────────────┘  │  │                     │   │   │
│  │  └─────────────────────┘  └─────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

Resolution order (bottom → top):
  Child Component → Parent Component → Module → Root

If found at any level → use that instance
If not found anywhere → ERROR: No provider for X
```

### Practical Example — Shared vs Isolated State

```typescript
// Scenario: Two shipment forms on the same page
// Each form needs its OWN state, but shares the same ShipmentService

@Injectable()  // Note: NO providedIn — must be provided explicitly
export class FormStateService {
  private formData: Partial<Shipment> = {};

  setField(key: string, value: any): void {
    this.formData = { ...this.formData, [key]: value };
  }

  getFormData(): Partial<Shipment> {
    return { ...this.formData };
  }

  reset(): void {
    this.formData = {};
  }
}

@Component({
  selector: 'app-shipment-form',
  template: '...',
  providers: [FormStateService]  // ← Each form gets its OWN instance
})
export class ShipmentFormComponent {
  constructor(
    private formState: FormStateService,     // Unique to THIS form
    private shipmentService: ShipmentService  // Shared singleton from root
  ) {}
}
```

```
Page layout:

┌──────────────────────────────────────────────────┐
│  AppComponent                                     │
│  ShipmentService (singleton from root)            │
│                                                   │
│  ┌─────────────────┐  ┌─────────────────┐       │
│  │ ShipmentForm #1  │  │ ShipmentForm #2  │       │
│  │ FormState: { }   │  │ FormState: { }   │       │
│  │ (own instance)   │  │ (own instance)   │       │
│  │                  │  │                  │       │
│  │ Uses shared      │  │ Uses shared      │       │
│  │ ShipmentService  │  │ ShipmentService  │       │
│  └─────────────────┘  └─────────────────┘       │
└──────────────────────────────────────────────────┘

FormState = prototype scope (new per component)
ShipmentService = singleton scope (shared)
```

---

## Phase 5: HttpClient — Calling Your Spring Boot API

> [!info] Spring Parallel
> Angular's `HttpClient` = Spring's `RestTemplate` or `WebClient`
> But it returns **Observables** (reactive streams), like Spring WebFlux's `WebClient`.
> See also: [[RxJS and Reactive Programming]], [[Spring WebFlux]]

### Setup

```typescript
// In app.config.ts (standalone apps — Angular 17+)
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor])
    )
  ]
};
```

### Full CRUD Service — Shipment API

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, map, retry } from 'rxjs/operators';

// DTOs — like Java DTOs / Records
export interface Shipment {
  id: string;
  trackingNumber: string;
  origin: string;
  destination: string;
  status: ShipmentStatus;
  carrier: string;
  estimatedDelivery: string;
  weight: number;
}

export type ShipmentStatus = 'CREATED' | 'PICKED_UP' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELLED';

export interface CreateShipmentDto {
  origin: string;
  destination: string;
  carrier: string;
  weight: number;
}

export interface ShipmentPage {
  content: Shipment[];
  totalElements: number;
  totalPages: number;
  currentPage: number;
}

@Injectable({ providedIn: 'root' })
export class ShipmentService {
  private readonly apiUrl = '/api/v1/shipments';

  constructor(private http: HttpClient) {}

  // GET /api/v1/shipments?status=IN_TRANSIT&page=0&size=20
  getShipments(status?: ShipmentStatus, page = 0, size = 20): Observable<ShipmentPage> {
    let params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString());

    if (status) {
      params = params.set('status', status);
    }

    return this.http.get<ShipmentPage>(this.apiUrl, { params }).pipe(
      retry(2),                               // Retry failed requests up to 2 times
      catchError(this.handleError('getShipments'))
    );
  }

  // GET /api/v1/shipments/:id
  getById(id: string): Observable<Shipment> {
    return this.http.get<Shipment>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError('getById'))
    );
  }

  // POST /api/v1/shipments
  create(dto: CreateShipmentDto): Observable<Shipment> {
    return this.http.post<Shipment>(this.apiUrl, dto).pipe(
      catchError(this.handleError('create'))
    );
  }

  // PUT /api/v1/shipments/:id
  update(id: string, shipment: Partial<Shipment>): Observable<Shipment> {
    return this.http.put<Shipment>(`${this.apiUrl}/${id}`, shipment).pipe(
      catchError(this.handleError('update'))
    );
  }

  // PATCH /api/v1/shipments/:id/status
  updateStatus(id: string, status: ShipmentStatus): Observable<Shipment> {
    return this.http.patch<Shipment>(`${this.apiUrl}/${id}/status`, { status }).pipe(
      catchError(this.handleError('updateStatus'))
    );
  }

  // DELETE /api/v1/shipments/:id
  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`).pipe(
      catchError(this.handleError('delete'))
    );
  }

  // Reusable error handler — like @ControllerAdvice in Spring
  private handleError(operation: string) {
    return (error: any): Observable<never> => {
      console.error(`${operation} failed:`, error);

      const message = error.error?.message || error.message || 'Unknown error';
      // Could also push to a NotificationService here
      return throwError(() => new Error(`${operation}: ${message}`));
    };
  }
}
```

### Comparison with Spring's WebClient

```java
// Spring Boot WebClient — reactive HTTP client
@Service
public class ExternalCarrierService {
    private final WebClient webClient;

    public ExternalCarrierService(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("https://carrier-api.com").build();
    }

    public Mono<CarrierRate> getRate(String carrierId) {
        return webClient.get()
            .uri("/rates/{id}", carrierId)
            .retrieve()
            .bodyToMono(CarrierRate.class)     // Returns reactive Mono
            .retryWhen(Retry.fixedDelay(2, Duration.ofSeconds(1)));
    }
}
```

```typescript
// Angular HttpClient — also reactive!
@Injectable({ providedIn: 'root' })
export class CarrierService {
  constructor(private http: HttpClient) {}

  getRate(carrierId: string): Observable<CarrierRate> {
    return this.http.get<CarrierRate>(
      `https://carrier-api.com/rates/${carrierId}`
    ).pipe(
      retry(2)                                 // Returns reactive Observable
    );
  }
}
```

```
Spring WebClient returns:   Mono<T>  / Flux<T>     (Project Reactor)
Angular HttpClient returns:  Observable<T>          (RxJS)

Both are reactive. Both are lazy. Both need to be subscribed to.
```

### HTTP Interceptors — Cross-Cutting Concerns

> [!info] Spring Parallel
> Angular interceptors = Spring's `ClientHttpRequestInterceptor` or `WebFilter`.
> They modify EVERY HTTP request/response — perfect for auth tokens, logging, error handling.

```typescript
// auth.interceptor.ts — adds JWT token to every request
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  if (token) {
    // Clone the request and add the auth header
    const authReq = req.clone({
      headers: req.headers.set('Authorization', `Bearer ${token}`)
    });
    return next(authReq);
  }

  return next(req);
};
```

```typescript
// logging.interceptor.ts — logs request/response timing
import { HttpInterceptorFn } from '@angular/common/http';
import { tap } from 'rxjs/operators';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = Date.now();
  console.log(`→ ${req.method} ${req.url}`);

  return next(req).pipe(
    tap({
      next: () => {
        const elapsed = Date.now() - started;
        console.log(`← ${req.method} ${req.url} (${elapsed}ms)`);
      },
      error: (err) => {
        const elapsed = Date.now() - started;
        console.error(`✗ ${req.method} ${req.url} FAILED (${elapsed}ms)`, err);
      }
    })
  );
};
```

```typescript
// error.interceptor.ts — global error handling
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      switch (error.status) {
        case 401:
          router.navigate(['/login']);       // Redirect to login
          break;
        case 403:
          router.navigate(['/forbidden']);   // Access denied
          break;
        case 500:
          // Show global error notification
          console.error('Server error:', error.message);
          break;
      }
      return throwError(() => error);
    })
  );
};
```

```typescript
// Register interceptors in app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        authInterceptor,      // Runs first — adds token
        loggingInterceptor,   // Runs second — logs timing
        errorInterceptor      // Runs third — handles errors
      ])
    )
  ]
};
```

```
Interceptor execution order:

  Request:   authInterceptor → loggingInterceptor → errorInterceptor → HTTP
  Response:  HTTP → errorInterceptor → loggingInterceptor → authInterceptor

Same as Spring's filter chain or HandlerInterceptor chain.
```

---

## Phase 6: Advanced DI Patterns

### Provider Types — The Four Ways to Provide

```typescript
// 1. useClass — provide a specific implementation
// Like: @Bean public PaymentGateway gateway() { return new StripeGateway(); }
{ provide: PaymentGateway, useClass: StripePaymentGateway }

// 2. useValue — provide a static value
// Like: @Value("${api.url}") or @Bean public String apiUrl() { return "..."; }
{ provide: API_URL, useValue: 'https://api.logistics.com' }

// 3. useFactory — provide via factory function
// Like: @Bean with conditional logic
{
  provide: ShipmentService,
  useFactory: (http: HttpClient, config: AppConfig) => {
    if (config.environment === 'dev') {
      return new MockShipmentService();       // Mock in dev
    }
    return new ShipmentService(http);         // Real in prod
  },
  deps: [HttpClient, APP_CONFIG]              // Factory dependencies
}

// 4. useExisting — alias one token to another
// Like: having two interfaces point to the same bean
{ provide: AbstractLogger, useExisting: ConsoleLoggerService }
```

### Multi Providers — Multiple Values for One Token

```typescript
// Like Spring's List<Validator> injection
export const SHIPMENT_VALIDATOR = new InjectionToken<ShipmentValidator[]>('SHIPMENT_VALIDATOR');

// Each module can contribute validators
providers: [
  { provide: SHIPMENT_VALIDATOR, useClass: WeightValidator, multi: true },
  { provide: SHIPMENT_VALIDATOR, useClass: AddressValidator, multi: true },
  { provide: SHIPMENT_VALIDATOR, useClass: CarrierValidator, multi: true },
]

// Inject ALL validators
@Injectable({ providedIn: 'root' })
export class ShipmentValidationService {
  constructor(
    @Inject(SHIPMENT_VALIDATOR) private validators: ShipmentValidator[]
  ) {}

  validate(shipment: Shipment): ValidationResult[] {
    return this.validators.map(v => v.validate(shipment));
  }
}
```

### Optional Dependencies

```typescript
@Injectable({ providedIn: 'root' })
export class ShipmentService {
  constructor(
    private http: HttpClient,
    @Optional() private logger?: LoggingService  // Won't throw if not provided
  ) {}

  getShipments(): Observable<Shipment[]> {
    this.logger?.log('Fetching shipments');  // Safe — might be undefined
    return this.http.get<Shipment[]>(this.apiUrl);
  }
}
```

```java
// Spring equivalent
@Service
public class ShipmentService {
    private final LoggingService logger;  // Could be null

    public ShipmentService(
        @Autowired(required = false) LoggingService logger
    ) {
        this.logger = logger;
    }
}
```

### Environment-Based Configuration

```typescript
// environments/environment.ts (dev)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api',
  features: { enableTracking: true }
};

// environments/environment.prod.ts (prod)
export const environment = {
  production: true,
  apiUrl: 'https://api.logistics.com',
  features: { enableTracking: true }
};

// Provide as InjectionToken
export const ENVIRONMENT = new InjectionToken<Environment>('ENVIRONMENT');

providers: [
  { provide: ENVIRONMENT, useValue: environment }
]

// Use in services
@Injectable({ providedIn: 'root' })
export class ApiService {
  constructor(@Inject(ENVIRONMENT) private env: Environment) {}

  getBaseUrl(): string {
    return this.env.apiUrl;
  }
}
```

```
Spring equivalent:
  application-dev.yml    → environment.ts
  application-prod.yml   → environment.prod.ts
  @Profile("dev")        → angular.json build configurations
  @Value("${api.url}")   → @Inject(ENVIRONMENT)
```

---

## Phase 7: Best Practices

### ✅ Do

| Practice | Why |
|----------|-----|
| One service per domain concept | `ShipmentService`, `CarrierService`, `BookingService` |
| Use `providedIn: 'root'` for singletons | Tree-shakeable, no module registration needed |
| Constructor injection only | Angular's design — no field injection |
| Keep services stateless when possible | Easier to test, fewer bugs |
| Use interceptors for cross-cutting concerns | Auth tokens, logging, error handling |
| Return Observables from services | Let the component decide when to subscribe |
| Use typed DTOs / interfaces | Type safety catches bugs at compile time |
| Handle errors in services | Consistent error handling across components |

### ❌ Don't

| Anti-Pattern | Why It's Bad |
|-------------|-------------|
| HttpClient in components | Breaks separation of concerns |
| Business logic in components | Can't reuse or unit test |
| `any` types for API responses | Loses TypeScript's safety |
| Subscribing in services | Services should return Observables, not subscribe |
| Forgetting to handle errors | Silent failures are the worst bugs |
| Giant "god" services | Split by domain — same as Spring |
| Manual `new Service()` | Bypasses DI — can't test, can't swap |

### Service File Structure

```
src/
  app/
    core/                          ← App-wide singletons
      services/
        auth.service.ts
        http-error.service.ts
      interceptors/
        auth.interceptor.ts
        logging.interceptor.ts
      tokens/
        app-config.token.ts
    features/
      shipments/                   ← Feature-specific services
        services/
          shipment.service.ts
          shipment-validation.service.ts
        models/
          shipment.model.ts
          shipment-dto.model.ts
        components/
          shipment-list/
          shipment-detail/
      carriers/
        services/
          carrier.service.ts
        models/
          carrier.model.ts
```

```
Spring Boot equivalent structure:
src/main/java/com/logistics/
  config/                          ← @Configuration classes
  security/                        ← Filters, interceptors
  shipment/
    service/ShipmentService.java
    dto/ShipmentDto.java
    controller/ShipmentController.java
  carrier/
    service/CarrierService.java
    dto/CarrierDto.java
```

---

## Quick Reference — DI Cheat Sheet

```
┌────────────────────────────────────────────────────────────┐
│              ANGULAR DI CHEAT SHEET                        │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  CREATE A SERVICE:                                         │
│  @Injectable({ providedIn: 'root' })                       │
│  export class MyService { }                                │
│                                                            │
│  INJECT A SERVICE:                                         │
│  constructor(private myService: MyService) { }             │
│                                                            │
│  INJECT A TOKEN:                                           │
│  constructor(@Inject(MY_TOKEN) private val: string) { }    │
│                                                            │
│  CREATE A TOKEN:                                           │
│  export const MY_TOKEN = new InjectionToken<string>('..'); │
│                                                            │
│  PROVIDE A VALUE:                                          │
│  providers: [{ provide: MY_TOKEN, useValue: 'hello' }]     │
│                                                            │
│  COMPONENT-LEVEL (prototype scope):                        │
│  @Component({ providers: [MyService] })                    │
│                                                            │
│  OPTIONAL DEPENDENCY:                                      │
│  constructor(@Optional() private log?: LogService) { }     │
│                                                            │
│  INTERCEPTOR:                                              │
│  provideHttpClient(withInterceptors([myInterceptor]))      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Related Notes

- [[Angular Fundamentals]] — Core concepts and project structure
- [[Components and Templates]] — Building blocks that consume services
- [[RxJS and Reactive Programming]] — Observables returned by HttpClient
- [[React vs Angular Comparison]] — How Angular DI compares to React patterns
- [[Spring WebFlux]] — Reactive Spring (parallels with Angular's Observable-based HTTP)

---

> [!quote] Final Thought
> *"If you can write a Spring `@Service` with constructor injection, you can write an Angular service. The syntax changed. The pattern didn't."*
