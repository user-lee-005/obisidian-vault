---
title: Angular Forms
phase: fundamentals
topic: forms
stack: angular
difficulty: beginner-to-intermediate
tags:
  - angular
  - forms
  - reactive-forms
  - template-driven-forms
  - validation
  - frontend
related:
  - "[[Components and Templates]]"
  - "[[Services and Dependency Injection]]"
  - "[[RxJS and Reactive Programming]]"
  - "[[Event Handling and Forms]]"
created: 2025-07-14
---

# Angular Forms

> [!quote] 💡
> *"Forms are the loading docks of your frontend — they're where data enters the system.
> Get them right, and operations run smoothly. Get them wrong, and bad data floods your warehouse."*

> [!info] 🎯 Why This Matters for a Java Developer
> If you've used `@Valid`, `BindingResult`, and Bean Validation annotations like `@NotNull`
> and `@Size` in Spring Boot, Angular forms will feel conceptually familiar.
> Angular has **two** form systems — one simple (template-driven) and one powerful (reactive).
> Reactive forms are the closest equivalent to Spring's validation + DTO pattern.

---

## Navigation — Where Are We Going?

| Phase | Topic | Spring Boot Parallel |
|-------|-------|---------------------|
| 1 | Two Approaches to Forms | `@ModelAttribute` vs `@RequestBody` + `@Valid` |
| 2 | Template-Driven Forms | Thymeleaf form binding |
| 3 | Reactive Forms (Recommended) | DTO + `@Valid` + `BindingResult` |
| 4 | Validation | Bean Validation (`@NotNull`, `@Size`, custom) |
| 5 | FormArray — Dynamic Forms | Dynamic field lists |
| 6 | Form Submission | HTTP POST to Spring Boot API |
| 7 | Angular Forms vs React Forms | Framework comparison |

---

## Phase 1: Two Approaches to Forms

> [!quote]
> *"Template-driven forms are like filling out a paper form at the counter —
> simple and quick. Reactive forms are like an automated conveyor belt system —
> more setup, but handles complexity and scale."*

Angular provides **two** distinct approaches to building forms:

### The Two Form Modules

```
┌─────────────────────────────────────────────────────────────────┐
│                     ANGULAR FORMS                               │
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐   │
│  │  Template-Driven     │    │  Reactive Forms              │   │
│  │  FormsModule         │    │  ReactiveFormsModule         │   │
│  │                      │    │                              │   │
│  │  • Logic in HTML     │    │  • Logic in TypeScript       │   │
│  │  • ngModel binding   │    │  • FormGroup / FormControl   │   │
│  │  • Simple forms      │    │  • Complex forms             │   │
│  │  • Quick prototyping │    │  • Full control & testable   │   │
│  │                      │    │                              │   │
│  │  Like Thymeleaf      │    │  Like @Valid + BindingResult │   │
│  └──────────────────────┘    └──────────────────────────────┘   │
│                                                                 │
│  ⭐ Recommendation: Use REACTIVE forms for anything non-trivial │
└─────────────────────────────────────────────────────────────────┘
```

### Decision Table — When to Use Which

| Feature | Template-Driven | Reactive |
|---------|----------------|----------|
| **Complexity** | ✅ Simple login, search | ✅ Complex multi-step forms |
| **Logic location** | HTML template | TypeScript class |
| **Data binding** | Two-way `[(ngModel)]` | One-way `[formGroup]` |
| **Validation** | HTML directives (`required`, `minlength`) | TypeScript (`Validators.required`) |
| **Testing** | ❌ Harder (needs DOM) | ✅ Easier (pure TypeScript) |
| **Dynamic fields** | ❌ Difficult | ✅ Easy with `FormArray` |
| **Form state access** | Template references | `.value`, `.valid`, `.dirty`, `.touched` |
| **Module required** | `FormsModule` | `ReactiveFormsModule` |
| **Spring parallel** | Thymeleaf `th:field` | `@Valid` + `@RequestBody` + DTO |

> [!tip] Rule of Thumb
> - **Template-driven**: Use for simple forms (login, search, quick filters)
> - **Reactive**: Use for everything else (CRUD, multi-step, dynamic, complex validation)
>
> Most Angular teams standardize on **Reactive forms** for consistency.

### Spring Boot Comparison

