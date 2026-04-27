---
tags:
  - angular
  - frontend
  - typescript
  - fundamentals
  - beginner
created: 2025-07-15
---

# Angular Fundamentals

> *"Angular is a platform and framework for building single-page client applications using HTML and TypeScript."* — Angular.io

> *If Spring Boot is the opinionated backend framework that gives you everything out of the box, Angular is its frontend twin — opinionated, batteries-included, and built for enterprise scale.*

---

## Phase 1: What is Angular?

### The Short Answer

Angular is a **full-featured frontend framework** built by Google for building single-page applications (SPAs). Unlike libraries like React that only handle the "view" layer, Angular is a **complete platform** — it ships with routing, forms, HTTP, testing, dependency injection, and more.

### Key Facts

| Fact | Detail |
|---|---|
| **Creator** | Google |
| **First Release** | September 2016 (Angular 2) |
| **Current Version** | Angular 17+ (as of 2024) |
| **Language** | TypeScript (required, not optional) |
| **Architecture** | Component-based with modules |
| **Rendering** | Client-side (SSR available via Angular Universal) |
| **License** | MIT |

> [!warning] Angular ≠ AngularJS
> **AngularJS** (v1.x, released 2010) is a completely different framework written in JavaScript. It's **dead** — end-of-life since December 2021. When anyone says "Angular" today, they mean **Angular 2+** (the TypeScript rewrite). Don't confuse them.

### Framework vs Library — Why It Matters

```
Library (React):
  You build the house. You pick the plumbing, electrical, roofing.
  Maximum flexibility, but you make 100 decisions.

Framework (Angular):
  You get a pre-built house with plumbing, electrical, and roofing included.
  Less flexibility, but you ship faster with fewer decisions.
```

### 🔁 The Spring Boot Analogy

If you know Spring Boot, you already understand Angular's philosophy:

| Concept | Spring Boot | Angular |
|---|---|---|
| **Philosophy** | Opinionated, convention over config | Opinionated, convention over config |
| **Setup** | Spring Initializr generates project | Angular CLI generates project |
| **DI** | `@Autowired` / constructor injection | Constructor injection (built-in DI) |
| **Modules** | `@Configuration` classes | `@NgModule` / Standalone components |
| **HTTP** | `RestTemplate` / `WebClient` | `HttpClient` |
| **Routing** | `@RequestMapping`, `@GetMapping` | `RouterModule`, route configs |
| **Testing** | JUnit + Mockito | Jasmine + Karma (or Jest) |
| **Build** | Maven / Gradle | Angular CLI (Webpack/esbuild) |
| **Config** | `application.yml` | `angular.json`, `environment.ts` |

> [!tip] Spring Boot Developer Advantage
> If React is like Spring Boot with manual configuration (you pick every starter, configure every bean), Angular is like **Spring Boot with Initializr + ALL starters pre-configured**. Everything works together because it was designed together.

### What Angular Ships With (Batteries Included)

| Feature | Angular Module | Spring Boot Equivalent |
|---|---|---|
| **Routing** | `@angular/router` | Spring MVC `@RequestMapping` |
| **Forms** | `@angular/forms` | Spring Form Binding |
| **HTTP Client** | `@angular/common/http` | `RestTemplate` / `WebClient` |
| **Testing** | `@angular/core/testing` | `spring-boot-starter-test` |
| **Animations** | `@angular/animations` | (no equivalent) |
| **i18n** | `@angular/localize` | `MessageSource` |
| **PWA Support** | `@angular/pwa` | (no equivalent) |
| **CLI** | `@angular/cli` | Spring Boot CLI |

---

## Phase 2: Angular Architecture Overview

### The Big Picture

