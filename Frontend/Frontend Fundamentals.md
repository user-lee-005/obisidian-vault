# Frontend Fundamentals

> *"The best interface is no interface."* — Golden Krishna

---

## Phase 1: What is Frontend Development?

### The UI Layer

Frontend development is everything the **user sees and interacts with** in a web application — buttons, forms, tables, navigation, animations, and layout. It runs in the **browser** (Chrome, Firefox, Safari) on the user's machine.

**Analogy:** If your backend (Spring Boot) is the **warehouse** — processing orders, managing inventory, coordinating shipments — then the frontend is the **customer-facing storefront**. Customers never see the warehouse. They see the storefront's shelves, signage, and checkout counter. The storefront must be fast, intuitive, and attractive, even though all the real logistics happen behind the scenes.

```
┌─────────────────────────────────────────────────────┐
│                    USER'S BROWSER                    │
│                                                     │
│   ┌─────────────┐  ┌──────────┐  ┌──────────────┐  │
│   │    HTML      │  │   CSS    │  │  JavaScript  │  │
│   │ (Structure)  │  │ (Style)  │  │ (Behavior)   │  │
│   └──────┬──────┘  └────┬─────┘  └──────┬───────┘  │
│          │              │               │           │
│          └──────────────┼───────────────┘           │
│                         ▼                           │
│              ┌──────────────────┐                   │
│              │   Rendered Page  │                   │
│              └────────┬─────────┘                   │
│                       │                             │
└───────────────────────┼─────────────────────────────┘
                        │ HTTP Requests (fetch/axios)
                        ▼
              ┌──────────────────┐
              │  Spring Boot API │
              │  (Your Backend)  │
              └──────────────────┘
```

### The Three Pillars

| Technology | Role | Analogy |
|------------|------|---------|
| **HTML** | Structure / content | The **warehouse floor plan** — where things go |
| **CSS** | Styling / appearance | The **paint, signage, and lighting** — how it looks |
| **JavaScript** | Behavior / interactivity | The **workers and forklifts** — making things move |

### Client-Side vs Server-Side Rendering

| | Client-Side Rendering (CSR) | Server-Side Rendering (SSR) |
|---|---|---|
| **How** | Browser downloads JS → JS builds the HTML | Server generates full HTML → sends to browser |
| **Initial Load** | Slower (blank page until JS loads) | Faster (HTML arrives ready to display) |
| **Subsequent Nav** | Very fast (no full page reload) | Slower (every navigation = new server request) |
| **SEO** | Harder (search engines may not execute JS) | Better (HTML is already there for crawlers) |
| **Server Load** | Lower (browser does the work) | Higher (server renders every page) |
| **Example** | React SPA, Angular SPA | Next.js (React SSR), Angular Universal |

**Analogy:** CSR is like giving the customer a **flat-pack furniture kit** (JavaScript) — they assemble the page themselves in the browser. SSR is like delivering **pre-assembled furniture** (HTML) — it's ready to use immediately, but the delivery truck (server) does more work.

> [!note] Why does this matter?
> As a backend developer, you're used to server-rendered pages (Thymeleaf, JSP). Modern frontends flip this — the server sends a near-empty HTML page, and JavaScript builds the entire UI in the browser.

---

## Phase 2: Single Page Applications (SPA) vs Multi-Page Applications (MPA)

### MPA — The Traditional Approach

Every click triggers a **full page reload** from the server. This is what you know from Thymeleaf/JSP.

```
User clicks "Shipments" link
        │
        ▼
Browser sends GET /shipments to server
        │
        ▼
Server renders full HTML page (header + nav + content + footer)
        │
        ▼
Browser discards old page, loads entirely new page
        │
        ▼
User sees flash of white → new page appears
```

### SPA — The Modern Approach

One HTML page is loaded **once**. JavaScript **intercepts navigation** and updates only the part of the page that changes — no full reload.