```
┌──────────────────────────────────────────────────────────────┐
│                SPRING BOOT FORM HANDLING                     │
│                                                              │
│  // 1. Define a DTO with validation annotations             │
│  public class ShipmentRequest {                              │
│      @NotNull @Size(min=5)                                   │
│      private String trackingId;                              │
│                                                              │
│      @NotBlank                                               │
│      private String origin;                                  │
│                                                              │
│      @Min(0.1)                                               │
│      private double weight;                                  │
│  }                                                           │
│                                                              │
│  // 2. Validate in controller                                │
│  @PostMapping("/shipments")                                  │
│  public ResponseEntity create(                               │
│      @Valid @RequestBody ShipmentRequest dto,                │
│      BindingResult result) {                                 │
│      if (result.hasErrors()) return badRequest();            │
│      // process...                                           │
│  }                                                           │
├──────────────────────────────────────────────────────────────┤
│                ANGULAR REACTIVE FORM                         │
│                                                              │
│  // 1. Define form model with validators (like DTO)          │
│  shipmentForm = this.fb.group({                              │
│    trackingId: ['', [Validators.required,                    │
│                      Validators.minLength(5)]],              │
│    origin:     ['', Validators.required],                    │
│    weight:     [0,  [Validators.required,                    │
│                      Validators.min(0.1)]]                   │
│  });                                                         │
│                                                              │
│  // 2. Validate on submit (like BindingResult)               │
│  onSubmit() {                                                │
│    if (this.shipmentForm.invalid) return;                    │
│    this.http.post('/api/shipments',                          │
│      this.shipmentForm.value);                               │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

---

## Phase 2: Template-Driven Forms

> [!quote]
> *"Template-driven forms are the paper-and-pen approach — fast for simple tasks,
> but you wouldn't use them to manage a fleet of 10,000 shipments."*

### Setup — Import FormsModule

```typescript
// app.component.ts (standalone component)
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [FormsModule],  // Required for ngModel, ngForm
  templateUrl: './app.component.html'
})
export class AppComponent {}
```

### Basic Template-Driven Form

```html
<!-- create-shipment.component.html -->
<!-- Template-driven form — logic lives primarily in the HTML template -->
<!-- Like a Thymeleaf form with th:field and th:errors -->

<form #shipmentForm="ngForm" (ngSubmit)="onSubmit(shipmentForm)">

  <!-- Tracking ID field -->
  <div class="form-group">
    <label for="trackingId">Tracking ID</label>
    <input
      id="trackingId"
      name="trackingId"
      [(ngModel)]="shipment.trackingId"
      required
      minlength="5"
      maxlength="20"
      #trackingIdField="ngModel"
    >
    <!-- Error messages — shown only after user interacts with the field -->
    <div *ngIf="trackingIdField.invalid && trackingIdField.touched" class="errors">
      <span *ngIf="trackingIdField.errors?.['required']">Tracking ID is required.</span>
      <span *ngIf="trackingIdField.errors?.['minlength']">
        Minimum {{ trackingIdField.errors?.['minlength'].requiredLength }} characters.
      </span>
    </div>
  </div>

  <!-- Origin field -->
  <div class="form-group">
    <label for="origin">Origin</label>
    <input
      id="origin"
      name="origin"
      [(ngModel)]="shipment.origin"
      required
      #originField="ngModel"
    >
    <div *ngIf="originField.invalid && originField.touched" class="errors">
      <span *ngIf="originField.errors?.['required']">Origin is required.</span>
    </div>
  </div>

  <!-- Destination field -->
  <div class="form-group">
    <label for="destination">Destination</label>
    <input
      id="destination"
      name="destination"
      [(ngModel)]="shipment.destination"
      required
      #destField="ngModel"
    >
    <div *ngIf="destField.invalid && destField.touched" class="errors">
      <span *ngIf="destField.errors?.['required']">Destination is required.</span>
    </div>
  </div>

  <!-- Weight field -->
  <div class="form-group">
    <label for="weight">Weight (kg)</label>
    <input
      id="weight"
      name="weight"
      type="number"
      [(ngModel)]="shipment.weight"
      required
      min="0.1"
      #weightField="ngModel"
    >
    <div *ngIf="weightField.invalid && weightField.touched" class="errors">
      <span *ngIf="weightField.errors?.['required']">Weight is required.</span>
      <span *ngIf="weightField.errors?.['min']">Weight must be at least 0.1 kg.</span>
    </div>
  </div>

  <!-- Submit button — disabled when form is invalid -->
  <button type="submit" [disabled]="shipmentForm.invalid">
    Create Shipment
  </button>

  <!-- Debug: show form state -->
  <pre>Form valid: {{ shipmentForm.valid }}</pre>
  <pre>Form value: {{ shipmentForm.value | json }}</pre>
</form>
```

### Component Class

```typescript
// create-shipment.component.ts
import { Component } from '@angular/core';
import { NgForm } from '@angular/forms';

@Component({
  selector: 'app-create-shipment',
  templateUrl: './create-shipment.component.html'
})
export class CreateShipmentComponent {
  // The model object — two-way bound to form fields via [(ngModel)]
  shipment = {
    trackingId: '',
    origin: '',
    destination: '',
    weight: 0
  };