```
┌─────────────────────────────────────────────────────┐
│                  Angular Application                 │
│                                                     │
│  ┌─────────────── Module / Standalone ────────────┐ │
│  │                                                │ │
│  │  ┌── Components (UI) ──────────────────────┐  │ │
│  │  │  ┌─────────┐ ┌──────────┐ ┌─────────┐  │  │ │
│  │  │  │Template  │ │  Class   │ │ Styles  │  │  │ │
│  │  │  │  (HTML)  │ │  (.ts)   │ │ (CSS)   │  │  │ │
│  │  │  └─────────┘ └──────────┘ └─────────┘  │  │ │
│  │  └────────────────────────────────────────┘  │ │
│  │                                                │ │
│  │  ┌── Services (Business Logic) ────────────┐  │ │
│  │  │  @Injectable → like @Service in Spring  │  │ │
│  │  └────────────────────────────────────────┘  │ │
│  │                                                │ │
│  │  ┌── Directives ──┐  ┌── Pipes ──────────┐   │ │
│  │  │ DOM behavior   │  │ Data transformers  │   │ │
│  │  │ (like AOP)     │  │ (like Converters)  │   │ │
│  │  └────────────────┘  └───────────────────┘   │ │
│  │                                                │ │
│  │  ┌── Guards (Route Protection) ────────────┐  │ │
│  │  │ Like Spring Security Filters             │  │ │
│  │  └────────────────────────────────────────┘  │ │
│  │                                                │ │
│  └────────────────────────────────────────────────┘ │
│                                                     │
│  ┌── Router ────────────────────────────────────┐   │
│  │  Maps URLs to components (like @GetMapping)  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Angular ↔ Spring Boot Architecture Mapping

| Angular Concept | Spring Boot Equivalent | Purpose |
|---|---|---|
| **Component** | `@Controller` + Thymeleaf view | UI + presentation logic |
| **Service** | `@Service` | Business logic, data access |
| **Module** | `@Configuration` class | Groups related functionality |
| **Guard** | `SecurityFilterChain` / `OncePerRequestFilter` | Route protection, auth |
| **Pipe** | `Converter<S,T>` / `Formatter<T>` | Data transformation for display |
| **Directive** | AOP `@Aspect` | Cross-cutting DOM behavior |
| **DI** | `@Autowired` / constructor injection | Dependency injection |
| **Interceptor** | `HandlerInterceptor` / `ClientHttpRequestInterceptor` | HTTP request/response middleware |
| **Route** | `@RequestMapping` / `@GetMapping` | URL → handler mapping |
| **Environment** | `application.yml` profiles | Environment-specific config |

> [!tip] Mental Model
> Think of an Angular app like a Spring Boot app where:
> - **Components** are your `@Controller`s — they handle a specific URL/view
> - **Services** are your `@Service`s — they hold business logic and call APIs
> - **The Router** is like `@RequestMapping` — it maps URLs to components
> - **Guards** are like `@PreAuthorize` or security filters — they protect routes
> - **Pipes** are like `@JsonSerialize` — they transform data for display

### Data Flow in Angular

```
User Action (click, type, navigate)
         │
         ▼
    ┌─────────┐      ┌──────────┐      ┌──────────┐
    │Component│ ───→ │ Service  │ ───→ │ HTTP API │
    │  (View) │      │ (Logic)  │      │ (Backend)│
    └─────────┘      └──────────┘      └──────────┘
         ▲                                    │
         │                                    │
         └────────── Data Response ───────────┘

  Compare to Spring Boot:
    Controller → Service → Repository → Database
```

**Logistics Example:**
1. User clicks "Track Shipment" → **Component** captures the event
2. Component calls `ShipmentService.track(trackingId)` → **Service** makes HTTP call
3. Service calls `GET /api/shipments/{trackingId}` → **Backend** returns data
4. Component receives data → **Template** renders the shipment card

---

## Phase 3: Setting Up Angular

### Prerequisites

```bash
# You need Node.js (LTS version recommended)
node --version    # Should be v18+ or v20+
npm --version     # Comes with Node.js
```

> [!tip] For Spring Boot developers
> Think of `npm` as your `Maven/Gradle` — it manages dependencies from a central registry (npmjs.com ≈ Maven Central).

### Install Angular CLI

```bash
# Install globally (like installing Maven on your system)
npm install -g @angular/cli

# Verify installation
ng version
```

### Create a New Project

```bash
# Create a new Angular app (like Spring Initializr)
ng new freight-tracker