```
INITIAL LOAD:
Browser loads index.html + bundle.js (one time)
        │
        ▼
JavaScript renders the full UI

SUBSEQUENT NAVIGATION:
User clicks "Shipments" link
        │
        ▼
JavaScript intercepts the click (no server request for HTML)
        │
        ▼
JS fetches data: GET /api/shipments (JSON only)
        │
        ▼
JS updates ONLY the content area (header/nav stay intact)
        │
        ▼
URL updates to /shipments (browser history API)
        │
        ▼
User sees instant, smooth transition — no white flash
```

### Side-by-Side Comparison

```
MPA Flow:                              SPA Flow:

Browser        Server                  Browser        Server
  │               │                      │               │
  │──GET /home───→│                      │──GET /────────→│
  │←──Full HTML───│                      │←──index.html───│
  │               │                      │  + bundle.js   │
  │──GET /about──→│                      │               │
  │←──Full HTML───│                      │ (user clicks   │
  │               │                      │  "About")      │
  │──GET /ship───→│                      │               │
  │←──Full HTML───│                      │──GET /api/──→  │
  │               │                      │  about (JSON)  │
  │ (3 full page  │                      │←──{ data }────│
  │  reloads)     │                      │               │
  │               │                      │ (1 page load + │
                                         │  2 API calls)  │
```

### Pros and Cons

| Aspect | MPA | SPA |
|--------|-----|-----|
| **User Experience** | ❌ Page flashes on every navigation | ✅ Smooth, app-like feel |
| **Performance** | ❌ Redundant HTML sent every time | ✅ Only JSON data transferred after initial load |
| **Initial Load** | ✅ Fast first page (small HTML) | ❌ Slower first load (large JS bundle) |
| **SEO** | ✅ Every page has its own URL + HTML | ❌ Harder for search engines to crawl |
| **Complexity** | ✅ Simple — server does everything | ❌ Need client-side routing, state management |
| **Backend Coupling** | ❌ Tight — server generates HTML | ✅ Loose — backend is just an API |
| **Offline Support** | ❌ Not possible | ✅ Possible with service workers |

### When to Use Each

| Use Case | Recommendation |
|----------|---------------|
| Content-heavy sites (blogs, docs) | **MPA** or SSR |
| Dashboards, admin panels | **SPA** |
| E-commerce (SEO-critical) | **SSR** (Next.js, Nuxt.js) — hybrid |
| Internal enterprise tools | **SPA** (Angular or React) |
| Shipment tracking portal | **SPA** with SSR for landing pages |

> [!tip] For your use case
> If you're building internal logistics tools (dashboards, shipment management, carrier portals), SPA is almost always the right choice. SEO doesn't matter for internal tools, and the smooth UX is a huge productivity boost.

---

## Phase 3: The JavaScript Ecosystem

### Package Managers

Package managers download and manage JavaScript libraries (like Maven/Gradle for Java).

| Feature | npm | yarn | pnpm |
|---------|-----|------|------|
| **Speed** | Moderate | Fast | Fastest |
| **Disk Usage** | High (duplicates) | High | Low (content-addressed store) |
| **Lock File** | `package-lock.json` | `yarn.lock` | `pnpm-lock.yaml` |
| **Workspaces** | ✅ (v7+) | ✅ | ✅ |
| **Default With** | Node.js (bundled) | Install separately | Install separately |
| **Analogy** | Maven Central | Maven Central (faster mirror) | Maven Central (shared cache) |

> [!note] Which one?
> **npm** comes with Node.js and is the default. Start with npm. Switch to pnpm only if disk usage becomes a problem in monorepos.

### package.json — Your pom.xml / build.gradle

```json
{
  "name": "shipment-dashboard",       // Project name (like artifactId)
  "version": "1.0.0",                 // Semantic versioning (like <version>)
  "scripts": {                         // Custom commands (like Maven goals)
    "dev": "vite",                     //   npm run dev → starts dev server
    "build": "vite build",            //   npm run build → production build
    "test": "jest",                    //   npm run test → run tests
    "lint": "eslint src/"             //   npm run lint → lint code
  },
  "dependencies": {                    // Runtime deps (like <dependencies>)
    "react": "^18.2.0",               //   ^ = compatible with 18.x.x
    "axios": "^1.6.0"
  },
  "devDependencies": {                 // Build/test-only deps (like <scope>test</scope>)
    "jest": "^29.7.0",
    "eslint": "^8.56.0",
    "typescript": "^5.3.0"
  }
}
```