  onSubmit(form: NgForm) {
    if (form.valid) {
      console.log('Submitting:', this.shipment);
      // Call API service...
    }
  }
}
```

### How Two-Way Binding Works

```
┌──────────────────────────────────────────────────────┐
│  [(ngModel)] — "Banana in a Box" Syntax              │
│                                                      │
│  [ngModel]  → Property binding (model → view)        │
│  (ngModelChange) → Event binding (view → model)      │
│                                                      │
│  [(ngModel)] = both combined = two-way binding        │
│                                                      │
│  ┌──────────┐  value   ┌──────────────┐              │
│  │ Component │ ───────► │ Input Field  │              │
│  │ .ts class │ ◄─────── │ in Template  │              │
│  └──────────┘  change   └──────────────┘              │
│                                                      │
│  When user types → model updates instantly            │
│  When model changes → input value updates             │
└──────────────────────────────────────────────────────┘
```

### Template-Driven Validation Summary

| HTML Attribute | Validation | Spring Equivalent |
|---------------|-----------|-------------------|
| `required` | Field must have a value | `@NotNull` / `@NotBlank` |
| `minlength="5"` | Minimum string length | `@Size(min=5)` |
| `maxlength="20"` | Maximum string length | `@Size(max=20)` |
| `min="0.1"` | Minimum number value | `@Min(0.1)` |
| `max="1000"` | Maximum number value | `@Max(1000)` |
| `pattern="[A-Z]+"` | Regex pattern match | `@Pattern(regexp="[A-Z]+")` |
| `email` | Email format check | `@Email` |

> [!warning] Template-Driven Limitations
> - ❌ Validation logic scattered across the HTML template
> - ❌ Hard to unit test (requires rendering the template)
> - ❌ Difficult to add/remove fields dynamically
> - ❌ Complex cross-field validation is awkward
>
> For anything beyond a simple login form, use **Reactive Forms** (Phase 3).

---

## Phase 3: Reactive Forms (Recommended)

> [!quote]
> *"Reactive forms are the automated sorting system of your warehouse —
> every field, validation rule, and state change is tracked programmatically
> in TypeScript, giving you full control over the process."*

### Setup — Import ReactiveFormsModule

```typescript
// create-shipment.component.ts (standalone)
import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators, FormArray } from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-create-shipment',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],  // Required for reactive forms
  templateUrl: './create-shipment.component.html'
})
export class CreateShipmentComponent {
  // ... form definition below
}
```

### Defining the Form Model in TypeScript

```typescript
// create-shipment.component.ts
// This is like defining a DTO with validation annotations in Spring Boot

export class CreateShipmentComponent {
  private fb = inject(FormBuilder);

  // Define the form model — like a Java DTO class with @Valid constraints
  // FormBuilder.group() creates a FormGroup (like a Java class)
  // Each property is a FormControl (like a Java field with annotations)
  shipmentForm = this.fb.group({
    // FormControl: [initialValue, [syncValidators], [asyncValidators]]
    trackingId: ['', [
      Validators.required,        // Like @NotBlank
      Validators.minLength(5),    // Like @Size(min=5)
      Validators.maxLength(20)    // Like @Size(max=20)
    ]],
    origin: ['', [
      Validators.required         // Like @NotBlank
    ]],
    destination: ['', [
      Validators.required
    ]],
    weight: [0, [
      Validators.required,        // Like @NotNull
      Validators.min(0.1)         // Like @Min(0.1)
    ]],
    carrier: ['MAERSK', [
      Validators.required
    ]],
    priority: ['STANDARD'],       // No validation — optional field
    notes: [''],                  // No validation — optional field

    // Nested FormGroup — like an embedded object in a DTO
    sender: this.fb.group({
      name: ['', Validators.required],
      email: ['', [Validators.required, Validators.email]],
      phone: ['']
    }),

    // FormArray — dynamic list of items (see Phase 5)
    items: this.fb.array([])
  });

  // Getter for easy template access to the items FormArray
  get items() {
    return this.shipmentForm.get('items') as FormArray;
  }

  // Add a new item to the dynamic list
  addItem() {
    const itemGroup = this.fb.group({
      description: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]],
      weightKg: [0, [Validators.required, Validators.min(0.01)]]
    });
    this.items.push(itemGroup);
  }

  // Remove an item by index
  removeItem(index: number) {
    this.items.removeAt(index);
  }

  onSubmit() {
    // Mark all fields as touched to trigger error display
    this.shipmentForm.markAllAsTouched();

    if (this.shipmentForm.valid) {
      const formData = this.shipmentForm.value;
      console.log('Submitting:', formData);
      // Call API: this.shipmentService.create(formData).subscribe(...)
    }
  }

  // Reset the form to initial state
  onReset() {
    this.shipmentForm.reset({
      carrier: 'MAERSK',
      priority: 'STANDARD'
    });
  }
}
```

### Template Binding for Reactive Forms

```html
<!-- create-shipment.component.html -->
<!-- Reactive form template — much cleaner than template-driven -->

<form [formGroup]="shipmentForm" (ngSubmit)="onSubmit()">

  <!-- Tracking ID -->
  <div class="form-group">
    <label for="trackingId">Tracking ID</label>
    <input id="trackingId" formControlName="trackingId">
    <div *ngIf="shipmentForm.get('trackingId')?.invalid
                && shipmentForm.get('trackingId')?.touched"
         class="errors">
      <span *ngIf="shipmentForm.get('trackingId')?.errors?.['required']">
        Tracking ID is required.
      </span>
      <span *ngIf="shipmentForm.get('trackingId')?.errors?.['minlength']">
        Minimum 5 characters.
      </span>
    </div>
  </div>

  <!-- Origin -->
  <div class="form-group">
    <label for="origin">Origin</label>
    <input id="origin" formControlName="origin">
    <div *ngIf="shipmentForm.get('origin')?.invalid
                && shipmentForm.get('origin')?.touched"
         class="errors">
      Required.
    </div>
  </div>

  <!-- Destination -->
  <div class="form-group">
    <label for="destination">Destination</label>
    <input id="destination" formControlName="destination">
  </div>

  <!-- Weight -->
  <div class="form-group">
    <label for="weight">Weight (kg)</label>
    <input id="weight" type="number" formControlName="weight">
  </div>

  <!-- Carrier dropdown -->
  <div class="form-group">
    <label for="carrier">Carrier</label>
    <select id="carrier" formControlName="carrier">
      <option value="MAERSK">Maersk</option>
      <option value="MSC">MSC</option>
      <option value="COSCO">COSCO</option>
      <option value="EVERGREEN">Evergreen</option>
    </select>
  </div>

  <!-- Nested FormGroup — sender details -->
  <fieldset formGroupName="sender">
    <legend>Sender Information</legend>
    <div class="form-group">
      <label>Name</label>
      <input formControlName="name">
    </div>
    <div class="form-group">
      <label>Email</label>
      <input type="email" formControlName="email">
    </div>
    <div class="form-group">
      <label>Phone</label>
      <input formControlName="phone">
    </div>
  </fieldset>

  <!-- Dynamic items — FormArray (see Phase 5) -->
  <fieldset formArrayName="items">
    <legend>Shipment Items</legend>
    <div *ngFor="let item of items.controls; let i = index" [formGroupName]="i">
      <input formControlName="description" placeholder="Description">
      <input type="number" formControlName="quantity" placeholder="Qty">
      <input type="number" formControlName="weightKg" placeholder="Weight">
      <button type="button" (click)="removeItem(i)">✕</button>
    </div>
    <button type="button" (click)="addItem()">+ Add Item</button>
  </fieldset>

  <!-- Submit -->
  <button type="submit" [disabled]="shipmentForm.invalid">Create Shipment</button>
  <button type="button" (click)="onReset()">Reset</button>

  <!-- Debug -->
  <pre>Valid: {{ shipmentForm.valid }}</pre>
  <pre>Value: {{ shipmentForm.value | json }}</pre>