# CLI will ask:
# ? Which stylesheet format? → CSS (pick CSS for now)
# ? Do you want to enable Server-Side Rendering (SSR)? → No
```

### Project Structure Explained

```
freight-tracker/
├── src/
│   ├── app/
│   │   ├── app.component.ts        ← Root component class
│   │   ├── app.component.html      ← Root component template
│   │   ├── app.component.css       ← Root component styles
│   │   ├── app.component.spec.ts   ← Root component tests
│   │   ├── app.config.ts           ← App-level configuration
│   │   └── app.routes.ts           ← Route definitions
│   │
│   ├── main.ts                     ← Entry point (like main() in Spring Boot)
│   ├── index.html                  ← The single HTML page (SPA)
│   └── styles.css                  ← Global styles
│
├── public/                         ← Static assets (images, fonts)
│
├── angular.json                    ← Build configuration (like pom.xml)
├── tsconfig.json                   ← TypeScript compiler options
├── tsconfig.app.json               ← App-specific TS config
├── tsconfig.spec.json              ← Test-specific TS config
├── package.json                    ← Dependencies (like pom.xml dependencies)
├── package-lock.json               ← Locked dependency versions
└── .editorconfig                   ← Editor settings
```

### Spring Boot ↔ Angular File Mapping

| Angular File | Spring Boot Equivalent | Purpose |
|---|---|---|
| `main.ts` | `public static void main(String[] args)` | Application entry point |
| `app.config.ts` | `@SpringBootApplication` config | Application-level setup |
| `app.routes.ts` | `@RequestMapping` annotations | URL routing |
| `angular.json` | `pom.xml` / `build.gradle` | Build configuration |
| `package.json` | `pom.xml` (dependencies section) | Dependency declarations |
| `tsconfig.json` | (no direct equiv — Java doesn't need this) | Compiler configuration |
| `environment.ts` | `application.yml` / `application-dev.yml` | Environment-specific config |
| `.spec.ts` files | `*Test.java` / `*IT.java` files | Unit/integration tests |

### Essential CLI Commands

```bash
# Start development server (like mvn spring-boot:run)
ng serve
# App runs at http://localhost:4200 with hot reload

# Start on a different port
ng serve --port 3000

# Build for production (like mvn package)
ng build
# Output goes to dist/ folder

# Run unit tests (like mvn test)
ng test

# Run end-to-end tests
ng e2e

# Generate a new component (like creating a new @Controller class)
ng generate component shipment-card
# shorthand: ng g c shipment-card

# Generate a service (like creating a new @Service class)
ng generate service shipment
# shorthand: ng g s shipment

# Lint your code
ng lint
```

> [!tip] Hot Reload
> `ng serve` watches your files and **auto-reloads** the browser when you save changes. It's like Spring Boot DevTools — but even faster because it only recompiles the changed files.

---

## Phase 4: Modules vs Standalone Components

### The Evolution

Angular's architecture has evolved significantly:

```
Angular 2-13:    NgModules (required)
Angular 14-16:   Standalone components (optional, new)
Angular 17+:     Standalone components (default, recommended)
```

### NgModules (Traditional Approach)

An NgModule is a class decorated with `@NgModule` that groups related components, services, directives, and pipes together.

```typescript
// app.module.ts — like a @Configuration class in Spring
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';
import { ShipmentListComponent } from './shipment-list/shipment-list.component';
import { ShipmentCardComponent } from './shipment-card/shipment-card.component';
import { ShipmentService } from './services/shipment.service';

@NgModule({
  // Components, directives, pipes that BELONG to this module
  declarations: [
    AppComponent,
    ShipmentListComponent,
    ShipmentCardComponent
  ],
  // Other modules this module NEEDS
  imports: [
    BrowserModule,
    HttpClientModule
  ],
  // Services available for DI (app-wide)
  providers: [
    ShipmentService
  ],
  // The root component to bootstrap
  bootstrap: [AppComponent]
})
export class AppModule { }
```

**Spring Boot analogy:**
```java
// This is roughly equivalent to:
@Configuration
@ComponentScan(basePackages = "com.freight.tracker")
@EnableWebMvc
@Import({ SecurityConfig.class, HttpConfig.class })
public class AppConfig {
    @Bean
    public ShipmentService shipmentService() {
        return new ShipmentService();
    }
}
```

### Standalone Components (Modern Approach — Recommended ✅)

Starting with Angular 14+, components can declare their own dependencies without needing a module. This is now the **default** in Angular 17+.

```typescript
// shipment-card.component.ts — self-contained, no module needed!
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ShipmentStatusPipe } from '../pipes/shipment-status.pipe';