**Version syntax decoded:**

| Syntax | Meaning | Java Equivalent |
|--------|---------|-----------------|
| `^18.2.0` | Compatible with 18.x.x (minor + patch updates) | `[18.2.0, 19.0.0)` |
| `~18.2.0` | Patch updates only: 18.2.x | `[18.2.0, 18.3.0)` |
| `18.2.0` | Exact version only | `18.2.0` exactly |
| `*` | Any version | Not recommended |

### node_modules — Your .m2/repository

When you run `npm install`, all dependencies are downloaded into a `node_modules/` folder in your project. This folder can easily contain **10,000+ files** and hundreds of MB.

```
shipment-dashboard/
├── node_modules/          ← Downloaded dependencies (DO NOT commit)
│   ├── react/
│   ├── axios/
│   └── ... (thousands of packages)
├── package.json           ← Dependency declarations
├── package-lock.json      ← Exact resolved versions (DO commit)
└── src/                   ← Your source code
```

> [!warning] Always .gitignore node_modules
> Add `node_modules/` to your `.gitignore`. It's like committing your entire `.m2/repository` — massive, redundant, and recreatable from the lock file.

### Module Systems

JavaScript has two module systems. This is confusing but important to understand.

| Feature | CommonJS (CJS) | ES Modules (ESM) |
|---------|----------------|-------------------|
| **Syntax** | `require()` / `module.exports` | `import` / `export` |
| **Loading** | Synchronous | Asynchronous (supports tree-shaking) |
| **Used In** | Node.js (legacy), older packages | Browsers, modern Node.js, all frameworks |
| **File Extension** | `.js` or `.cjs` | `.js`, `.mjs`, or `.ts` |
| **Analogy** | Java's old classpath loading | Java's module system (Java 9+) |

```javascript
// ===== CommonJS (older, Node.js style) =====
const express = require('express');           // Import
module.exports = { startServer };             // Export

// ===== ES Modules (modern, use this) =====
import express from 'express';                // Import
export function startServer() { /* ... */ }   // Named export
export default app;                           // Default export
```

### Essential Commands

```bash
# Initialize a new project (like mvn archetype:generate)
npm init -y

# Install a runtime dependency (like adding to <dependencies>)
npm install react
npm install axios

# Install a dev-only dependency (like <scope>test</scope>)
npm install --save-dev jest
npm install --save-dev typescript

# Install all deps from package.json (like mvn dependency:resolve)
npm install

# Run scripts defined in package.json
npm run dev       # Start development server
npm run build     # Create production build
npm run test      # Run tests
```

---

## Phase 4: Build Tools

### Why Do We Need Build Tools?