</form>
```

### Form State Properties

Every `FormControl`, `FormGroup`, and `FormArray` has these state properties:

| Property | Type | Description | Spring Parallel |
|----------|------|-------------|-----------------|
| `.value` | `any` | Current value of the control | DTO field value |
| `.valid` | `boolean` | True if all validators pass | `!result.hasErrors()` |
| `.invalid` | `boolean` | True if any validator fails | `result.hasErrors()` |
| `.dirty` | `boolean` | True if user has changed the value | Changed from default |
| `.pristine` | `boolean` | True if user has NOT changed the value | Still at default |
| `.touched` | `boolean` | True if field has been focused then blurred | User interacted |
| `.untouched` | `boolean` | True if field has never been focused | Not yet interacted |
| `.errors` | `object \| null` | Validation error details | `FieldError` objects |
| `.enabled` | `boolean` | True if the control is active | Field is editable |
| `.disabled` | `boolean` | True if the control is inactive | Field is read-only |

### Reactive Forms Architecture Diagram

```
┌────────────────────────────────────────────────────────────┐
│                   REACTIVE FORMS ARCHITECTURE              │
│                                                            │
│  TypeScript Class              HTML Template               │
│  ┌──────────────────┐          ┌────────────────────────┐  │
│  │ FormGroup        │◄────────►│ [formGroup]="form"     │  │
│  │                  │          │                        │  │
│  │ ┌──────────────┐ │          │ ┌────────────────────┐ │  │
│  │ │ FormControl  │ │◄────────►│ │ formControlName=   │ │  │
│  │ │ "trackingId" │ │          │ │ "trackingId"       │ │  │
│  │ └──────────────┘ │          │ └────────────────────┘ │  │
│  │                  │          │                        │  │
│  │ ┌──────────────┐ │          │ ┌────────────────────┐ │  │
│  │ │ FormControl  │ │◄────────►│ │ formControlName=   │ │  │
│  │ │ "origin"     │ │          │ │ "origin"           │ │  │
│  │ └──────────────┘ │          │ └────────────────────┘ │  │
│  │                  │          │                        │  │
│  │ ┌──────────────┐ │          │ ┌────────────────────┐ │  │
│  │ │ FormArray    │ │◄────────►│ │ formArrayName=     │ │  │
│  │ │ "items"      │ │          │ │ "items"            │ │  │
│  │ └──────────────┘ │          │ └────────────────────┘ │  │
│  └──────────────────┘          └────────────────────────┘  │
│                                                            │
│  Form model defined in TS   ←→   Template binds to model   │
│  Validators in TS                 Errors displayed in HTML  │
│  Full control & testable          Clean, minimal template   │
└────────────────────────────────────────────────────────────┘
```

> [!success] Why Reactive Forms Win for Java Developers
> - ✅ Form model in TypeScript = like a DTO class
> - ✅ Validators in code = like `@NotNull`, `@Size` annotations
> - ✅ Programmatic control = like `BindingResult` error handling
> - ✅ Easy to unit test = no DOM rendering needed
> - ✅ Type-safe with `FormBuilder.group()`

---

## Phase 4: Validation

> [!quote]
> *"Validation is the quality control checkpoint at your warehouse.
> Bad data in = bad shipments out. Catch errors before they hit your API."*

### Built-in Validators

Angular provides validators that mirror Java's Bean Validation annotations:

| Angular Validator | Usage | Java Bean Validation |
|------------------|-------|---------------------|
| `Validators.required` | Field must have a value | `@NotNull` / `@NotBlank` |
| `Validators.minLength(n)` | Min string length | `@Size(min=n)` |
| `Validators.maxLength(n)` | Max string length | `@Size(max=n)` |
| `Validators.min(n)` | Min number value | `@Min(n)` |
| `Validators.max(n)` | Max number value | `@Max(n)` |
| `Validators.pattern(regex)` | Regex pattern match | `@Pattern(regexp="...")` |
| `Validators.email` | Email format | `@Email` |

### Custom Validators — Synchronous

```typescript
// validators/tracking-id.validator.ts
// Custom validator — like a custom @Constraint annotation in Java