@Component({
  selector: 'app-shipment-card',
  standalone: true,           // ← This makes it standalone
  imports: [                  // ← Declares its own dependencies
    CommonModule,
    ShipmentStatusPipe
  ],
  template: `
    <div class="shipment-card">
      <h3>{{ trackingId }}</h3>
      <p>Status: {{ status | shipmentStatus }}</p>
      <p>{{ origin }} → {{ destination }}</p>
    </div>
  `
})
export class ShipmentCardComponent {
  @Input() trackingId!: string;
  @Input() status!: string;
  @Input() origin!: string;
  @Input() destination!: string;
}
```

### NgModule vs Standalone — Comparison

| Aspect | NgModule | Standalone |
|---|---|---|
| **Boilerplate** | High — every component needs a module | Low — each component is self-contained |
| **Dependencies** | Declared at module level | Declared at component level |
| **Tree-shaking** | Module-level (larger bundles) | Component-level (smaller bundles) |
| **Learning curve** | Steeper — must understand module system | Lower — more intuitive |
| **Default in Angular 17+** | ❌ No | ✅ Yes |
| **Recommended for new projects** | ❌ No | ✅ Yes |

> [!tip] Recommendation
> **Use standalone components for all new projects.** NgModules still work and you'll see them in older codebases, but standalone is the future of Angular. This is similar to how Spring Boot moved from XML configuration to annotation-based configuration — the old way works, but nobody starts new projects with it.

---

## Phase 5: Decorators — Angular's Annotations

### What Are Decorators?

TypeScript decorators are **metadata annotations** — they tell Angular how to process a class. If you know Java annotations, you already understand decorators.

```
Java:         @Component, @Service, @Autowired, @RequestMapping
TypeScript:   @Component, @Injectable, @Input, @Output
```

### Core Decorators

| Decorator | Purpose | Java Equivalent |
|---|---|---|
| `@Component({...})` | Marks a class as a UI component | `@Controller` + `@RequestMapping` |
| `@Injectable({...})` | Marks a class as a service for DI | `@Service` / `@Component` |
| `@NgModule({...})` | Defines a module | `@Configuration` |
| `@Input()` | Accepts data from parent component | Method parameter in controller |
| `@Output()` | Emits events to parent component | (no direct equiv) |
| `@ViewChild()` | References a child component/element | `@Autowired` a specific bean |
| `@HostListener()` | Listens to host element events | `@EventListener` |
| `@Pipe({...})` | Marks a class as a data transformer | Implementing `Converter<S,T>` |
| `@Directive({...})` | Marks a class as a directive | `@Aspect` (AOP) |

### Example: How Decorators Map to Java Annotations

```typescript
// Angular
@Component({
  selector: 'app-shipment-list',  // like @RequestMapping("/shipments")
  standalone: true,
  imports: [CommonModule],
  templateUrl: './shipment-list.component.html',
  styleUrls: ['./shipment-list.component.css']
})
export class ShipmentListComponent {
  // @Input is like a method parameter
  @Input() warehouseId!: string;

  // constructor injection — exactly like Spring!
  constructor(private shipmentService: ShipmentService) {}
}
```

```java
// Spring Boot equivalent (conceptually)
@Controller
@RequestMapping("/shipments")
public class ShipmentListController {
    private final ShipmentService shipmentService;

    // Constructor injection — same pattern!
    public ShipmentListController(ShipmentService shipmentService) {
        this.shipmentService = shipmentService;
    }

    @GetMapping
    public String list(@RequestParam String warehouseId, Model model) {
        model.addAttribute("shipments",
            shipmentService.getByWarehouse(warehouseId));
        return "shipment-list";
    }
}
```

> [!tip] Key Insight
> Angular uses **constructor injection** for DI — exactly like Spring Boot's recommended approach. If you've been using constructor injection in Spring (as you should), Angular's DI will feel completely natural.

---

## Phase 6: Angular CLI — Your Best Friend

### Why the CLI Matters

The Angular CLI is like **Spring Initializr + Maven + DevTools** combined. It generates code, runs the dev server, builds, tests, and lints — all from the command line.

### Code Generation Commands

```bash
# Generate a component (creates 4 files: .ts, .html, .css, .spec.ts)
ng generate component shipment-card
ng g c shipment-card                    # shorthand

# Generate inside a specific folder
ng g c features/shipment/shipment-card

# Generate a component with inline template (no separate .html)
ng g c shipment-badge --inline-template

# Generate a component without tests
ng g c shipment-badge --skip-tests

# Generate a service
ng g s services/shipment               # creates shipment.service.ts
ng g s services/freight-rate            # creates freight-rate.service.ts

# Generate a pipe
ng g pipe pipes/shipment-status        # creates shipment-status.pipe.ts

# Generate a directive
ng g directive directives/highlight    # creates highlight.directive.ts

# Generate a guard
ng g guard guards/auth                 # creates auth.guard.ts