Browsers understand HTML, CSS, and **vanilla JavaScript**. They don't understand:
- **JSX** (React's HTML-in-JS syntax)
- **TypeScript** (typed JavaScript)
- **SCSS/SASS** (CSS preprocessors)
- **npm imports** (`import React from 'react'` — the browser can't resolve this)

Build tools **transform** your source code into browser-compatible output.

**Analogy:** Think of a build tool as a **freight consolidation center**. Raw cargo (your source files) arrives in different formats and sizes. The consolidation center sorts, repackages, labels, and loads everything onto a single container (bundle) that's ready for shipping (the browser).

### The Build Pipeline

```
Source Files                    Build Pipeline                      Output
─────────────                   ──────────────                      ──────

  .tsx files ──┐
  .ts files  ──┤                ┌─────────────┐
  .jsx files ──┼──→ Transpile ──┤   Babel or   │
  .scss files ──┤   (convert)   │     tsc      │
  .css files ──┘                └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │   Bundle     │     ┌──────────────┐
                                │  (combine    │────→│  index.html  │
                                │   files)     │     │  bundle.js   │
                                └──────┬──────┘     │  styles.css  │
                                       │            └──────────────┘
                                       ▼
                                ┌─────────────┐
                                │   Minify     │
                                │  (compress)  │
                                └─────────────┘
```

**Each stage explained:**

| Stage | What It Does | Java Equivalent |
|-------|-------------|-----------------|
| **Transpile** | Converts JSX/TSX/modern JS → vanilla JS | Annotation processing, Lombok |
| **Bundle** | Combines 100s of files into 1-3 bundles | Creating a fat JAR |
| **Minify** | Removes whitespace, shortens variable names | ProGuard / code obfuscation |
| **Tree-shake** | Removes unused code from bundles | Dead code elimination |
| **Source Map** | Maps minified code back to source (for debugging) | Debug symbols |

### Vite vs Webpack

| Feature | Vite | Webpack |
|---------|------|---------|
| **Dev Server Start** | ⚡ <500ms (instant) | 🐌 10-30 seconds |
| **Hot Reload** | ⚡ Instant (ESM-native) | 🐌 Slower (rebundles) |
| **Config Complexity** | ✅ Minimal, convention-based | ❌ Complex, verbose config files |
| **Production Build** | Uses Rollup (fast) | Uses its own bundler |
| **Ecosystem** | Newer, growing | Mature, huge plugin ecosystem |
| **Recommendation** | ✅ Use for new projects | For legacy projects only |

> [!tip] Just use Vite
> For new projects in 2024+, Vite is the standard choice. Both React and Angular CLI support Vite. Don't waste time learning Webpack config unless you're maintaining a legacy project.

### Babel and the TypeScript Compiler

**Babel** — Transpiles modern JavaScript (ES2024) to older JavaScript (ES5) that all browsers support.

```
Your Code:         const greet = (name) => `Hello, ${name}!`;
                                    ↓ Babel
Browser-safe:      var greet = function(name) { return "Hello, " + name + "!"; };
```

**TypeScript Compiler (tsc)** — Strips type annotations and compiles TypeScript to JavaScript.

```
Your Code:         function add(a: number, b: number): number { return a + b; }
                                    ↓ tsc
Output:            function add(a, b) { return a + b; }
```

> [!note] Types are erased at build time
> TypeScript types only exist during development and compilation. The browser never sees them. This is like Java generics — `List<String>` becomes just `List` after compilation (type erasure).

---

## Phase 5: TypeScript Essentials (for Angular)

### Why TypeScript?

TypeScript adds **static typing** to JavaScript. If you know Java, you already understand the value of types — TypeScript brings that same safety to the frontend.

**Analogy:** TypeScript is to JavaScript what **labeled containers** are to unlabeled ones in a warehouse. Without labels (types), you have to open every container to see what's inside. With labels, you know immediately — and the forklift (compiler) refuses to put a container in the wrong bay.

### Key Syntax — Java Developer's Cheat Sheet

```typescript
// ===== Type Annotations =====
let shipmentId: string = "SHP-12345";       // Java: String shipmentId = "SHP-12345";
let weight: number = 450.5;                  // Java: double weight = 450.5;
let isDelivered: boolean = false;            // Java: boolean isDelivered = false;
let tags: string[] = ["fragile", "express"]; // Java: List<String> tags = List.of(...);

// ===== Interfaces (like Java interfaces, but for data shapes) =====
interface Shipment {
  id: string;                 // Required property
  origin: string;
  destination: string;
  weight: number;
  deliveredAt?: Date;         // Optional property (? means nullable)
}

// Using the interface
const shipment: Shipment = {
  id: "SHP-001",
  origin: "Chennai",
  destination: "Singapore",
  weight: 1200
  // deliveredAt is optional, so we can omit it
};

// ===== Generics (identical concept to Java) =====
function firstItem<T>(items: T[]): T | undefined {
  return items.length > 0 ? items[0] : undefined;
}

const firstShipment = firstItem<Shipment>(shipments);  // Type-safe!

// ===== Enums =====
enum ShipmentStatus {
  PENDING = "PENDING",
  IN_TRANSIT = "IN_TRANSIT",
  DELIVERED = "DELIVERED",
  CANCELLED = "CANCELLED"
}

let status: ShipmentStatus = ShipmentStatus.IN_TRANSIT;

// ===== Union Types (no Java equivalent — very powerful) =====
type TransportMode = "AIR" | "SEA" | "ROAD" | "RAIL";
let mode: TransportMode = "SEA";    // ✅ Valid
// let mode: TransportMode = "BIKE"; // ❌ Compile error!

// ===== Type Guards =====
function processId(id: string | number) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());  // TypeScript knows id is string here
  } else {
    console.log(id.toFixed(2));     // TypeScript knows id is number here
  }
}

// ===== Decorators (critical for Angular — like Spring annotations) =====
// In Angular, decorators configure classes:
@Component({
  selector: 'app-shipment-list',
  templateUrl: './shipment-list.component.html'
})
export class ShipmentListComponent {
  // This is like @Controller + @RequestMapping in Spring
}

@Injectable({ providedIn: 'root' })
export class ShipmentService {
  // This is like @Service in Spring
}
```

### Java ↔ TypeScript Comparison

| Concept | Java | TypeScript |
|---------|------|-----------|
| **Type declaration** | `String name` | `let name: string` |
| **Interface** | `interface User { }` | `interface User { }` |
| **Generics** | `List<String>` | `string[]` or `Array<string>` |
| **Enum** | `enum Status { ACTIVE }` | `enum Status { ACTIVE = "ACTIVE" }` |
| **Nullable** | `@Nullable String` | `string \| null` or `string?` |
| **Lambda** | `(x) -> x * 2` | `(x) => x * 2` |
| **Class annotation** | `@Component` | `@Component({...})` |
| **DI annotation** | `@Autowired` | `@Injectable()` + constructor injection |
| **Access modifiers** | `public`, `private`, `protected` | Same — `public`, `private`, `protected` |
| **Readonly** | `final String` | `readonly name: string` |
| **Any type** | `Object` | `any` (avoid!) or `unknown` (safer) |

### ✅ Good TypeScript Practices

```typescript
// ✅ Use strict types — avoid 'any'
function getShipment(id: string): Promise<Shipment> { ... }

// ✅ Use interfaces for data shapes
interface CarrierRate {
  carrierId: string;
  ratePerKg: number;
  currency: string;
}

// ✅ Use union types instead of magic strings
type Priority = "LOW" | "MEDIUM" | "HIGH" | "CRITICAL";
```

### ❌ Bad TypeScript Practices

```typescript
// ❌ Using 'any' defeats the purpose of TypeScript
function getShipment(id: any): any { ... }

// ❌ Not typing function parameters
function calculateRate(shipment, carrier) { ... }

// ❌ Using magic strings without types
let priority = "high";  // Typo "hgih" won't be caught
```

> [!tip] Coming from Java?
> You'll feel right at home with TypeScript. The syntax is slightly different, but the concepts — interfaces, generics, enums, access modifiers, decorators — map almost 1:1 from Java. TypeScript is the reason Angular feels familiar to backend developers.

---

## Phase 6: How Frontend Connects to Backend

### The Communication Flow

Your React/Angular frontend talks to your Spring Boot backend via **HTTP requests** — exactly the same REST APIs you already build.

```
┌──────────────────────┐         HTTP (REST)         ┌──────────────────────┐
│   Frontend (React)   │ ──────────────────────────→ │  Spring Boot API     │
│                      │                              │                      │
│  localhost:5173      │ ←────────────────────────── │  localhost:8080      │
│  (Vite dev server)   │         JSON Response        │  /api/shipments      │
└──────────────────────┘                              └──────────────────────┘
```

### fetch() — The Browser's Built-in HTTP Client

```javascript
// Fetching shipments from your Spring Boot API
// This is like calling a REST endpoint with RestTemplate/WebClient — but from the browser

async function getShipments() {
  try {
    // GET request — like restTemplate.getForEntity(url, Shipment[].class)
    const response = await fetch('http://localhost:8080/api/shipments');

    // Check if the request was successful (HTTP 2xx)
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    // Parse JSON body — like objectMapper.readValue(body, List.class)
    const shipments = await response.json();
    console.log(shipments);
    return shipments;

  } catch (error) {
    console.error('Failed to fetch shipments:', error);
  }
}

// POST request — like restTemplate.postForEntity(url, body, Shipment.class)
async function createShipment(shipment) {
  const response = await fetch('http://localhost:8080/api/shipments', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',  // Tell Spring Boot we're sending JSON
    },
    body: JSON.stringify(shipment),         // Serialize JS object to JSON string
  });
  return response.json();
}
```

### Axios — A Popular HTTP Library (like WebClient)

```javascript
import axios from 'axios';

// GET — cleaner syntax, automatic JSON parsing
const response = await axios.get('http://localhost:8080/api/shipments');
const shipments = response.data;  // Already parsed from JSON

// POST
const newShipment = await axios.post('http://localhost:8080/api/shipments', {
  origin: "Chennai",
  destination: "Singapore",
  weight: 1200
});

// Axios supports interceptors (like Spring's WebClient filters)
axios.interceptors.request.use(config => {
  config.headers.Authorization = `Bearer ${getToken()}`;
  return config;
});
```

### CORS — The #1 Headache When Connecting Frontend to Backend

When your React app (`localhost:5173`) tries to call your Spring Boot API (`localhost:8080`), the browser **blocks the request** with a CORS error:

```
Access to fetch at 'http://localhost:8080/api/shipments' from origin
'http://localhost:5173' has been blocked by CORS policy
```

**Why?** The browser enforces the **Same-Origin Policy** — JavaScript on one origin (host:port) cannot make requests to a different origin by default. This is a security feature.

```
Same Origin (✅ allowed):
  http://localhost:5173  →  http://localhost:5173/api/data

Cross Origin (❌ blocked by default):
  http://localhost:5173  →  http://localhost:8080/api/data
  ^^^^^^^^^^^^^^^^^^^^       ^^^^^^^^^^^^^^^^^^^^
  Different port = different origin!
```

**Fix it in Spring Boot:**

```java
// Option 1: Global CORS config (recommended)
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:5173")  // Your frontend's URL
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}

// Option 2: Per-controller annotation
@RestController
@CrossOrigin(origins = "http://localhost:5173")
@RequestMapping("/api/shipments")
public class ShipmentController { ... }
```

> [!warning] CORS is a browser-only restriction
> Postman and curl work fine because they don't enforce CORS. Only browsers do. This is why your API works in Postman but not from your React app.

---

## Phase 7: DOM and Virtual DOM

### What is the DOM?

The DOM (Document Object Model) is the browser's **in-memory tree representation** of your HTML. When you write HTML, the browser parses it into a tree of objects that JavaScript can read and modify.

```
HTML:                              DOM Tree:
<html>                             Document
  <body>                             └── html
    <h1>Shipments</h1>                    └── body
    <ul>                                       ├── h1
      <li>SHP-001</li>                        │   └── "Shipments"
      <li>SHP-002</li>                        └── ul
    </ul>                                          ├── li
  </body>                                         │   └── "SHP-001"
</html>                                           └── li
                                                       └── "SHP-002"
```

### Why Direct DOM Manipulation is Slow

Every time you change the DOM, the browser must:
1. **Recalculate styles** — which CSS rules apply?
2. **Reflow / Layout** — recalculate positions and sizes of all elements
3. **Repaint** — redraw pixels on screen

If you update 100 items in a list one by one, the browser does this cycle **100 times**. This is like updating 100 rows in a database one at a time instead of using a batch update.

### Virtual DOM (React's Approach)

React keeps a **lightweight copy** of the DOM in memory (the Virtual DOM). When state changes:

```
1. State changes (e.g., new shipment added)
       │
       ▼
2. React creates a NEW Virtual DOM tree
       │
       ▼
3. React DIFFS the new tree against the old tree
       │
       ▼
4. React calculates the MINIMUM set of real DOM changes
       │
       ▼
5. React applies ONLY those changes to the real DOM (batch update)
```

```
Old Virtual DOM:              New Virtual DOM:
    <ul>                          <ul>
      <li>SHP-001</li>             <li>SHP-001</li>
      <li>SHP-002</li>             <li>SHP-002</li>
    </ul>                          <li>SHP-003</li>  ← NEW
                                 </ul>

Diff result: Add one <li> node → Only ONE real DOM operation
```

**Analogy:** Instead of repainting an entire warehouse wall because one label changed, the Virtual DOM identifies exactly which label needs updating and only repaints that one spot.

### Incremental DOM (Angular's Approach)

Angular doesn't use a Virtual DOM. Instead, it compiles templates into **efficient instructions** at build time that directly update the DOM.

```
Angular Template:                  Compiled Instructions:
<h1>{{ title }}</h1>      →       if (title_changed) {
<ul>                                 h1_element.textContent = title;
  <li *ngFor="let s                }
    of shipments">                 for (let s of shipments) {
    {{ s.id }}                       if (s_changed) {
  </li>                                li_element.textContent = s.id;
</ul>                                }
                                   }
```

### Comparison

| Aspect | Virtual DOM (React) | Incremental DOM (Angular) |
|--------|-------------------|--------------------------|
| **How** | In-memory copy → diff → patch | Compiled instructions → direct updates |
| **Memory** | Higher (two trees in memory) | Lower (no copy of the DOM) |
| **Performance** | Great for frequent, scattered updates | Great for large templates |
| **Developer Experience** | More flexible (JSX = JavaScript) | More structured (templates) |

> [!note] Don't over-optimize
> Both approaches are fast enough for virtually all applications. The Virtual DOM vs Incremental DOM debate is academic for most projects. Pick the framework you're more productive with.

---

## Phase 8: Developer Tools

### Browser DevTools (F12)

Every browser ships with powerful development tools. As a frontend developer, these are your **equivalent of application logs + debugger**.

| Tab | Purpose | Backend Equivalent |
|-----|---------|-------------------|
| **Elements** | Inspect/edit HTML and CSS live | Viewing database schema |
| **Console** | JavaScript REPL, error logs, `console.log()` | Application logs (Logback/SLF4J) |
| **Network** | See all HTTP requests, headers, timing, payloads | Wireshark / Spring Boot Actuator |
| **Sources** | Set breakpoints, step through JavaScript | IntelliJ debugger |
| **Performance** | Profile rendering, identify bottlenecks | JProfiler / VisualVM |
| **Application** | View cookies, local storage, session storage | Redis / session data inspection |

### Framework-Specific DevTools

| Tool | Purpose | Install |
|------|---------|---------|
| **React DevTools** | Inspect component tree, props, state, hooks | Chrome/Firefox extension |
| **Angular DevTools** | Inspect component hierarchy, change detection, DI | Chrome extension |
| **Redux DevTools** | Time-travel debugging for Redux state | Chrome extension |

### VS Code Extensions for Frontend

| Extension | Purpose |
|-----------|---------|
| **ES7+ React/Redux Snippets** | Quick React boilerplate (`rfce` → functional component) |
| **Angular Language Service** | Autocomplete, error checking in Angular templates |
| **Prettier** | Auto-format code on save |
| **ESLint** | Lint JavaScript/TypeScript (like Checkstyle for Java) |
| **Auto Rename Tag** | Rename matching HTML tags |
| **Thunder Client** | REST client inside VS Code (like Postman) |
| **Tailwind CSS IntelliSense** | Autocomplete for Tailwind classes |

> [!tip] Must-have setup
> Install **Prettier** + **ESLint** and configure them to run on save. This is the frontend equivalent of having Checkstyle + auto-format in IntelliJ. You'll never argue about code formatting again.

---

## What's Next?

Now that you understand the fundamentals, pick your framework:

- **React path:** Continue to [[React Fundamentals]] — learn JSX, components, and hooks
- **Angular path:** Continue to [[Angular Fundamentals]] — learn TypeScript, modules, and decorators
- **Compare first:** Read [[React vs Angular Comparison]] to decide which framework suits you

> [!note] Recommendation for Spring Boot developers
> If you value structure and conventions (like Spring Boot provides), start with **Angular**. If you want a simpler learning curve and more flexibility, start with **React**. Either way, the fundamentals in this note apply to both.