import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

/**
 * Validates that the tracking ID follows the format: ABC-123456
 * (3 uppercase letters, dash, 6 digits)
 *
 * Spring equivalent:
 * @Constraint(validatedBy = TrackingIdValidator.class)
 * public @interface ValidTrackingId { ... }
 */
export function trackingIdValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    if (!control.value) {
      return null;  // Let 'required' validator handle empty values
    }

    const valid = /^[A-Z]{3}-\d{6}$/.test(control.value);
    // Return null if valid, or an error object if invalid
    return valid ? null : {
      invalidTrackingId: {
        value: control.value,
        message: 'Must match format: ABC-123456'
      }
    };
  };
}

// Usage in form definition:
// trackingId: ['', [Validators.required, trackingIdValidator()]]
```

### Custom Validators — Asynchronous

```typescript
// validators/unique-tracking-id.validator.ts
// Async validator — calls an API to check if the tracking ID already exists
// Like a custom validator that queries the database

import { AbstractControl, AsyncValidatorFn, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { map, catchError, debounceTime, switchMap, first } from 'rxjs/operators';
import { ShipmentService } from '../services/shipment.service';

/**
 * Checks if a tracking ID is already in use by calling the backend API.
 *
 * Spring equivalent:
 * A custom ConstraintValidator that calls a repository:
 *   @Override
 *   public boolean isValid(String trackingId, ...) {
 *       return !shipmentRepo.existsByTrackingId(trackingId);
 *   }
 */
export function uniqueTrackingIdValidator(
  shipmentService: ShipmentService
): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) {
      return of(null);
    }

    return shipmentService.checkTrackingIdExists(control.value).pipe(
      map(exists => exists ? { duplicateTrackingId: true } : null),
      catchError(() => of(null))  // On error, don't block the form
    );
  };
}

// Usage in form definition:
// trackingId: ['',
//   [Validators.required, trackingIdValidator()],         // sync validators
//   [uniqueTrackingIdValidator(this.shipmentService)]     // async validators
// ]
```

### Cross-Field Validation

```typescript
// validators/route.validator.ts
// Validates that origin and destination are different
// Like a class-level @Constraint in Java

import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

/**
 * Ensures origin ≠ destination.
 *
 * Spring equivalent:
 * @Constraint applied at class level:
 * @ValidRoute
 * public class ShipmentRequest { ... }
 */
export function routeValidator(): ValidatorFn {
  return (formGroup: AbstractControl): ValidationErrors | null => {
    const origin = formGroup.get('origin')?.value;
    const destination = formGroup.get('destination')?.value;

    if (origin && destination && origin === destination) {
      return { sameRoute: { origin, destination } };
    }
    return null;
  };
}

// Usage — apply to the FormGroup, not individual controls:
// shipmentForm = this.fb.group({
//   origin: ['', Validators.required],
//   destination: ['', Validators.required],
// }, { validators: [routeValidator()] });
```

### Displaying Validation Errors — Helper Component Pattern

```typescript
// shared/form-error.component.ts
// A reusable error display component — DRY principle

@Component({
  selector: 'app-form-error',
  standalone: true,
  template: `
    <div *ngIf="control?.invalid && control?.touched" class="error-messages">
      <ng-content></ng-content>
    </div>
  `,
  styles: [`.error-messages { color: #dc3545; font-size: 0.85rem; margin-top: 4px; }`]
})
export class FormErrorComponent {
  @Input() control: AbstractControl | null = null;
}
```

```html
<!-- Usage in a form template -->
<div class="form-group">
  <label>Tracking ID</label>
  <input formControlName="trackingId">
  <app-form-error [control]="shipmentForm.get('trackingId')">
    <span *ngIf="shipmentForm.get('trackingId')?.errors?.['required']">Required.</span>
    <span *ngIf="shipmentForm.get('trackingId')?.errors?.['invalidTrackingId']">
      Format: ABC-123456
    </span>
    <span *ngIf="shipmentForm.get('trackingId')?.errors?.['duplicateTrackingId']">
      This tracking ID already exists.
    </span>
  </app-form-error>
</div>
```

### Validation Comparison — Full Table

| Concept | Angular (Reactive) | Java Bean Validation |
|---------|-------------------|---------------------|
| Required field | `Validators.required` | `@NotNull` / `@NotBlank` |
| String length | `Validators.minLength(5)` | `@Size(min=5, max=20)` |
| Number range | `Validators.min(0.1)` | `@Min(0)` / `@DecimalMin("0.1")` |
| Pattern match | `Validators.pattern(/regex/)` | `@Pattern(regexp="...")` |
| Email format | `Validators.email` | `@Email` |
| Custom sync validator | `ValidatorFn` returning errors | `ConstraintValidator` class |
| Custom async validator | `AsyncValidatorFn` returning Observable | Custom validator calling repo |
| Cross-field validation | Validator on `FormGroup` | Class-level `@Constraint` |
| Error object structure | `{ errorKey: { details } }` | `FieldError` / `ObjectError` |

---

## Phase 5: FormArray — Dynamic Forms

> [!quote]
> *"FormArray is like a packing list — you don't know how many items
> will be in each shipment, so the form needs to grow and shrink dynamically."*

### The Problem

Some forms need a variable number of fields. For example:
- A shipment with 1–50 line items
- An invoice with multiple charge lines
- A route with multiple stops

You can't hardcode the number of fields in the template.

### FormArray Solution

```typescript
// create-shipment.component.ts — full FormArray example

