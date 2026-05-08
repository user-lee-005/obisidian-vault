---
tags:
  - typescript
  - frontend
  - fundamentals
  - type-system
  - javascript
  - spring-boot-parallel
created: 2026-05-08
status: in-progress
related:
  - "[[Angular Fundamentals]]"
  - "[[Frontend Fundamentals]]"
  - "[[Components and Templates]]"
  - "[[Services and Dependency Injection]]"
---

# TypeScript Fundamentals

> *"TypeScript is JavaScript that scales."* — TypeScript tagline

> *If Java is a language that prevents you from shooting yourself in the foot, TypeScript is JavaScript's attempt to do the same — and it gets remarkably close.*

This note is a prerequisite for [[Angular Fundamentals]]. Angular is written entirely in TypeScript, and understanding the type system is essential before diving into components, services, and the rest of the framework.

As a backend Java/Spring Boot developer, you'll find TypeScript surprisingly familiar — it has classes, interfaces, generics, access modifiers, and even decorators (like Java annotations). But it also has features Java doesn't: union types, mapped types, template literal types, and structural typing.

---

## Navigation

| Phase | Topic | Java Parallel |
|-------|-------|---------------|
| [[#Phase 1 — What is TypeScript]] | Overview and setup | — |
| [[#Phase 2 — Basic Types]] | Primitive and compound types | Java primitives and wrappers |
| [[#Phase 3 — Interfaces and Type Aliases]] | Defining object shapes | Java interfaces |
| [[#Phase 4 — Classes]] | OOP in TypeScript | Java classes |
| [[#Phase 5 — Generics]] | Type parameters | Java generics |
| [[#Phase 6 — Union and Intersection Types]] | Combining types | — (no Java equivalent) |
| [[#Phase 7 — Utility Types]] | Built-in type transformations | — |
| [[#Phase 8 — Decorators]] | Metadata annotations | Java annotations (`@`) |
| [[#Phase 9 — Advanced Types]] | Mapped, conditional, branded | — |
| [[#Phase 10 — Async Await and Promises]] | Async programming | `CompletableFuture<T>` |
| [[#Phase 11 — Modules]] | Import/export system | Java packages |
| [[#Phase 12 — tsconfig]] | Compiler configuration | `pom.xml` compiler settings |
| [[#Phase 13 — Common Mistakes]] | Pitfalls to avoid | — |
| [[#Phase 14 — Interview Questions]] | Prep for frontend interviews | — |
| [[#Phase 15 — Practice Exercises]] | Hands-on challenges | — |

---

## Phase 1 — What is TypeScript?

### 1.1 — The Short Answer

TypeScript is **JavaScript + a type system**. It's a **strict superset** of JavaScript — every valid JavaScript file is also valid TypeScript. TypeScript adds optional static types that are checked at **compile time** and then **erased** — the output is plain JavaScript that browsers can run.

```
┌──────────────────────────────────────────────────┐
│               TypeScript (.ts)                    │
│                                                  │
│   JavaScript syntax   +   Type annotations       │
│   (runs in browsers)      (erased at compile)     │
│                                                  │
│       let name = "John";                         │
│       let name: string = "John"; ← type added    │
│                                                  │
│   ┌──────────────┐                               │
│   │  tsc compile  │  → removes types             │
│   └──────┬───────┘                               │
│          ▼                                       │
│   ┌──────────────┐                               │
│   │ JavaScript    │  → runs in browser/Node.js   │
│   │ var name =    │                               │
│   │   "John";     │                               │
│   └──────────────┘                               │
└──────────────────────────────────────────────────┘
```

### 1.2 — Why TypeScript Exists

JavaScript was designed for small scripts. As apps grew to millions of lines (Google Maps, Gmail, VSCode), its lack of types became a liability:

| Problem | JavaScript | TypeScript |
|---------|-----------|-----------|
| Calling method on wrong type | Runtime crash (`undefined is not a function`) | **Compile-time error** |
| Misspelled property | Silent `undefined` | **Compile-time error** |
| Wrong argument count | Silently ignores extra args | **Compile-time error** |
| Refactoring | Find-and-replace, pray | **IDE refactoring with type safety** |
| API contracts | Read documentation, hope | **Interface definitions enforced** |
| Team collaboration | "What does this function return?" | **Type signature answers it** |

### 1.3 — TypeScript vs Java Type System

| Aspect | Java | TypeScript |
|--------|------|-----------|
| **Type checking** | Compile-time + runtime | Compile-time only (types erased) |
| **Type system** | Nominal (name-based) | **Structural** (shape-based) |
| **Null safety** | `Optional<T>`, null checks | `strictNullChecks` compiler flag |
| **Generics** | Type erasure at runtime | Type erasure at compile time |
| **Union types** | No (use inheritance) | Yes: `string \| number` |
| **Intersection types** | No (use composition) | Yes: `A & B` |
| **Type inference** | Limited (`var` in Java 10+) | Extensive (infers most types) |
| **Enums** | Full objects with methods | Simple constants (numeric or string) |
| **Access modifiers** | `public`, `private`, `protected` | Same + `readonly` |
| **Decorators** | `@Annotations` | `@Decorators` (similar concept) |

> [!warning] Structural vs Nominal Typing — The Biggest Difference
> In Java, two classes with identical fields are **different types** if they have different names. In TypeScript, two types with the same structure are **compatible** regardless of their names:
> ```typescript
> interface Dog { name: string; age: number; }
> interface Person { name: string; age: number; }
> 
> const dog: Dog = { name: "Rex", age: 5 };
> const person: Person = dog;  // ✅ This works! Same shape = same type
> ```
> In Java, `Dog` and `Person` would be incompatible even with identical fields.

### 1.4 — Setting Up TypeScript

```bash
# Install TypeScript globally
npm install -g typescript

# Check version
tsc --version

# Compile a single file
tsc hello.ts        # → generates hello.js

# Initialize a project with tsconfig.json
tsc --init

# Watch mode (recompile on changes)
tsc --watch
```

> [!tip] In Angular, You Never Run `tsc` Directly
> Angular CLI (`ng serve`, `ng build`) handles TypeScript compilation for you. But understanding `tsc` helps when debugging compiler errors.

---

## Phase 2 — Basic Types

### 2.1 — Primitive Types

```typescript
// String
let trackingNumber: string = "SHIP-001";
let greeting: string = `Hello, ${trackingNumber}`;  // Template literal

// Number (no int/float distinction — everything is a floating point)
let weight: number = 15.5;
let quantity: number = 42;
let hex: number = 0xff;

// Boolean
let isDelivered: boolean = false;

// Null and Undefined
let nothing: null = null;
let notDefined: undefined = undefined;
```

> [!warning] No `int` vs `float` in TypeScript
> Unlike Java's `int`, `long`, `float`, `double`, TypeScript has ONE number type: `number`. It's a 64-bit floating point (like Java's `double`). There's no integer type — `42` and `42.0` are the same type.

### 2.2 — Arrays and Tuples

```typescript
// Arrays — two equivalent syntaxes
let shipmentIds: number[] = [1, 2, 3];
let shipmentIds2: Array<number> = [1, 2, 3];  // Generic syntax

// Tuples — fixed-length array with specific types per position
let shipmentEntry: [string, number] = ["SHIP-001", 15.5];
// shipmentEntry[0] is string, shipmentEntry[1] is number

// Tuple with labels (TypeScript 4.0+)
type ShipmentTuple = [trackingNumber: string, weight: number, delivered: boolean];
```

### 2.3 — Enums

```typescript
// Numeric enum (default)
enum ShipmentStatus {
  PENDING,      // 0
  IN_TRANSIT,   // 1
  DELIVERED,    // 2
  CANCELLED     // 3
}

// String enum (recommended for readability)
enum ShipmentStatus {
  PENDING = 'PENDING',
  IN_TRANSIT = 'IN_TRANSIT',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED'
}

// Usage
let status: ShipmentStatus = ShipmentStatus.IN_TRANSIT;

// const enum — inlined at compile time (better performance)
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
  Left = 'LEFT',
  Right = 'RIGHT'
}
```

> [!tip] Java Enums vs TypeScript Enums
> Java enums are full classes — they can have methods, fields, and constructors. TypeScript enums are simpler — just named constants. For complex enums with behavior, use a class or object instead.
>
> Many TypeScript developers prefer **union types** over enums:
> ```typescript
> // Preferred by many TS developers
> type ShipmentStatus = 'PENDING' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELLED';
> ```

### 2.4 — Special Types: `any`, `unknown`, `never`, `void`

```typescript
// any — disables type checking (avoid!)
let data: any = 42;
data = "hello";    // ✅ No error — but defeats the purpose of TypeScript
data.foo.bar;      // ✅ No error — but will crash at runtime!

// unknown — safe version of any (must narrow before use)
let input: unknown = getUserInput();
// input.toUpperCase();      // ❌ Error — can't use unknown directly
if (typeof input === 'string') {
  input.toUpperCase();       // ✅ Now TypeScript knows it's a string
}

// void — function returns nothing (like Java's void)
function logMessage(msg: string): void {
  console.log(msg);
}

// never — function never returns (throws or infinite loop)
function throwError(msg: string): never {
  throw new Error(msg);
}

function infiniteLoop(): never {
  while (true) {}
}
```

> [!warning] `any` vs `unknown` — Always Prefer `unknown`
> `any` turns off the type checker. `unknown` is type-safe — you must narrow the type before using it. Think of `unknown` as "I don't know the type yet" and `any` as "I don't care about types." In production code, `any` should be avoided.

### 2.5 — Type Inference

TypeScript can **infer** types without explicit annotations:

```typescript
// TypeScript infers these types automatically
let name = "John";         // inferred as string
let age = 30;              // inferred as number
let active = true;         // inferred as boolean
let items = [1, 2, 3];    // inferred as number[]

// You don't need to write this:
let name: string = "John";  // redundant — TypeScript already knows
```

> [!tip] When to Annotate vs Let TypeScript Infer
> - **Let TypeScript infer:** Variable declarations, return types of simple functions
> - **Annotate explicitly:** Function parameters, public API contracts, complex objects, when inference gives a wider type than you want

### 2.6 — Type Comparison: TypeScript vs Java

| TypeScript | Java | Notes |
|-----------|------|-------|
| `string` | `String` | Always lowercase in TS |
| `number` | `int`, `long`, `float`, `double` | Single type for all numbers |
| `boolean` | `boolean` / `Boolean` | Same concept |
| `number[]` | `int[]` / `List<Integer>` | Array syntax differs |
| `[string, number]` | — | Tuples (no Java equivalent) |
| `enum` | `enum` | TS enums are simpler |
| `any` | `Object` | Disables type checking |
| `unknown` | `Object` with instanceof checks | Safe unknown type |
| `void` | `void` | Same concept |
| `never` | — | No Java equivalent |
| `null` | `null` | Same concept |
| `undefined` | — | No Java equivalent |
| `string \| number` | — | Union types (no Java equivalent) |
| `bigint` | `BigInteger` | Arbitrary precision integers |
| `symbol` | — | Unique identifiers |

---

## Phase 3 — Interfaces and Type Aliases

### 3.1 — Interfaces

Interfaces define the **shape** of an object — what properties it has and their types:

```typescript
interface Shipment {
  id: number;
  trackingNumber: string;
  status: ShipmentStatus;
  weight: number;
  origin: Address;
  destination: Address;
  notes?: string;        // Optional property (can be undefined)
  readonly createdAt: string;  // Cannot be modified after creation
}

interface Address {
  street: string;
  city: string;
  state: string;
  zipCode: string;
  country: string;
}

// Usage
const shipment: Shipment = {
  id: 1,
  trackingNumber: 'SHIP-001',
  status: 'IN_TRANSIT',
  weight: 15.5,
  origin: { street: '123 Main', city: 'Sydney', state: 'NSW', zipCode: '2000', country: 'AU' },
  destination: { street: '456 Oak', city: 'Melbourne', state: 'VIC', zipCode: '3000', country: 'AU' },
  createdAt: '2025-07-18T10:00:00Z'
  // notes is optional — can be omitted
};
```

#### Function Signatures in Interfaces

```typescript
interface ShipmentValidator {
  validate(shipment: Shipment): boolean;
  getErrors(): string[];
}

// Call signature
interface SearchFunction {
  (query: string, limit: number): Shipment[];
}
```

### 3.2 — Type Aliases

Type aliases create a **new name** for a type:

```typescript
// Simple alias
type ShipmentStatus = 'PENDING' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELLED';

// Object shape (similar to interface)
type Shipment = {
  id: number;
  trackingNumber: string;
  status: ShipmentStatus;
};

// Function type
type ShipmentFilter = (shipment: Shipment) => boolean;

// Complex union
type ApiResponse = SuccessResponse | ErrorResponse;
```

### 3.3 — Interface vs Type Alias: When to Use Which

| Feature | `interface` | `type` |
|---------|-----------|--------|
| Object shapes | ✅ Yes | ✅ Yes |
| Extends/implements | ✅ `extends` keyword | ✅ Intersection `&` |
| Union types | ❌ No | ✅ `string \| number` |
| Declaration merging | ✅ Yes (auto-merged) | ❌ No (error on duplicate) |
| Mapped types | ❌ No | ✅ Yes |
| Primitives/tuples | ❌ No | ✅ Yes |
| **Convention** | **API contracts, objects** | **Unions, utility, complex** |

> [!tip] Rule of Thumb
> - Use **`interface`** for object shapes, class contracts, and public APIs (they can be extended and merged)
> - Use **`type`** for unions, intersections, mapped types, and anything that isn't a plain object shape
>
> In Angular, **interfaces are preferred for models** (e.g., `interface Shipment { ... }`)

### 3.4 — Extending Interfaces

```typescript
// Base interface
interface Entity {
  id: number;
  createdAt: string;
  updatedAt: string;
}

// Extended interfaces — like Java's extends
interface Shipment extends Entity {
  trackingNumber: string;
  status: ShipmentStatus;
  weight: number;
}

interface Carrier extends Entity {
  name: string;
  code: string;
  active: boolean;
}

// Multiple inheritance (Java interfaces support this too)
interface AuditableShipment extends Shipment, Auditable {
  auditLog: AuditEntry[];
}
```

### 3.5 — Intersection Types

```typescript
// Intersection = combine multiple types into one
type Timestamped = { createdAt: string; updatedAt: string };
type SoftDeletable = { deletedAt?: string; isDeleted: boolean };

type AuditableEntity = Timestamped & SoftDeletable;
// Has: createdAt, updatedAt, deletedAt?, isDeleted

type AuditableShipment = Shipment & AuditableEntity;
```

### 3.6 — Declaration Merging

Interfaces can be declared multiple times — TypeScript **merges** them:

```typescript
interface Shipment {
  id: number;
  trackingNumber: string;
}

// Same name — TypeScript merges both declarations
interface Shipment {
  status: string;
  weight: number;
}

// Result: Shipment has id, trackingNumber, status, weight
```

> [!warning] Declaration Merging Pitfall
> This feature is useful for extending third-party types but can be confusing in your own code. If you declare the same interface name in different files, they silently merge. Type aliases (`type`) do NOT merge — they throw a duplicate identifier error, which is often safer.

### 3.7 — Comparison with Java Interfaces

| Aspect | Java Interface | TypeScript Interface |
|--------|---------------|---------------------|
| **Purpose** | Define contract (methods) | Define shape (properties + methods) |
| **Fields** | Constants only (`static final`) | Any properties |
| **Methods** | Abstract + default methods | Method signatures |
| **Implementation** | `implements` keyword | `implements` keyword (for classes) |
| **Checked at** | Compile + runtime (`instanceof`) | Compile only (erased) |
| **Multiple inheritance** | Yes | Yes |

---

## Phase 4 — Classes

### 4.1 — Class Syntax

```typescript
class Shipment {
  // Properties with types
  id: number;
  trackingNumber: string;
  status: ShipmentStatus;
  private weight: number;
  protected carrier: string;
  readonly createdAt: Date;

  // Constructor
  constructor(
    id: number,
    trackingNumber: string,
    weight: number,
    carrier: string
  ) {
    this.id = id;
    this.trackingNumber = trackingNumber;
    this.status = 'PENDING';
    this.weight = weight;
    this.carrier = carrier;
    this.createdAt = new Date();
  }

  // Method
  deliver(): void {
    this.status = 'DELIVERED';
  }

  // Getter (like Java's getter methods)
  getWeight(): number {
    return this.weight;
  }

  // TypeScript getter/setter syntax
  get formattedWeight(): string {
    return `${this.weight} kg`;
  }
}
```

### 4.2 — Constructor Shorthand (Parameter Properties)

TypeScript has a shortcut that eliminates repetitive property declarations:

```typescript
// ❌ Verbose — declaring properties AND assigning in constructor
class Shipment {
  id: number;
  trackingNumber: string;
  weight: number;

  constructor(id: number, trackingNumber: string, weight: number) {
    this.id = id;
    this.trackingNumber = trackingNumber;
    this.weight = weight;
  }
}

// ✅ Shorthand — access modifier in constructor auto-creates properties
class Shipment {
  constructor(
    public id: number,
    public trackingNumber: string,
    private weight: number,
    readonly createdAt: Date = new Date()
  ) {}
  // Properties are automatically declared AND assigned!
}
```

> [!tip] This is Everywhere in Angular
> Angular services use this shorthand constantly:
> ```typescript
> @Injectable({ providedIn: 'root' })
> export class ShipmentService {
>   constructor(private http: HttpClient) {}  // Auto-creates private http property
> }
> ```

### 4.3 — Access Modifiers

| Modifier | TypeScript | Java | Description |
|----------|-----------|------|-------------|
| `public` | Default (implicit) | Must declare | Accessible anywhere |
| `private` | `private` | `private` | Class only |
| `protected` | `protected` | `protected` | Class + subclasses |
| `readonly` | `readonly` | `final` | Cannot reassign after init |

> [!warning] TypeScript `private` Is Compile-Time Only
> Unlike Java, TypeScript's `private` is **not enforced at runtime**. The compiled JavaScript has no access modifiers — the property is fully accessible. For true runtime privacy, use JavaScript's `#` private fields:
> ```typescript
> class Secret {
>   #password: string;  // True runtime privacy (ES2022)
>   private token: string;  // Compile-time only (erased by tsc)
> }
> ```

### 4.4 — Abstract Classes

```typescript
abstract class Vehicle {
  constructor(public licensePlate: string) {}

  // Abstract method — must be implemented by subclasses
  abstract calculateCapacity(): number;

  // Concrete method — inherited as-is
  getDescription(): string {
    return `Vehicle ${this.licensePlate}`;
  }
}

class Truck extends Vehicle {
  constructor(
    licensePlate: string,
    private axles: number
  ) {
    super(licensePlate);
  }

  calculateCapacity(): number {
    return this.axles * 5000;  // 5000 kg per axle
  }
}

// const v = new Vehicle('ABC');  // ❌ Error — can't instantiate abstract class
const t = new Truck('TRUCK-01', 3);  // ✅
console.log(t.calculateCapacity());   // 15000
```

### 4.5 — Implements Keyword

```typescript
interface Trackable {
  getLocation(): string;
  getETA(): Date;
}

interface Loggable {
  log(message: string): void;
}

// Implement multiple interfaces — like Java
class TrackedShipment implements Trackable, Loggable {
  getLocation(): string {
    return 'Sydney, NSW';
  }

  getETA(): Date {
    return new Date('2025-08-01');
  }

  log(message: string): void {
    console.log(`[Shipment] ${message}`);
  }
}
```

### 4.6 — TypeScript vs Java Classes Comparison

| Feature | Java | TypeScript |
|---------|------|-----------|
| Constructor overloading | ✅ Multiple constructors | ❌ One constructor (use optional params) |
| Method overloading | ✅ Multiple signatures | ⚠️ Overload signatures (one implementation) |
| `static` methods/fields | ✅ Yes | ✅ Yes |
| `abstract` classes | ✅ Yes | ✅ Yes |
| `implements` | ✅ Yes | ✅ Yes |
| `extends` | ✅ Single inheritance | ✅ Single inheritance |
| `final` class | ✅ `final` keyword | ❌ No (workarounds exist) |
| Parameter properties | ❌ No | ✅ Access modifier in constructor |
| Getters/Setters | Methods by convention | `get`/`set` keywords |
| `instanceof` check | ✅ Runtime check | ✅ Runtime check (for classes) |

---

## Phase 5 — Generics

> [!quote] "Generics let you write code that works with any type while maintaining type safety — just like Java generics."

### 5.1 — Generic Functions

```typescript
// Without generics — loses type information
function first(arr: any[]): any {
  return arr[0];
}
const val = first([1, 2, 3]);  // val is 'any' — not useful

// With generics — preserves type information
function first<T>(arr: T[]): T {
  return arr[0];
}
const val = first([1, 2, 3]);        // val is 'number'
const name = first(["a", "b"]);     // name is 'string'
```

**Java equivalent:**

```java
public <T> T first(List<T> list) {
    return list.get(0);
}
```

### 5.2 — Generic Interfaces

```typescript
// Generic API response — works for any data type
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: string;
}

// Usage with different types
type ShipmentResponse = ApiResponse<Shipment>;
type CarrierListResponse = ApiResponse<Carrier[]>;
type CountResponse = ApiResponse<number>;

// Generic paginated response
interface Page<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  page: number;
  size: number;
  hasNext: boolean;
}
```

### 5.3 — Generic Classes

```typescript
class Repository<T extends { id: number }> {
  private items: Map<number, T> = new Map();

  save(item: T): void {
    this.items.set(item.id, item);
  }

  findById(id: number): T | undefined {
    return this.items.get(id);
  }

  findAll(): T[] {
    return Array.from(this.items.values());
  }

  delete(id: number): boolean {
    return this.items.delete(id);
  }
}

// Usage
const shipmentRepo = new Repository<Shipment>();
shipmentRepo.save({ id: 1, trackingNumber: 'TRK-001', status: 'PENDING', weight: 10 });
const found = shipmentRepo.findById(1);  // Shipment | undefined
```

> [!example] Spring Parallel
> This is exactly like Spring Data's `JpaRepository<T, ID>`:
> ```java
> public interface ShipmentRepository extends JpaRepository<Shipment, Long> {}
> ```
> Same concept — generic repository that works with any entity type.

### 5.4 — Generic Constraints

```typescript
// T must have a 'name' property
function getNames<T extends { name: string }>(items: T[]): string[] {
  return items.map(item => item.name);
}

// T must be a key of U
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const shipment = { trackingNumber: 'TRK-001', weight: 15.5 };
const tn = getProperty(shipment, 'trackingNumber');  // string
const wt = getProperty(shipment, 'weight');           // number
// getProperty(shipment, 'foo');  // ❌ Error — 'foo' is not a key of shipment
```

### 5.5 — Default Type Parameters

```typescript
interface Collection<T = string> {
  items: T[];
  add(item: T): void;
}

const strings: Collection = { items: [], add(item) { this.items.push(item); } };
const numbers: Collection<number> = { items: [], add(item) { this.items.push(item); } };
```

### 5.6 — TypeScript Generics vs Java Generics

| Aspect | Java | TypeScript |
|--------|------|-----------|
| Syntax | `<T>` | `<T>` (identical) |
| Constraints | `<T extends Comparable>` | `<T extends { compareTo: ... }>` |
| Type erasure | At runtime (reflection available for raw type) | At compile time (completely gone) |
| Wildcards | `? extends T`, `? super T` | No wildcards (use conditional types) |
| Multiple bounds | `<T extends A & B>` | `<T extends A & B>` (same syntax) |
| Default types | ❌ No | ✅ `<T = string>` |
| `instanceof` with generics | ❌ Can't do `instanceof List<String>` | ❌ Can't check generic types at runtime |
| Primitive types | ❌ No `List<int>` (need `Integer`) | ✅ `Array<number>` works fine |

> [!warning] No Runtime Generic Information
> Like Java, TypeScript erases generic type parameters. You cannot write `if (x instanceof Array<string>)` because `string` is erased at compile time. This matters when you need runtime type validation — see [[#Phase 7 — Utility Types]] for workarounds.

---

## Phase 6 — Union and Intersection Types

> [!info] These Are TypeScript's Superpower
> Union and intersection types have no direct Java equivalent. They let you express type relationships that are impossible in Java without inheritance hierarchies.

### 6.1 — Union Types (`A | B`)

A value that can be **one of several types**:

```typescript
// Simple union
type StringOrNumber = string | number;
let id: StringOrNumber = "abc";  // ✅
id = 42;                          // ✅
// id = true;                     // ❌ boolean is not in the union

// Literal union (like enums, but simpler)
type ShipmentStatus = 'PENDING' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELLED';
let status: ShipmentStatus = 'PENDING';  // ✅
// status = 'UNKNOWN';                   // ❌ Not in the union

// Function with union parameter
function formatId(id: string | number): string {
  if (typeof id === 'string') {
    return id.toUpperCase();   // TypeScript knows it's string here
  }
  return id.toFixed(2);       // TypeScript knows it's number here
}
```

### 6.2 — Intersection Types (`A & B`)

A value that is **ALL of the specified types combined**:

```typescript
type HasId = { id: number };
type HasTimestamps = { createdAt: string; updatedAt: string };
type HasSoftDelete = { deletedAt?: string; isDeleted: boolean };

// Intersection — must satisfy ALL types
type AuditableEntity = HasId & HasTimestamps & HasSoftDelete;

// An AuditableEntity must have: id, createdAt, updatedAt, deletedAt?, isDeleted
const entity: AuditableEntity = {
  id: 1,
  createdAt: '2025-01-01',
  updatedAt: '2025-07-18',
  isDeleted: false
};
```

### 6.3 — Discriminated Unions (Tagged Unions)

The most powerful pattern — model different states with type safety:

```typescript
// Each variant has a common 'kind' discriminator
interface PendingShipment {
  kind: 'pending';
  trackingNumber: string;
  estimatedPickup: Date;
}

interface InTransitShipment {
  kind: 'in-transit';
  trackingNumber: string;
  currentLocation: string;
  estimatedDelivery: Date;
}

interface DeliveredShipment {
  kind: 'delivered';
  trackingNumber: string;
  deliveredAt: Date;
  signature: string;
}

type Shipment = PendingShipment | InTransitShipment | DeliveredShipment;

// TypeScript narrows the type based on the discriminator
function getStatusMessage(shipment: Shipment): string {
  switch (shipment.kind) {
    case 'pending':
      return `Awaiting pickup: ${shipment.estimatedPickup}`;  // TS knows it's PendingShipment
    case 'in-transit':
      return `At ${shipment.currentLocation}, ETA: ${shipment.estimatedDelivery}`;
    case 'delivered':
      return `Delivered at ${shipment.deliveredAt}, signed by ${shipment.signature}`;
  }
}
```

> [!tip] Spring Parallel
> Discriminated unions model state machines. In Java, you'd use an enum + switch or the State pattern. In TypeScript, the compiler **exhaustively checks** that you handle every variant — if you add a new state and forget to handle it, you get a compile error.

### 6.4 — Type Guards and Narrowing

```typescript
// typeof guard — for primitives
function process(value: string | number): string {
  if (typeof value === 'string') {
    return value.toUpperCase();  // TS knows: string
  }
  return value.toFixed(2);      // TS knows: number
}

// instanceof guard — for classes
function handleError(error: Error | HttpErrorResponse): string {
  if (error instanceof HttpErrorResponse) {
    return `HTTP ${error.status}: ${error.message}`;
  }
  return error.message;
}

// 'in' operator guard — for property existence
function getArea(shape: Circle | Rectangle): number {
  if ('radius' in shape) {
    return Math.PI * shape.radius ** 2;  // TS knows: Circle
  }
  return shape.width * shape.height;     // TS knows: Rectangle
}
```

### 6.5 — Custom Type Guards (`is` keyword)

```typescript
interface Fish { swim(): void; }
interface Bird { fly(): void; }

// Custom type guard function
function isFish(animal: Fish | Bird): animal is Fish {
  return (animal as Fish).swim !== undefined;
}

// Usage
function move(animal: Fish | Bird): void {
  if (isFish(animal)) {
    animal.swim();  // TS knows it's Fish
  } else {
    animal.fly();   // TS knows it's Bird
  }
}
```

---

## Phase 7 — Utility Types

> [!info] Built-in Type Transformations
> TypeScript ships with utility types that transform existing types. Think of them as "type-level functions" — they take a type as input and return a new type.

### `Partial<T>` — Make All Properties Optional

```typescript
interface Shipment {
  id: number;
  trackingNumber: string;
  status: string;
  weight: number;
}

// Partial<Shipment> = all properties become optional
type UpdateShipmentRequest = Partial<Shipment>;
// Equivalent to:
// { id?: number; trackingNumber?: string; status?: string; weight?: number; }

// Use case: PATCH requests where you only send changed fields
function updateShipment(id: number, changes: Partial<Shipment>): void {
  this.http.patch(`/api/shipments/${id}`, changes);
}

updateShipment(1, { status: 'DELIVERED' });  // Only updating status
```

### `Required<T>` — Make All Properties Required

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

type StrictConfig = Required<Config>;
// { host: string; port: number; debug: boolean; }  — all required now
```

### `Readonly<T>` — Make All Properties Readonly

```typescript
type FrozenShipment = Readonly<Shipment>;
// All properties are readonly — can't be reassigned

const shipment: FrozenShipment = { id: 1, trackingNumber: 'TRK', status: 'PENDING', weight: 10 };
// shipment.status = 'DELIVERED';  // ❌ Error — readonly
```

### `Pick<T, K>` — Select Specific Properties

```typescript
type ShipmentSummary = Pick<Shipment, 'id' | 'trackingNumber' | 'status'>;
// { id: number; trackingNumber: string; status: string; }

// Use case: API response that only includes certain fields
```

### `Omit<T, K>` — Remove Specific Properties

```typescript
type CreateShipmentRequest = Omit<Shipment, 'id' | 'createdAt'>;
// Everything except id and createdAt (server generates these)
```

### `Record<K, V>` — Object with Known Keys

```typescript
type StatusCounts = Record<ShipmentStatus, number>;
// { PENDING: number; IN_TRANSIT: number; DELIVERED: number; CANCELLED: number; }

const counts: StatusCounts = {
  PENDING: 10,
  IN_TRANSIT: 25,
  DELIVERED: 100,
  CANCELLED: 5
};

// Also useful for dictionaries
type ShipmentMap = Record<string, Shipment>;
```

### `Extract<T, U>` and `Exclude<T, U>`

```typescript
type AllStatus = 'PENDING' | 'IN_TRANSIT' | 'DELIVERED' | 'CANCELLED' | 'ERROR';

// Extract — keep only types assignable to U
type ActiveStatus = Extract<AllStatus, 'PENDING' | 'IN_TRANSIT'>;
// 'PENDING' | 'IN_TRANSIT'

// Exclude — remove types assignable to U
type CompletedStatus = Exclude<AllStatus, 'PENDING' | 'IN_TRANSIT'>;
// 'DELIVERED' | 'CANCELLED' | 'ERROR'
```

### `NonNullable<T>` — Remove null and undefined

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;  // string
```

### `ReturnType<T>` — Get Function's Return Type

```typescript
function createShipment() {
  return { id: 1, trackingNumber: 'TRK', status: 'PENDING' };
}

type ShipmentResult = ReturnType<typeof createShipment>;
// { id: number; trackingNumber: string; status: string; }
```

### `Parameters<T>` — Get Function's Parameter Types

```typescript
function search(query: string, limit: number, offset: number): Shipment[] { ... }

type SearchParams = Parameters<typeof search>;
// [string, number, number]
```

### Utility Types Summary Table

| Utility | Input | Output | Use Case |
|---------|-------|--------|----------|
| `Partial<T>` | All required | All optional | PATCH/update requests |
| `Required<T>` | Some optional | All required | Strict configuration |
| `Readonly<T>` | Mutable | Immutable | Frozen state |
| `Pick<T, K>` | Full type | Subset | Summary/DTO objects |
| `Omit<T, K>` | Full type | Without keys | Create requests (no id) |
| `Record<K, V>` | Key + Value types | Object | Dictionaries, maps |
| `Extract<T, U>` | Union | Matching subset | Filter union members |
| `Exclude<T, U>` | Union | Non-matching subset | Exclude union members |
| `NonNullable<T>` | Nullable | Non-null | Remove null/undefined |
| `ReturnType<T>` | Function | Return type | Infer result type |
| `Parameters<T>` | Function | Param tuple | Infer argument types |

---

## Phase 8 — Decorators

> [!quote] "If you know Java annotations, you already know decorators. They look different, but the concept is identical — metadata attached to classes, methods, or properties."

### 8.1 — What Are Decorators?

Decorators are special functions prefixed with `@` that modify classes, methods, properties, or parameters at design time. Angular uses them extensively.

**Java annotations vs TypeScript decorators:**

| Java | TypeScript (Angular) | Purpose |
|------|---------------------|---------|
| `@Component` | `@Component({...})` | Define a UI component |
| `@Service` | `@Injectable({...})` | Define an injectable service |
| `@Controller` | `@Component` (with routing) | Handle user requests/UI |
| `@Autowired` | Constructor injection | Inject dependencies |
| `@RequestMapping` | Route config | Map URLs to handlers |
| `@Valid` | Form validators | Input validation |
| `@Transactional` | — | Cross-cutting concerns |

### 8.2 — Class Decorators

```typescript
// A class decorator receives the constructor and can modify or replace the class
function Logger(constructor: Function) {
  console.log(`Class created: ${constructor.name}`);
}

@Logger
class ShipmentService {
  // When this class is loaded, "Class created: ShipmentService" is logged
}

// Decorator factory — returns a decorator function (like @Component({...}))
function Component(config: { selector: string; template: string }) {
  return function(constructor: Function) {
    // Attach metadata to the class
    (constructor as any).selector = config.selector;
    (constructor as any).template = config.template;
  };
}

@Component({
  selector: 'app-shipment',
  template: '<h1>Shipments</h1>'
})
class ShipmentComponent {}
```

### 8.3 — Method Decorators

```typescript
// Log method execution time
function LogExecutionTime(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    const start = performance.now();
    const result = original.apply(this, args);
    const duration = performance.now() - start;
    console.log(`${propertyKey} took ${duration.toFixed(2)}ms`);
    return result;
  };
}

class ShipmentService {
  @LogExecutionTime
  processShipment(id: number): void {
    // ... heavy processing
  }
}
```

### 8.4 — Property and Parameter Decorators

```typescript
// Property decorator
function Required(target: any, propertyKey: string) {
  // Mark property as required (for validation)
  const requiredProps = Reflect.getMetadata('required', target) || [];
  requiredProps.push(propertyKey);
  Reflect.defineMetadata('required', requiredProps, target);
}

class CreateShipmentDto {
  @Required trackingNumber!: string;
  @Required weight!: number;
  notes?: string;
}

// Parameter decorator (used by Angular's DI)
function Inject(token: string) {
  return function(target: any, propertyKey: string | undefined, parameterIndex: number) {
    // Store injection metadata
  };
}
```

### 8.5 — How Angular Uses Decorators

```typescript
// Every Angular building block is defined with a decorator:

@Component({                    // ← Class decorator
  selector: 'app-shipment-list',
  templateUrl: './shipment-list.component.html',
  styleUrls: ['./shipment-list.component.css']
})
export class ShipmentListComponent {
  constructor(
    private shipmentService: ShipmentService  // ← DI (no decorator needed)
  ) {}
}

@Injectable({ providedIn: 'root' })  // ← Class decorator
export class ShipmentService {
  constructor(private http: HttpClient) {}
}

@NgModule({                     // ← Class decorator
  declarations: [ShipmentListComponent],
  imports: [CommonModule, HttpClientModule],
  exports: [ShipmentListComponent]
})
export class ShipmentModule {}
```

### 8.6 — experimentalDecorators vs TC39 Decorators

```typescript
// tsconfig.json — you'll see this in Angular projects
{
  "compilerOptions": {
    "experimentalDecorators": true,    // Angular's current requirement
    "emitDecoratorMetadata": true      // Needed for reflection-based DI
  }
}
```

> [!warning] Two Decorator Standards
> - **`experimentalDecorators`** — The original TypeScript decorator implementation. Angular uses this. Enabled in `tsconfig.json`.
> - **TC39 Stage 3 Decorators** — The JavaScript standard proposal (different API). Not yet used by Angular but may become the future standard.
>
> For now, always enable `experimentalDecorators` in Angular projects.

---

## Phase 9 — Advanced Types

### 9.1 — Mapped Types

Transform every property in a type:

```typescript
// Make every property optional (this is how Partial<T> works internally)
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make every property a getter function
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Shipment {
  id: number;
  status: string;
}

type ShipmentGetters = Getters<Shipment>;
// { getId: () => number; getStatus: () => string; }
```

### 9.2 — Conditional Types

Types that depend on a condition:

```typescript
// If T extends string, result is "text", otherwise "other"
type TypeName<T> = T extends string ? "text"
                 : T extends number ? "number"
                 : T extends boolean ? "boolean"
                 : "other";

type A = TypeName<string>;   // "text"
type B = TypeName<42>;       // "number"
type C = TypeName<Date>;     // "other"

// Practical: Extract the resolved type from a Promise
type Awaited<T> = T extends Promise<infer U> ? U : T;
type Result = Awaited<Promise<Shipment>>;  // Shipment
```

### 9.3 — Template Literal Types

Construct string types from other types:

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type Endpoint = '/shipments' | '/carriers' | '/routes';

// Combine them
type ApiRoute = `${HttpMethod} ${Endpoint}`;
// 'GET /shipments' | 'GET /carriers' | 'GET /routes' |
// 'POST /shipments' | 'POST /carriers' | ... (all combinations)

// Event handler pattern
type EventName = 'click' | 'hover' | 'focus';
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onHover' | 'onFocus'
```

### 9.4 — Index Signatures

For objects with dynamic keys:

```typescript
interface ShipmentMetadata {
  [key: string]: string | number | boolean;
}

const metadata: ShipmentMetadata = {
  priority: 'HIGH',
  attempts: 3,
  fragile: true,
  customField: 'any value'  // Any string key works
};
```

### 9.5 — `keyof` and `typeof` Operators

```typescript
interface Shipment {
  id: number;
  trackingNumber: string;
  status: string;
}

// keyof — gets union of property names
type ShipmentKeys = keyof Shipment;  // 'id' | 'trackingNumber' | 'status'

// typeof — gets the type of a value
const config = {
  apiUrl: 'http://localhost:8080',
  timeout: 5000,
  retries: 3
};

type Config = typeof config;
// { apiUrl: string; timeout: number; retries: number; }
```

### 9.6 — Branded Types (Nominal Typing in TS)

Since TypeScript uses structural typing, two types with the same shape are interchangeable. Branded types add a phantom property to make them distinct:

```typescript
// Problem: ShipmentId and CarrierId are both numbers — easy to mix up
type ShipmentId = number;
type CarrierId = number;

function getShipment(id: ShipmentId): Shipment { ... }
getShipment(carrierId);  // ✅ No error — but it's the wrong ID!

// Solution: Branded types
type ShipmentId = number & { __brand: 'ShipmentId' };
type CarrierId = number & { __brand: 'CarrierId' };

function createShipmentId(id: number): ShipmentId {
  return id as ShipmentId;
}

function getShipment(id: ShipmentId): Shipment { ... }

const shipId = createShipmentId(42);
const carrierId = createCarrierId(42);

getShipment(shipId);      // ✅
// getShipment(carrierId); // ❌ Error — CarrierId is not ShipmentId
```

> [!tip] Spring Parallel
> Branded types solve the same problem as Java's value objects — preventing primitive obsession. Instead of passing `long` everywhere, you'd create `ShipmentId` and `CarrierId` classes in Java.

---

## Phase 10 — Async Await and Promises

### 10.1 — Promise Typing

```typescript
// A Promise that resolves to a Shipment
function fetchShipment(id: number): Promise<Shipment> {
  return fetch(`/api/shipments/${id}`)
    .then(response => response.json());
}

// A Promise that resolves to void
async function deleteShipment(id: number): Promise<void> {
  await fetch(`/api/shipments/${id}`, { method: 'DELETE' });
}
```

### 10.2 — async/await Syntax

```typescript
// async function always returns a Promise
async function loadDashboard(): Promise<DashboardData> {
  try {
    const shipments = await fetchShipments();
    const carriers = await fetchCarriers();
    const stats = await fetchStats();

    return { shipments, carriers, stats };
  } catch (error) {
    console.error('Dashboard loading failed:', error);
    throw error;
  }
}

// Parallel execution with Promise.all
async function loadDashboardParallel(): Promise<DashboardData> {
  const [shipments, carriers, stats] = await Promise.all([
    fetchShipments(),
    fetchCarriers(),
    fetchStats()
  ]);

  return { shipments, carriers, stats };
}
```

### 10.3 — Error Handling

```typescript
async function safelyFetch<T>(url: string): Promise<T | null> {
  try {
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return await response.json() as T;
  } catch (error) {
    if (error instanceof TypeError) {
      console.error('Network error:', error.message);
    } else if (error instanceof Error) {
      console.error('API error:', error.message);
    }
    return null;
  }
}
```

### 10.4 — Comparison with Java's CompletableFuture

| TypeScript | Java | Description |
|-----------|------|-------------|
| `Promise<T>` | `CompletableFuture<T>` | Async result container |
| `async/await` | `.thenApply()` / `.join()` | Chain async operations |
| `Promise.all()` | `CompletableFuture.allOf()` | Parallel execution |
| `Promise.race()` | `CompletableFuture.anyOf()` | First to complete wins |
| `Promise.allSettled()` | — | All results (success or failure) |
| `.then()` | `.thenApply()` | Transform result |
| `.catch()` | `.exceptionally()` | Handle errors |
| `try/catch` with `await` | `try/catch` with `.join()` | Synchronous-style error handling |

> [!tip] In Angular, Prefer Observables Over Promises
> Angular's HttpClient returns `Observable<T>`, not `Promise<T>`. While you can convert with `.toPromise()` or `firstValueFrom()`, stick with Observables — they integrate better with Angular's change detection, the `async` pipe, and RxJS operators. See [[Angular HTTP Client]] and [[RxJS and Reactive Programming]].

---

## Phase 11 — Modules

### 11.1 — Import/Export Syntax

```typescript
// Named exports
export interface Shipment { id: number; trackingNumber: string; }
export type ShipmentStatus = 'PENDING' | 'IN_TRANSIT' | 'DELIVERED';
export function createShipment(data: Partial<Shipment>): Shipment { ... }
export const API_URL = '/api/shipments';

// Named imports
import { Shipment, ShipmentStatus, createShipment } from './models/shipment';

// Import with alias
import { Shipment as ShipmentModel } from './models/shipment';
```

### 11.2 — Default Exports

```typescript
// Default export — one per file
export default class ShipmentService {
  // ...
}

// Default import — any name works
import ShipmentService from './services/shipment.service';
import MyService from './services/shipment.service';  // Same class, different name
```

> [!warning] Avoid Default Exports in Angular
> Angular convention uses **named exports** exclusively. Default exports make refactoring harder (no consistent name across imports) and don't work well with Angular's dependency injection. In the Angular ecosystem, always use named exports.

### 11.3 — Re-exports (Barrel Files)

```typescript
// models/index.ts — barrel file
export { Shipment } from './shipment.model';
export { Carrier } from './carrier.model';
export { Address } from './address.model';
export { ShipmentStatus } from './shipment-status.enum';

// Clean import from the barrel
import { Shipment, Carrier, Address } from './models';
```

### 11.4 — Module Resolution

TypeScript resolves imports differently than Java:

```typescript
// Relative import — file path
import { Shipment } from './models/shipment';        // ./models/shipment.ts
import { ShipmentService } from '../services/shipment.service';

// Non-relative import — node_modules or path alias
import { HttpClient } from '@angular/common/http';   // node_modules/@angular/...
import { Observable } from 'rxjs';                    // node_modules/rxjs/...
```

### 11.5 — Comparison with Java Packages

| Concept | Java | TypeScript |
|---------|------|-----------|
| Organization unit | Package (`com.company.service`) | Module (file) |
| Import | `import com.company.Shipment;` | `import { Shipment } from './shipment';` |
| Wildcard import | `import com.company.*;` | `import * as Models from './models';` |
| Visibility | `public`, package-private | `export` (public) or no export (private) |
| Entry point | Main class | `index.ts` barrel file |
| Config | `module-info.java` | `tsconfig.json` paths |

---

## Phase 12 — tsconfig

### 12.1 — Key Compiler Options

```json
{
  "compilerOptions": {
    // Target JavaScript version
    "target": "ES2022",          // Output JS version (like -source in javac)
    "module": "ES2022",          // Module system (ESModules)
    "moduleResolution": "node",  // How to find modules

    // Strict mode (ALWAYS enable these)
    "strict": true,              // Enables all strict checks
    "strictNullChecks": true,    // null/undefined are distinct types
    "strictPropertyInitialization": true,  // Properties must be initialized
    "noImplicitAny": true,       // Can't use implicit 'any'

    // Output
    "outDir": "./dist",          // Compiled output directory
    "sourceMap": true,           // Generate .map files for debugging
    "declaration": true,         // Generate .d.ts type declaration files

    // Angular-specific
    "experimentalDecorators": true,   // Enable @Decorators
    "emitDecoratorMetadata": true,    // Emit design-time type metadata

    // Path aliases (like Maven dependency management)
    "baseUrl": "./src",
    "paths": {
      "@models/*": ["app/models/*"],
      "@services/*": ["app/services/*"],
      "@shared/*": ["app/shared/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

### 12.2 — Strict Mode Options Breakdown

| Option | What It Prevents | Java Equivalent |
|--------|-----------------|-----------------|
| `strictNullChecks` | Accessing `.` on possibly null values | `@NonNull` / `Optional<T>` |
| `noImplicitAny` | Using `any` without declaring it | Java requires explicit types |
| `strictPropertyInitialization` | Uninitialized class properties | Java requires constructor init |
| `strictFunctionTypes` | Incorrect function parameter types | Java method signature checking |
| `strictBindCallApply` | Wrong args to `.bind()`, `.call()` | — |
| `noImplicitThis` | `this` with unknown type | — |

> [!tip] Always Use `"strict": true`
> It's like enabling all warnings in the Java compiler. Without strict mode, TypeScript lets too many bugs slip through. Every professional Angular project uses strict mode.

### 12.3 — Path Aliases

```typescript
// Without path aliases — relative path hell
import { Shipment } from '../../../models/shipment.model';
import { AuthService } from '../../../../core/services/auth.service';

// With path aliases — clean imports
import { Shipment } from '@models/shipment.model';
import { AuthService } from '@services/auth.service';
```

---

## Phase 13 — Common Mistakes

> [!warning] Avoid These TypeScript Pitfalls

### ❌ Mistake 1: Using `any` Everywhere

```typescript
// BAD — defeats the purpose of TypeScript
function processData(data: any): any {
  return data.items.map((item: any) => item.name);
}

// GOOD — use proper types
function processData(data: ApiResponse<Shipment[]>): string[] {
  return data.items.map(item => item.trackingNumber);
}
```

### ❌ Mistake 2: Not Enabling Strict Mode

```json
// BAD — tsconfig.json
{ "compilerOptions": { "strict": false } }

// GOOD — catch bugs at compile time
{ "compilerOptions": { "strict": true } }
```

### ❌ Mistake 3: Ignoring Compiler Errors

```typescript
// BAD — @ts-ignore hides bugs
// @ts-ignore
const result = someObject.nonExistentMethod();

// GOOD — fix the actual type error
const result = (someObject as KnownType).existingMethod();
```

### ❌ Mistake 4: Not Understanding Structural Typing

```typescript
// This compiles even though Cat was never mentioned
interface Dog { name: string; bark(): void; }
class Cat {
  name = 'Whiskers';
  bark() { console.log('Meow'); }  // Same shape as Dog!
}

const dog: Dog = new Cat();  // ✅ Compiles — structural typing
```

### ❌ Mistake 5: Type Assertions Instead of Type Guards

```typescript
// BAD — type assertion (trust me, it's a string)
const value = someInput as string;
value.toUpperCase();  // Crashes if someInput isn't actually a string

// GOOD — type guard (verify before using)
if (typeof someInput === 'string') {
  someInput.toUpperCase();  // Safe — TypeScript verified it
}
```

### ❌ Mistake 6: Misusing Enums

```typescript
// BAD — numeric enums are error-prone
enum Status { Active, Inactive }
const s: Status = 99;  // ✅ No error — any number is a valid numeric enum!

// GOOD — use string enums or union types
enum Status { Active = 'ACTIVE', Inactive = 'INACTIVE' }
// OR
type Status = 'ACTIVE' | 'INACTIVE';
```

### ❌ Mistake 7: Not Handling null/undefined

```typescript
// BAD — crashes if user is null
function getUserName(user: User | null): string {
  return user.name;  // ❌ Object is possibly 'null'
}

// GOOD — handle null cases
function getUserName(user: User | null): string {
  return user?.name ?? 'Unknown';
}
```

### ❌ Mistake 8: Confusing `==` with `===`

```typescript
// BAD — loose equality (type coercion)
if (value == null) { }     // True for both null AND undefined
if (value == 0) { }        // True for 0, "", false, null

// GOOD — strict equality (no coercion)
if (value === null) { }    // True for null ONLY
if (value === 0) { }       // True for 0 ONLY
```

---

## Phase 14 — Interview Questions

> [!question] Q1: What is TypeScript and how does it relate to JavaScript?
> **Answer:** TypeScript is a strict superset of JavaScript that adds optional static types. Every valid JavaScript file is valid TypeScript. TypeScript compiles to plain JavaScript via `tsc`. Types exist only at compile time and are erased from the output — browsers never see TypeScript types. It's created by Microsoft and is required by Angular.

> [!question] Q2: What is the difference between `interface` and `type`?
> **Answer:** Both define object shapes. Key differences: interfaces support declaration merging (defining the same interface twice auto-merges them), while types don't. Types support union (`A | B`) and intersection (`A & B`) types, while interfaces don't. Types can alias primitives and tuples. Convention: use `interface` for object shapes and class contracts; use `type` for unions, intersections, and computed types.

> [!question] Q3: Explain `any` vs `unknown` vs `never`.
> **Answer:** `any` disables type checking entirely — any operation is allowed. `unknown` is the type-safe version — you must narrow it (via `typeof`, `instanceof`, or custom type guard) before using it. `never` represents values that should never occur — used in exhaustive switches and functions that never return (throw or infinite loop). Prefer `unknown` over `any` in production code.

> [!question] Q4: What is structural typing?
> **Answer:** TypeScript uses structural (shape-based) typing, not nominal (name-based) typing. Two types are compatible if they have the same shape (properties and methods), regardless of their names. This differs from Java, where `class Dog` and `class Person` are incompatible even with identical fields. Structural typing makes TypeScript more flexible for duck-typing patterns.

> [!question] Q5: What are generics and why are they useful?
> **Answer:** Generics let you write reusable code that works with any type while maintaining type safety. Instead of using `any`, you parameterize the type: `function first<T>(arr: T[]): T`. The type is determined at the call site. TypeScript generics are similar to Java generics — both use `<T>` syntax and support constraints (`extends`). Like Java, TypeScript erases generic types at compile time.

> [!question] Q6: Explain union types and discriminated unions.
> **Answer:** Union types (`string | number`) represent a value that can be one of several types. Discriminated unions combine union types with a common literal property (discriminator) to create type-safe tagged variants. TypeScript narrows the type automatically in `switch/if` blocks based on the discriminator. They're ideal for modeling state machines — like a shipment being 'pending', 'in-transit', or 'delivered', each with different properties.

> [!question] Q7: What are utility types? Name five and explain their use cases.
> **Answer:** Utility types are built-in type transformations. `Partial<T>` makes all properties optional (useful for PATCH updates). `Required<T>` makes all required. `Pick<T, K>` selects specific properties (summary DTOs). `Omit<T, K>` removes properties (create requests without id). `Record<K, V>` creates an object type with known keys (status → count mapping). Others include `Readonly<T>`, `ReturnType<T>`, `Extract<T, U>`, `Exclude<T, U>`.

> [!question] Q8: What are decorators and how does Angular use them?
> **Answer:** Decorators are functions prefixed with `@` that attach metadata to classes, methods, properties, or parameters — similar to Java annotations. Angular uses them extensively: `@Component` defines UI components, `@Injectable` marks classes for DI, `@NgModule` defines modules, `@Input`/`@Output` define component data flow. They require `experimentalDecorators: true` in `tsconfig.json`.

> [!question] Q9: What is the difference between `==` and `===` in TypeScript?
> **Answer:** `==` (loose equality) performs type coercion before comparison — `0 == ""` is `true`, `null == undefined` is `true`. `===` (strict equality) compares both value and type — no coercion. In TypeScript (and JavaScript), always use `===` to avoid subtle bugs. ESLint's `eqeqeq` rule enforces this.

> [!question] Q10: How does TypeScript's type system differ from Java's?
> **Answer:** Key differences: (1) TypeScript uses **structural typing** (shape-based), Java uses **nominal typing** (name-based). (2) TypeScript types are **erased at compile time** — no runtime type info. Java retains some via reflection. (3) TypeScript has **union types** (`A | B`) — Java doesn't (uses inheritance). (4) TypeScript has **type inference** everywhere — Java added `var` only in Java 10. (5) TypeScript's `private` is **compile-time only** — Java's `private` is enforced at runtime.

> [!question] Q11: What is a type guard and when would you use one?
> **Answer:** A type guard is an expression that narrows a type within a conditional block. Built-in guards: `typeof x === 'string'`, `x instanceof Date`, `'property' in x`. Custom guards use the `is` keyword: `function isFish(animal: Animal): animal is Fish`. Use them when working with union types to safely access type-specific properties without `as` assertions.

> [!question] Q12: Explain mapped types and give an example.
> **Answer:** Mapped types transform all properties of a type using a `[K in keyof T]` syntax. Example: `type Readonly<T> = { readonly [K in keyof T]: T[K] }` takes any type and makes all properties readonly. They're like a `for` loop over type properties. Advanced mapped types can rename keys using `as`: `[K in keyof T as \`get\${Capitalize<K>}\`]`. They power most utility types internally.

---

## Phase 15 — Practice Exercises

> [!example] Exercise 1: Type a REST API Response
> Define TypeScript interfaces for:
> 1. A `Shipment` with id, trackingNumber, status (union of 4 values), weight, origin/destination addresses
> 2. A `PaginatedResponse<T>` generic wrapper with content, totalPages, totalElements, page, size
> 3. An `ApiError` with status, message, timestamp, and optional fieldErrors map
>
> Create a `ShipmentService` class with methods that return the correct typed Promises.

> [!example] Exercise 2: Discriminated Union State Machine
> Model a `PaymentState` discriminated union with these variants:
> - `idle` — no properties beyond kind
> - `processing` — transactionId, startedAt
> - `succeeded` — transactionId, amount, receipt
> - `failed` — transactionId, error, retryable (boolean)
>
> Write a function `getPaymentMessage(state: PaymentState): string` that handles all variants. Ensure the compiler catches missing variants.

> [!example] Exercise 3: Generic Repository
> Create a generic `InMemoryRepository<T extends { id: number }>` class with:
> - `save(item: T): T`
> - `findById(id: number): T | undefined`
> - `findAll(): T[]`
> - `findBy(predicate: (item: T) => boolean): T[]`
> - `update(id: number, changes: Partial<T>): T | undefined`
> - `delete(id: number): boolean`
>
> Test it with both `Shipment` and `Carrier` types.

> [!example] Exercise 4: Utility Types Workshop
> Given this interface:
> ```typescript
> interface User {
>   id: number;
>   email: string;
>   name: string;
>   password: string;
>   role: 'admin' | 'user' | 'viewer';
>   createdAt: Date;
> }
> ```
> Using ONLY utility types, create:
> - `CreateUserRequest` — everything except `id` and `createdAt`
> - `UserProfile` — everything except `password`
> - `UpdateUserRequest` — all of `UserProfile`'s fields but optional
> - `UserCredentials` — only `email` and `password`
> - `AdminUser` — `User` where `role` is always `'admin'`

> [!example] Exercise 5: Type Guards
> Write type guard functions for:
> 1. `isString(value: unknown): value is string`
> 2. `isShipment(obj: unknown): obj is Shipment` — checks that obj has required properties with correct types
> 3. `isNonNullable<T>(value: T | null | undefined): value is T`
>
> Write tests that verify each guard works correctly with both valid and invalid inputs.

> [!example] Exercise 6: Decorator Implementation
> Create a `@Memoize` method decorator that:
> 1. Caches the return value based on arguments
> 2. Returns the cached value if the same arguments are passed again
> 3. Works with any method signature
>
> Apply it to a `fibonacci(n: number)` method and verify it only calculates each value once.

> [!example] Exercise 7: Async/Await Error Handling
> Create a `resilientFetch<T>` function that:
> 1. Takes a URL and options
> 2. Retries up to 3 times with exponential backoff (1s, 2s, 4s)
> 3. Returns a typed `Result<T, ApiError>` discriminated union (either success with data or failure with error)
> 4. Handles network errors, timeout errors, and HTTP errors differently
>
> Use `async/await` throughout (no `.then()` chains).

> [!example] Exercise 8: Mapped Types Deep Dive
> Create these custom mapped types:
> 1. `Nullable<T>` — makes every property `T[K] | null`
> 2. `Getters<T>` — transforms `{ name: string }` to `{ getName: () => string }`
> 3. `EventHandlers<T>` — transforms `{ click: MouseEvent }` to `{ onClick: (e: MouseEvent) => void }`
> 4. `DeepReadonly<T>` — makes all properties readonly, recursively (including nested objects)
>
> Test each with a complex nested object type.

> [!example] Exercise 9: Build a Type-Safe Event Emitter
> Create a type-safe `EventEmitter<T>` class where `T` is a record mapping event names to payload types:
> ```typescript
> type ShipmentEvents = {
>   created: { shipment: Shipment };
>   delivered: { shipmentId: number; signature: string };
>   error: { message: string; code: number };
> };
>
> const emitter = new EventEmitter<ShipmentEvents>();
> emitter.on('created', (payload) => { /* payload is { shipment: Shipment } */ });
> emitter.emit('delivered', { shipmentId: 1, signature: 'John' });
> ```
> The compiler should reject invalid event names and wrong payload types.

---

## 🔗 Related Notes

- [[Angular Fundamentals]] — Angular requires TypeScript; this is the prerequisite
- [[Frontend Fundamentals]] — JavaScript foundations before TypeScript
- [[Components and Templates]] — TypeScript in Angular component classes
- [[Services and Dependency Injection]] — TypeScript in Angular services
- [[Angular HTTP Client]] — Type-safe API calls with generics
- [[RxJS and Reactive Programming]] — Observables are heavily typed

---

> [!tip] Coming from Java?
> TypeScript will feel like a lightweight Java that runs in the browser. You'll love the type safety, generics, and interfaces. You'll be surprised by structural typing, union types, and how much the compiler can infer. The biggest adjustment is that types are **erased at compile time** — there's no reflection, no `instanceof` for interfaces, and no runtime generic type information. Embrace the compile-time safety and let go of runtime type checks.