# Generate an interface
ng g interface models/shipment         # creates shipment.ts with interface

# Generate an enum
ng g enum models/shipment-status       # creates shipment-status.ts with enum
```

### What `ng g c shipment-card` Creates

```
src/app/shipment-card/
├── shipment-card.component.ts         ← Component class
├── shipment-card.component.html       ← Template
├── shipment-card.component.css        ← Styles
└── shipment-card.component.spec.ts    ← Unit test
```

**Generated component:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-shipment-card',
  standalone: true,
  imports: [],
  templateUrl: './shipment-card.component.html',
  styleUrl: './shipment-card.component.css'
})
export class ShipmentCardComponent {

}
```

> [!tip] Naming Convention
> Angular CLI enforces **kebab-case** for file names and **PascalCase** for class names — like Java's package naming convention. `shipment-card` → `ShipmentCardComponent`. This is similar to Spring's convention where `ShipmentService.java` is the standard.

### Schematics — Code Generation Templates

Angular schematics are like **Maven archetypes** — they define templates for generating code. The CLI uses schematics under the hood.

```bash
# See all available schematics
ng generate --help

# Common schematics:
# component, service, pipe, directive, guard,
# module, interface, enum, class, interceptor
```

---

## Phase 7: Your First Angular Component

### Step 1: Generate the Component

```bash
# Generate a shipment tracker component
ng g c shipment-tracker
```

### Step 2: Define the Component Class

```typescript
// src/app/shipment-tracker/shipment-tracker.component.ts
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

// The interface for our shipment data
// (Like a Java DTO / record)
interface Shipment {
  trackingId: string;
  origin: string;
  destination: string;
  status: 'pending' | 'in-transit' | 'delivered' | 'cancelled';
  carrier: string;
  estimatedDelivery: Date;
}

@Component({
  selector: 'app-shipment-tracker',    // HTML tag name to use this component
  standalone: true,                     // Self-contained, no module needed
  imports: [CommonModule],              // Import Angular's common directives
  templateUrl: './shipment-tracker.component.html',
  styleUrls: ['./shipment-tracker.component.css']
})
export class ShipmentTrackerComponent {

  // Component title — displayed in template via interpolation
  title = 'Freight Tracker Dashboard';

  // Shipment data — in a real app, this comes from a service
  // (Like populating a Model in a Spring Controller)
  shipments: Shipment[] = [
    {
      trackingId: 'FRT-2024-001',
      origin: 'Shanghai, CN',
      destination: 'Sydney, AU',
      status: 'in-transit',
      carrier: 'Maersk',
      estimatedDelivery: new Date('2024-12-20')
    },
    {
      trackingId: 'FRT-2024-002',
      origin: 'Rotterdam, NL',
      destination: 'Mumbai, IN',
      status: 'pending',
      carrier: 'MSC',
      estimatedDelivery: new Date('2024-12-25')
    },
    {
      trackingId: 'FRT-2024-003',
      origin: 'Los Angeles, US',
      destination: 'Tokyo, JP',
      status: 'delivered',
      carrier: 'COSCO',
      estimatedDelivery: new Date('2024-12-10')
    }
  ];

  // Method to get CSS class based on status
  // (Like a utility method in a Spring controller)
  getStatusClass(status: string): string {
    const classMap: Record<string, string> = {
      'pending': 'status-pending',
      'in-transit': 'status-transit',
      'delivered': 'status-delivered',
      'cancelled': 'status-cancelled'
    };
    return classMap[status] || 'status-default';
  }

  // Method to get display text for status
  getStatusEmoji(status: string): string {
    const emojiMap: Record<string, string> = {
      'pending': '📦',
      'in-transit': '🚚',
      'delivered': '✅',
      'cancelled': '❌'
    };
    return emojiMap[status] || '❓';
  }
}
```

### Step 3: Create the Template

```html
<!-- src/app/shipment-tracker/shipment-tracker.component.html -->

<!-- Interpolation: {{ expression }} binds data from component class -->
<h1>{{ title }}</h1>

<p>Tracking {{ shipments.length }} shipments</p>

<div class="shipment-list">
  <!-- @for loop — iterates over shipments array -->
  <!-- track is required for performance (like key in React) -->
  @for (shipment of shipments; track shipment.trackingId) {
    <div class="shipment-card">
      <div class="card-header">
        <span class="tracking-id">{{ shipment.trackingId }}</span>
        <span [class]="getStatusClass(shipment.status)">
          {{ getStatusEmoji(shipment.status) }} {{ shipment.status }}
        </span>
      </div>
      <div class="card-body">
        <p><strong>Route:</strong> {{ shipment.origin }} → {{ shipment.destination }}</p>
        <p><strong>Carrier:</strong> {{ shipment.carrier }}</p>
        <p><strong>ETA:</strong> {{ shipment.estimatedDelivery.toLocaleDateString() }}</p>
      </div>
    </div>
  } @empty {
    <p>No shipments to display.</p>
  }
</div>
```