import { Component, inject } from '@angular/core';
import {
  ReactiveFormsModule, FormBuilder, FormArray,
  Validators, AbstractControl
} from '@angular/forms';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-create-shipment',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  templateUrl: './create-shipment.component.html'
})
export class CreateShipmentComponent {
  private fb = inject(FormBuilder);

  shipmentForm = this.fb.group({
    trackingId: ['', [Validators.required]],
    origin: ['', [Validators.required]],
    destination: ['', [Validators.required]],

    // FormArray — a dynamic list of form groups
    items: this.fb.array([], [Validators.required]),  // At least one item

    // Another FormArray — route stops
    stops: this.fb.array([])
  });

  // ─── Items ──────────────────────────────────────────

  get items(): FormArray {
    return this.shipmentForm.get('items') as FormArray;
  }

  addItem() {
    const itemGroup = this.fb.group({
      description: ['', Validators.required],
      quantity: [1, [Validators.required, Validators.min(1)]],
      weightKg: [0.0, [Validators.required, Validators.min(0.01)]],
      hazardous: [false]
    });
    this.items.push(itemGroup);
  }

  removeItem(index: number) {
    this.items.removeAt(index);
  }

  // ─── Stops ──────────────────────────────────────────

  get stops(): FormArray {
    return this.shipmentForm.get('stops') as FormArray;
  }

  addStop() {
    const stopGroup = this.fb.group({
      location: ['', Validators.required],
      estimatedArrival: ['', Validators.required],
      type: ['PICKUP']  // PICKUP or DELIVERY
    });
    this.stops.push(stopGroup);
  }

  removeStop(index: number) {
    this.stops.removeAt(index);
  }

  // ─── Submit ─────────────────────────────────────────

  onSubmit() {
    this.shipmentForm.markAllAsTouched();
    if (this.shipmentForm.valid) {
      console.log(this.shipmentForm.value);
    }
  }
}
```

### FormArray Template

```html
<!-- create-shipment.component.html — FormArray section -->

<!-- ... other fields ... -->

<!-- Dynamic Items List -->
<fieldset>
  <legend>
    Shipment Items ({{ items.length }})
    <button type="button" (click)="addItem()" class="btn-add">+ Add Item</button>
  </legend>

  <!-- Show validation error if no items -->
  <div *ngIf="items.errors?.['required'] && items.touched" class="errors">
    At least one item is required.
  </div>

  <!-- Loop over each item in the FormArray -->
  <div formArrayName="items">
    <div *ngFor="let item of items.controls; let i = index"
         [formGroupName]="i"
         class="item-row">

      <span class="item-number">{{ i + 1 }}.</span>

      <input formControlName="description" placeholder="Item description">
      <input type="number" formControlName="quantity" placeholder="Qty" style="width:80px">
      <input type="number" formControlName="weightKg" placeholder="Weight (kg)" style="width:100px">

      <label>
        <input type="checkbox" formControlName="hazardous"> Hazardous
      </label>

      <button type="button" (click)="removeItem(i)" class="btn-remove">✕</button>

      <!-- Per-item validation errors -->
      <div *ngIf="item.get('description')?.invalid && item.get('description')?.touched"
           class="errors">
        Description is required.
      </div>
    </div>
  </div>
</fieldset>

<!-- Dynamic Stops List -->
<fieldset>
  <legend>
    Route Stops ({{ stops.length }})
    <button type="button" (click)="addStop()" class="btn-add">+ Add Stop</button>
  </legend>

  <div formArrayName="stops">
    <div *ngFor="let stop of stops.controls; let i = index"
         [formGroupName]="i"
         class="stop-row">

      <input formControlName="location" placeholder="Location">
      <input type="date" formControlName="estimatedArrival">
      <select formControlName="type">
        <option value="PICKUP">Pickup</option>
        <option value="DELIVERY">Delivery</option>
      </select>

      <button type="button" (click)="removeStop(i)" class="btn-remove">✕</button>
    </div>
  </div>
</fieldset>
```

### FormArray Visual Diagram

```
FormGroup (shipmentForm)
├── FormControl ("trackingId")
├── FormControl ("origin")
├── FormControl ("destination")
│
├── FormArray ("items")              ← dynamic list
│   ├── FormGroup [0]
│   │   ├── FormControl ("description")
│   │   ├── FormControl ("quantity")
│   │   └── FormControl ("weightKg")
│   ├── FormGroup [1]
│   │   ├── FormControl ("description")
│   │   ├── FormControl ("quantity")
│   │   └── FormControl ("weightKg")
│   └── ... (user can add/remove)
│
└── FormArray ("stops")              ← dynamic list
    ├── FormGroup [0]
    │   ├── FormControl ("location")
    │   └── FormControl ("estimatedArrival")
    └── ... (user can add/remove)
```

---

## Phase 6: Form Submission

> [!quote]
> *"Form submission is the dispatch process — you've packed the parcel (form data),
> verified the label (validation), now it's time to send it to the warehouse (API)."*

### Full HTTP Submission Flow

```typescript
// create-shipment.component.ts — complete submission with HTTP

import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { ShipmentService } from '../services/shipment.service';
import { Router } from '@angular/router';