### Step 4: Add Styles

```css
/* src/app/shipment-tracker/shipment-tracker.component.css */

/* Angular scopes styles to this component automatically */
/* No CSS leaking between components — like shadow DOM */

.shipment-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1rem;
  padding: 1rem;
}

.shipment-card {
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.card-header {
  display: flex;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  background: #f5f5f5;
  font-weight: bold;
}

.card-body { padding: 1rem; }
.card-body p { margin: 0.5rem 0; }

.status-pending { color: #ff9800; }
.status-transit { color: #2196f3; }
.status-delivered { color: #4caf50; }
.status-cancelled { color: #f44336; }
```

### Step 5: Use the Component

```html
<!-- src/app/app.component.html -->
<!-- Use the shipment tracker component with its selector -->
<app-shipment-tracker />
```

```typescript
// src/app/app.component.ts — import the new component
import { Component } from '@angular/core';
import { ShipmentTrackerComponent } from './shipment-tracker/shipment-tracker.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ShipmentTrackerComponent],  // ← Import it here
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = 'freight-tracker';
}
```

### Step 6: Run It

```bash
ng serve
# Open http://localhost:4200 — you'll see the shipment tracker!
```

### How This Maps to Spring Boot

```
Angular                          Spring Boot
─────────────────────────────────────────────────────
ShipmentTrackerComponent    ≈    ShipmentController
  → title, shipments        ≈    model.addAttribute(...)
  → template.html           ≈    shipment-list.html (Thymeleaf)
  → component.css           ≈    (styles in HTML/CSS files)
  → getStatusClass()        ≈    utility method in controller

app.component.html          ≈    layout.html (Thymeleaf layout)
  → <app-shipment-tracker>  ≈    <div th:replace="fragments/shipment-list">

app.component.ts imports    ≈    @ComponentScan finding the controller
```

> [!tip] Component Encapsulation
> Angular **automatically scopes CSS to the component** — styles in `shipment-tracker.component.css` ONLY apply to that component's template. This is like having CSS modules built in. No more CSS class name conflicts across your app!

---

## Quick Reference — Angular CLI Cheat Sheet

| Command | Purpose | Spring Boot Equivalent |
|---|---|---|
| `ng new app-name` | Create project | Spring Initializr |
| `ng serve` | Dev server + hot reload | `mvn spring-boot:run` + DevTools |
| `ng build` | Production build | `mvn package` |
| `ng test` | Unit tests | `mvn test` |
| `ng e2e` | E2E tests | Integration tests |
| `ng g c name` | Generate component | Create `@Controller` class |
| `ng g s name` | Generate service | Create `@Service` class |
| `ng g pipe name` | Generate pipe | Create `Converter` |
| `ng g guard name` | Generate guard | Create `SecurityFilter` |
| `ng g directive name` | Generate directive | Create `@Aspect` |
| `ng lint` | Lint code | Checkstyle / SpotBugs |
| `ng update` | Update dependencies | `mvn versions:use-latest` |

---

## What's Next?

Now that you understand Angular's fundamentals, architecture, and how it maps to Spring Boot:

- **[[Components and Templates]]** — Deep dive into component anatomy, data binding, template syntax, and component communication
- **[[Directives and Pipes]]** — DOM manipulation with directives and data transformation with pipes
- **[[Angular Lifecycle Hooks]]** — Component lifecycle methods (like `@PostConstruct` in Spring)
- **[[Angular Services and DI]]** — Dependency injection deep dive (very similar to Spring DI)
- **[[Angular Routing]]** — Client-side routing (like `@RequestMapping` for the frontend)
- **[[Angular Forms]]** — Template-driven and reactive forms
- **[[Angular HTTP Client]]** — Making API calls (like `RestTemplate` / `WebClient`)

---

## See Also

- [[Frontend Fundamentals]]
- [[React Fundamentals]]
- [[React vs Angular Comparison]]
- [[TypeScript Fundamentals]]