@Component({
  selector: 'app-create-shipment',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  templateUrl: './create-shipment.component.html'
})
export class CreateShipmentComponent {
  private fb = inject(FormBuilder);
  private shipmentService = inject(ShipmentService);
  private router = inject(Router);

  // Form state flags
  isSubmitting = false;
  submitError: string | null = null;
  submitSuccess = false;

  shipmentForm = this.fb.group({
    trackingId: ['', [Validators.required, Validators.minLength(5)]],
    origin: ['', Validators.required],
    destination: ['', Validators.required],
    weight: [0, [Validators.required, Validators.min(0.1)]],
    carrier: ['MAERSK', Validators.required]
  });

  onSubmit() {
    // Step 1: Trigger all validation
    this.shipmentForm.markAllAsTouched();

    // Step 2: Check validity
    if (this.shipmentForm.invalid) {
      return;  // Don't submit — errors are shown in template
    }

    // Step 3: Set loading state
    this.isSubmitting = true;
    this.submitError = null;

    // Step 4: Call the API
    // This sends a POST request to your Spring Boot backend
    this.shipmentService.create(this.shipmentForm.value).subscribe({
      next: (response) => {
        this.isSubmitting = false;
        this.submitSuccess = true;
        // Navigate to the new shipment's detail page
        this.router.navigate(['/shipments', response.id]);
      },
      error: (err) => {
        this.isSubmitting = false;
        this.submitError = err.error?.message || 'Failed to create shipment.';
        console.error('Submission error:', err);
      }
    });
  }
}
```

### The Service Layer

```typescript
// services/shipment.service.ts
// Like a Spring @Service that calls another microservice

import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Shipment {
  id: string;
  trackingId: string;
  origin: string;
  destination: string;
  weight: number;
  carrier: string;
  status: string;
}

export interface CreateShipmentRequest {
  trackingId: string;
  origin: string;
  destination: string;
  weight: number;
  carrier: string;
}

@Injectable({ providedIn: 'root' })
export class ShipmentService {
  private http = inject(HttpClient);
  private apiUrl = '/api/shipments';

  // POST — like calling your Spring Boot @PostMapping("/shipments")
  create(data: CreateShipmentRequest): Observable<Shipment> {
    return this.http.post<Shipment>(this.apiUrl, data);
  }

  // GET all — like @GetMapping("/shipments")
  list(status?: string, page?: number): Observable<Shipment[]> {
    const params: any = {};
    if (status && status !== 'all') params.status = status;
    if (page) params.page = page;
    return this.http.get<Shipment[]>(this.apiUrl, { params });
  }

  // GET by ID — like @GetMapping("/shipments/{id}")
  getById(id: string): Observable<Shipment> {
    return this.http.get<Shipment>(`${this.apiUrl}/${id}`);
  }

  // Check if tracking ID exists — for async validation
  checkTrackingIdExists(trackingId: string): Observable<boolean> {
    return this.http.get<boolean>(
      `${this.apiUrl}/check-tracking-id?id=${trackingId}`
    );
  }
}
```

### Submission Template with States

```html
<!-- create-shipment.component.html — submit section -->

<!-- Success message -->
<div *ngIf="submitSuccess" class="alert alert-success">
  ✅ Shipment created successfully!
</div>

<!-- Error message -->
<div *ngIf="submitError" class="alert alert-error">
  ❌ {{ submitError }}
</div>

<!-- Cross-field error (if using routeValidator) -->
<div *ngIf="shipmentForm.errors?.['sameRoute']" class="alert alert-error">
  ❌ Origin and destination cannot be the same.
</div>

<form [formGroup]="shipmentForm" (ngSubmit)="onSubmit()">
  <!-- ... form fields ... -->

  <div class="form-actions">
    <button
      type="submit"
      [disabled]="isSubmitting"
      class="btn-primary">
      {{ isSubmitting ? 'Creating...' : 'Create Shipment' }}
    </button>
    <button type="button" (click)="onReset()" [disabled]="isSubmitting">
      Reset
    </button>
  </div>
</form>
```

### Full Submission Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    FORM SUBMISSION FLOW                          │
│                                                                  │
│  User fills form                                                 │
│       │                                                          │
│       ▼                                                          │
│  User clicks "Submit"                                            │
│       │                                                          │
│       ▼                                                          │
│  markAllAsTouched()  ───► Show validation errors if any          │
│       │                                                          │
│       ▼                                                          │
│  form.valid?                                                     │
│  ├── NO  → Stop. Errors are visible.                             │
│  └── YES → Continue...                                           │
│       │                                                          │
│       ▼                                                          │
│  isSubmitting = true  ───► Disable button, show "Creating..."    │
│       │                                                          │
│       ▼                                                          │
│  httpClient.post()                                               │
│       │                                                          │
│  ┌────┴────┐                                                     │
│  │         │                                                     │
│  ▼         ▼                                                     │
│ SUCCESS   ERROR                                                  │
│  │         │                                                     │
│  │         └── submitError = message                             │
│  │              isSubmitting = false                              │
│  │                                                               │
│  └── Navigate to detail page                                     │
│       isSubmitting = false                                        │
│                                                                  │
│  This mirrors the Spring Boot flow:                              │
│  @PostMapping → @Valid → BindingResult → service.save() → 201   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Phase 7: Angular Forms vs React Forms

> [!quote]
> *"Angular hands you a complete form toolkit — validators, state tracking, dynamic arrays.
> React hands you a blank canvas and says 'choose your own adventure'."*

### Fundamental Difference

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  ANGULAR:  Forms are a first-class framework feature          │
│            Built-in: FormGroup, FormControl, Validators       │
│            Two official approaches: template + reactive        │
│                                                               │
│  REACT:    No built-in form system                            │
│            Must choose a library:                             │
│            • react-hook-form (most popular)                   │
│            • formik (older, heavier)                          │
│            • Or manage state manually with useState            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Feature Comparison Table

| Feature | Angular (Reactive) | React (react-hook-form) |
|---------|-------------------|------------------------|
| **Built-in?** | ✅ Yes | ❌ No — external library |
| **Form model** | `FormGroup` / `FormControl` | `useForm()` hook |
| **Validation** | `Validators` class (built-in) | External: `yup`, `zod`, or custom |
| **Dynamic fields** | `FormArray` (built-in) | `useFieldArray()` hook |
| **Two-way binding** | `[(ngModel)]` or reactive | Controlled components via `register()` |
| **Error display** | `form.get('field')?.errors` | `errors.fieldName?.message` |
| **Submit handling** | `(ngSubmit)="onSubmit()"` | `handleSubmit(onSubmit)` |
| **Form state** | `.dirty`, `.touched`, `.valid` | `formState.isDirty`, `.isValid` |
| **Async validation** | `AsyncValidatorFn` (built-in) | Custom async validate function |
| **Testing** | Test form model directly | Test with render + fireEvent |

### Code Comparison — Same Form

```typescript
// ─── ANGULAR (Reactive Forms) ─────────────────────────

// Component class
shipmentForm = this.fb.group({
  trackingId: ['', [Validators.required, Validators.minLength(5)]],
  origin: ['', Validators.required],
  weight: [0, [Validators.required, Validators.min(0.1)]]
});

onSubmit() {
  if (this.shipmentForm.valid) {
    this.service.create(this.shipmentForm.value).subscribe();
  }
}

// Template
// <form [formGroup]="shipmentForm" (ngSubmit)="onSubmit()">
//   <input formControlName="trackingId">
//   <button type="submit">Create</button>
// </form>
```

```tsx
// ─── REACT (react-hook-form + zod) ───────────────────

// Component function
const schema = z.object({
  trackingId: z.string().min(5, 'Min 5 chars'),
  origin: z.string().min(1, 'Required'),
  weight: z.number().min(0.1, 'Min 0.1 kg')
});

function CreateShipment() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema)
  });

  const onSubmit = (data) => {
    createShipment(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('trackingId')} />
      {errors.trackingId && <span>{errors.trackingId.message}</span>}
      <button type="submit">Create</button>
    </form>
  );
}
```

### Verdict for Java Developers

| Aspect | Angular | React |
|--------|---------|-------|
| **Learning curve for Java devs** | ✅ Easier — structured, class-based | Steeper — functional, hooks-based |
| **Feels like Spring?** | ✅ Yes — DI, validators, modules | Less so — more functional style |
| **Enterprise forms** | ✅ Built-in tools for complex forms | Possible but needs library choices |
| **Flexibility** | More opinionated | More flexible |

> [!tip] Bottom Line
> If you're a Spring Boot developer building enterprise forms (CRUD, multi-step, dynamic):
> - ✅ **Angular Reactive Forms** will feel more natural — structured, validated, testable
> - React requires assembling your own form stack (`react-hook-form` + `zod` + manual state)
>
> See [[Event Handling and Forms]] for the React approach in detail.

---

## Quick Reference Cheat Sheet

```typescript
// ─── Import ───────────────────────────────────────────
import { ReactiveFormsModule, FormBuilder, Validators, FormArray } from '@angular/forms';

// ─── Define Form ──────────────────────────────────────
form = this.fb.group({
  name:  ['', Validators.required],
  email: ['', [Validators.required, Validators.email]],
  items: this.fb.array([])                      // Dynamic list
}, { validators: [crossFieldValidator()] });    // Group-level validator

// ─── Access Controls ──────────────────────────────────
this.form.get('name')?.value                    // Read value
this.form.get('name')?.valid                    // Check validity
this.form.get('name')?.errors                   // Get errors
(this.form.get('items') as FormArray).push(...) // Add to array

// ─── Template Binding ─────────────────────────────────
// <form [formGroup]="form" (ngSubmit)="submit()">
//   <input formControlName="name">
//   <fieldset formGroupName="address">...</fieldset>
//   <div formArrayName="items">...</div>
// </form>

// ─── Validation Display ──────────────────────────────
// *ngIf="form.get('name')?.invalid && form.get('name')?.touched"
// form.get('name')?.errors?.['required']
// form.get('name')?.errors?.['minlength']?.requiredLength

// ─── Custom Validator ─────────────────────────────────
function myValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    return isValid(control.value) ? null : { myError: true };
  };
}
```

---

## Cross-References

- [[Components and Templates]] — Building form components and template syntax
- [[Services and Dependency Injection]] — Services for API calls from forms
- [[RxJS and Reactive Programming]] — Observables in async validators and HTTP
- [[Angular Routing]] — Navigating after form submission
- [[Event Handling and Forms]] — React's approach to forms for comparison
- [[React vs Angular Comparison]] — Full framework comparison
