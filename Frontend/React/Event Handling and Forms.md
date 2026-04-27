---
tags:
  - react
  - events
  - forms
  - frontend
  - typescript
  - beginner
created: 2025-07-14
related:
  - "[[Components and Props]]"
  - "[[Hooks]]"
  - "[[State and Lifecycle]]"
  - "[[Angular Forms]]"
---

# Event Handling and Forms

> *"In React, you don't query the DOM and attach event listeners like jQuery. You declare what should happen, and React handles the wiring."* — React Philosophy

---

## Phase 1: Event Handling in React

### How Events Work in React

In vanilla JavaScript, you'd do `document.getElementById('btn').addEventListener('click', handler)`. In React, you **declare** event handlers directly in JSX — React wires everything up for you.

**Analogy:** In Spring Boot, you don't manually parse HTTP requests. You annotate a method with `@PostMapping` and Spring routes the request to your handler. React events work the same way — you declare `onClick={handler}` and React does the plumbing.

### Synthetic Events — Cross-Browser Consistency

React wraps native browser events in **SyntheticEvent** objects. This gives you the same API across all browsers — no more `window.event` quirks or IE compatibility headaches.

```
Browser Native Event                  React SyntheticEvent
─────────────────────                 ────────────────────
MouseEvent                    →       React.MouseEvent
InputEvent                    →       React.ChangeEvent
SubmitEvent                   →       React.FormEvent
KeyboardEvent                 →       React.KeyboardEvent
FocusEvent                    →       React.FocusEvent

All wrapped in a consistent API with:
  - e.target          → The element that triggered the event
  - e.currentTarget   → The element the handler is attached to
  - e.preventDefault() → Stop default browser behavior
  - e.stopPropagation() → Stop event bubbling
```

### Naming Convention: camelCase

```tsx
// ❌ HTML (lowercase)
<button onclick="handleClick()">Click</button>
<input onchange="handleChange()" />
<form onsubmit="handleSubmit()">

// ✅ React (camelCase)
<button onClick={handleClick}>Click</button>
<input onChange={handleChange} />
<form onSubmit={handleSubmit}>
```

> [!tip] No parentheses!
> Write `onClick={handleClick}`, **not** `onClick={handleClick()}`. With parentheses, you'd call the function immediately during render instead of passing it as a reference.

### Event Handler Patterns

```tsx
// ── Pattern 1: Inline handler (simple, one-liners) ────────
function QuickActions() {
  return (
    <button onClick={() => console.log('Shipment confirmed!')}>
      Confirm Shipment
    </button>
  );
}

// ── Pattern 2: Separate function (preferred for complex logic) ──
function ShipmentActions() {
  const handleConfirm = () => {
    // Complex logic: validate, call API, update state...
    console.log('Processing confirmation...');
  };

  return <button onClick={handleConfirm}>Confirm Shipment</button>;
}

// ── Pattern 3: With TypeScript event types ─────────────────
function CarrierSearch() {
  // Explicitly typed event — TypeScript knows e.target is an HTMLInputElement
  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    const query = e.target.value;  // TypeScript: string ✅
    console.log(`Searching carriers for: ${query}`);
  };

  return <input type="text" onChange={handleSearch} placeholder="Search carriers..." />;
}

// ── Pattern 4: Passing arguments (use arrow function wrapper) ──
function ShipmentList({ shipments }: { shipments: Shipment[] }) {
  const handleDelete = (shipmentId: string) => {
    console.log(`Deleting shipment: ${shipmentId}`);
  };

  return (
    <ul>
      {shipments.map((s) => (
        <li key={s.id}>
          {s.trackingId}
          {/* Arrow function to pass the argument */}
          <button onClick={() => handleDelete(s.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

### Common TypeScript Event Types

| Event | TypeScript Type | HTML Element | When It Fires |
|---|---|---|---|
| `onClick` | `React.MouseEvent<HTMLButtonElement>` | Button, div, etc. | User clicks |
| `onChange` | `React.ChangeEvent<HTMLInputElement>` | Input, select, textarea | Value changes |
| `onSubmit` | `React.FormEvent<HTMLFormElement>` | Form | Form submitted |
| `onKeyDown` | `React.KeyboardEvent<HTMLInputElement>` | Any focusable element | Key pressed down |
| `onFocus` | `React.FocusEvent<HTMLInputElement>` | Any focusable element | Element gains focus |
| `onBlur` | `React.FocusEvent<HTMLInputElement>` | Any focusable element | Element loses focus |
| `onMouseEnter` | `React.MouseEvent<HTMLDivElement>` | Any element | Mouse enters element |

### Preventing Default and Stopping Propagation

```tsx
function ShipmentForm() {
  // preventDefault — stop the browser from doing its default action
  // In forms: prevents page reload on submit (like return false in jQuery)
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();  // ← CRITICAL for forms! Without this, page reloads
    console.log('Form submitted via React, not browser default');
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" name="trackingId" />
      <button type="submit">Create Shipment</button>
    </form>
  );
}

// stopPropagation — prevent event from bubbling up to parent handlers
function ShipmentCard() {
  const handleCardClick = () => console.log('Card clicked');
  const handleDeleteClick = (e: React.MouseEvent) => {
    e.stopPropagation();  // Without this, card's onClick also fires
    console.log('Delete clicked — card click NOT triggered');
  };

  return (
    <div onClick={handleCardClick} style={{ padding: '1rem', border: '1px solid' }}>
      <p>Shipment SHIP-001</p>
      <button onClick={handleDeleteClick}>Delete</button>
    </div>
  );
}
```

```
Event Propagation (Bubbling)
════════════════════════════

  Click on Delete button:

  ┌─── div (onClick = handleCardClick) ───────────┐
  │                                                │
  │   ┌─── button (onClick = handleDeleteClick) ─┐ │
  │   │           ← Click HERE                   │ │
  │   └──────────────────────────────────────────┘ │
  │                                                │
  └────────────────────────────────────────────────┘

  Without stopPropagation:  button → div  (BOTH fire)
  With stopPropagation:     button only    (div's handler skipped)
```

---

## Phase 2: Controlled Components

### The Core Concept

In React, a **controlled component** is a form element whose value is **driven by React state**. The input doesn't manage its own value — React does.

**Analogy:** Think of a controlled component like a **GPS-tracked shipment**. The shipment doesn't decide its own route — the dispatch system (React state) controls where it goes, and every movement is tracked and logged. An uncontrolled component is like a shipment with no GPS — it gets there eventually, but you don't know the status until arrival.

```
Controlled Component Flow
═════════════════════════

  1. User types "M" in input
  2. onChange fires → calls setFormData({ carrier: "M" })
  3. React re-renders component
  4. Input displays value from state: "M"

  ┌────────────────────────────────────────────────┐
  │  State: { carrier: "M" }                       │
  │     ↑                        ↓                 │
  │  onChange fires          value={state.carrier}  │
  │     ↑                        ↓                 │
  │  ┌──────────────────────────────────────────┐  │
  │  │  <input value="M" onChange={handler} />  │  │
  │  └──────────────────────────────────────────┘  │
  └────────────────────────────────────────────────┘

  React is the SINGLE SOURCE OF TRUTH for the input's value.
```

### Every Form Element — Controlled

```tsx
import { useState } from 'react';

function AllInputTypes() {
  // ── Text Input ───────────────────────────────────────
  const [trackingId, setTrackingId] = useState('');

  // ── Number Input ─────────────────────────────────────
  const [weight, setWeight] = useState(0);

  // ── Checkbox ─────────────────────────────────────────
  const [isFragile, setIsFragile] = useState(false);

  // ── Radio ────────────────────────────────────────────
  const [shippingMethod, setShippingMethod] = useState('standard');

  // ── Select (Dropdown) ────────────────────────────────
  const [carrier, setCarrier] = useState('MAERSK');

  // ── Textarea ─────────────────────────────────────────
  const [notes, setNotes] = useState('');

  return (
    <form>
      {/* Text — value + onChange */}
      <input
        type="text"
        value={trackingId}
        onChange={(e) => setTrackingId(e.target.value)}
        placeholder="Tracking ID"
      />

      {/* Number — parse to number */}
      <input
        type="number"
        value={weight}
        onChange={(e) => setWeight(Number(e.target.value))}
        min={0}
      />

      {/* Checkbox — checked + onChange */}
      <label>
        <input
          type="checkbox"
          checked={isFragile}
          onChange={(e) => setIsFragile(e.target.checked)}
        />
        Fragile
      </label>

      {/* Radio — checked compares value */}
      <label>
        <input
          type="radio"
          name="shipping"
          value="standard"
          checked={shippingMethod === 'standard'}
          onChange={(e) => setShippingMethod(e.target.value)}
        />
        Standard
      </label>
      <label>
        <input
          type="radio"
          name="shipping"
          value="express"
          checked={shippingMethod === 'express'}
          onChange={(e) => setShippingMethod(e.target.value)}
        />
        Express
      </label>

      {/* Select — value + onChange */}
      <select
        value={carrier}
        onChange={(e) => setCarrier(e.target.value)}
      >
        <option value="MAERSK">Maersk</option>
        <option value="MSC">MSC</option>
        <option value="COSCO">COSCO</option>
        <option value="HAPAG">Hapag-Lloyd</option>
      </select>

      {/* Textarea — same pattern as text input */}
      <textarea
        value={notes}
        onChange={(e) => setNotes(e.target.value)}
        placeholder="Special handling instructions..."
        rows={4}
      />
    </form>
  );
}
```

### Full Example: Shipment Creation Form

This is a realistic form you'd build to submit data to your Spring Boot API.

```tsx
import { useState } from 'react';

interface ShipmentFormData {
  trackingId: string;
  origin: string;
  destination: string;
  weight: number;
  carrier: string;
  isFragile: boolean;
  notes: string;
}

const INITIAL_FORM_DATA: ShipmentFormData = {
  trackingId: '',
  origin: '',
  destination: '',
  weight: 0,
  carrier: 'MAERSK',
  isFragile: false,
  notes: '',
};

function CreateShipmentForm() {
  const [formData, setFormData] = useState<ShipmentFormData>(INITIAL_FORM_DATA);

  // Generic change handler — works for text, number, select, textarea
  // Uses computed property name [e.target.name] to update the right field
  const handleChange = (
    e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement>
  ) => {
    const { name, value, type } = e.target;

    setFormData(prev => ({
      ...prev,
      [name]: type === 'number' ? Number(value)
            : type === 'checkbox' ? (e.target as HTMLInputElement).checked
            : value,
    }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('Submitting:', formData);
    // POST to your Spring Boot API — covered in Phase 6
  };

  const handleReset = () => {
    setFormData(INITIAL_FORM_DATA);
  };

  return (
    <form onSubmit={handleSubmit}>
      <h2>Create Shipment</h2>

      <label>
        Tracking ID:
        <input
          type="text"
          name="trackingId"
          value={formData.trackingId}
          onChange={handleChange}
          required
        />
      </label>

      <label>
        Origin:
        <input
          type="text"
          name="origin"
          value={formData.origin}
          onChange={handleChange}
          required
        />
      </label>

      <label>
        Destination:
        <input
          type="text"
          name="destination"
          value={formData.destination}
          onChange={handleChange}
          required
        />
      </label>

      <label>
        Weight (kg):
        <input
          type="number"
          name="weight"
          value={formData.weight}
          onChange={handleChange}
          min={0}
          step={0.1}
        />
      </label>

      <label>
        Carrier:
        <select name="carrier" value={formData.carrier} onChange={handleChange}>
          <option value="MAERSK">Maersk</option>
          <option value="MSC">MSC</option>
          <option value="COSCO">COSCO</option>
          <option value="HAPAG">Hapag-Lloyd</option>
        </select>
      </label>

      <label>
        <input
          type="checkbox"
          name="isFragile"
          checked={formData.isFragile}
          onChange={handleChange}
        />
        Fragile Cargo
      </label>

      <label>
        Notes:
        <textarea
          name="notes"
          value={formData.notes}
          onChange={handleChange}
          rows={3}
          placeholder="Special handling instructions..."
        />
      </label>

      <div>
        <button type="submit">Create Shipment</button>
        <button type="button" onClick={handleReset}>Reset</button>
      </div>
    </form>
  );
}
```

> [!tip] The `[e.target.name]` Pattern
> Using computed property names lets you write ONE change handler for ALL fields instead of separate handlers per field. The `name` attribute on each input matches the key in your state object. This is the React equivalent of Spring's data binding — where form field names map to DTO properties.

---

## Phase 3: Uncontrolled Components

### When React Doesn't Control the Value

An **uncontrolled component** manages its own state internally. You read the value only when you need it (e.g., on submit) using a `ref`.

```
Controlled vs Uncontrolled
══════════════════════════

Controlled (React owns the value):
  State ──→ Input ──→ onChange ──→ State (loop)

Uncontrolled (DOM owns the value):
  Input manages itself
  You read via ref.current.value when needed
```

### When to Use Uncontrolled Components

| Use Case | Why |
|---|---|
| **File inputs** | `<input type="file">` is **always** uncontrolled — React can't set its value |
| **Integrating non-React libraries** | jQuery plugins, D3 charts that manage their own DOM |
| **Simple forms** that don't need real-time validation | Less boilerplate |
| **Performance-critical forms** with many fields | Avoids re-render on every keystroke |

### File Upload Example

```tsx
import { useRef } from 'react';

function ShipmentDocumentUpload() {
  // useRef to access the file input's DOM element
  const fileInputRef = useRef<HTMLInputElement>(null);

  const handleUpload = async (e: React.FormEvent) => {
    e.preventDefault();

    // Read the file from the DOM element directly
    const files = fileInputRef.current?.files;
    if (!files || files.length === 0) {
      alert('Please select a file');
      return;
    }

    // Create FormData — like MultipartFile in Spring Boot
    const formData = new FormData();
    formData.append('document', files[0]);
    formData.append('type', 'bill-of-lading');

    const response = await fetch('/api/shipments/documents', {
      method: 'POST',
      body: formData,  // No Content-Type header — browser sets it with boundary
    });

    if (response.ok) {
      console.log('Document uploaded successfully');
      // Reset the file input
      if (fileInputRef.current) fileInputRef.current.value = '';
    }
  };

  return (
    <form onSubmit={handleUpload}>
      <label>
        Upload Bill of Lading:
        {/* File inputs are ALWAYS uncontrolled — can't set value via React */}
        <input
          type="file"
          ref={fileInputRef}
          accept=".pdf,.jpg,.png"
        />
      </label>
      <button type="submit">Upload</button>
    </form>
  );
}
```

### Default Values

For uncontrolled inputs, use `defaultValue` (not `value`) to set an initial value:

```tsx
function QuickNoteForm() {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      console.log('Value:', inputRef.current?.value);
    }}>
      {/* defaultValue sets initial value; DOM manages updates */}
      <input ref={inputRef} defaultValue="SHIP-2024-" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

> [!warning] Controlled vs Uncontrolled — Don't Mix!
> Never switch an input between controlled and uncontrolled during its lifetime. Pick one approach and stick with it. React will warn you if you try to switch.

---

## Phase 4: Form Validation

### Client-Side Validation Patterns

Validation in React is manual — there's no built-in form validation framework like Bean Validation (`@NotNull`, `@Size`) in Java. You write the logic yourself.

**Analogy:** In Spring Boot, you annotate a DTO with `@Valid` and validation happens automatically. In React, you're writing the validator by hand — like implementing a `Validator<T>` interface yourself.

### Real-Time vs On-Submit Validation

| Strategy | When Errors Show | UX Feel | Implementation |
|---|---|---|---|
| **On Submit** | After clicking submit | Less noisy, but delayed feedback | Validate in `handleSubmit` |
| **On Blur** | After leaving a field | Balanced — validates when user moves on | Validate in `onBlur` handler |
| **On Change (real-time)** | While typing | Immediate but can be annoying | Validate in `onChange` handler |
| **Hybrid** | Blur first, then real-time after first error | Best UX — industry standard | Track "touched" state per field |

### Full Validation Example

```tsx
import { useState } from 'react';

interface ShipmentFormData {
  trackingId: string;
  origin: string;
  destination: string;
  weight: number;
  carrier: string;
}

// Errors map: field name → error message (empty = no error)
type FormErrors = Partial<Record<keyof ShipmentFormData, string>>;

function ValidatedShipmentForm() {
  const [formData, setFormData] = useState<ShipmentFormData>({
    trackingId: '',
    origin: '',
    destination: '',
    weight: 0,
    carrier: '',
  });
  const [errors, setErrors] = useState<FormErrors>({});
  const [touched, setTouched] = useState<Set<string>>(new Set());
  const [submitted, setSubmitted] = useState(false);

  // ── Validation function (like a Validator<ShipmentDTO> in Java) ──
  const validate = (data: ShipmentFormData): FormErrors => {
    const newErrors: FormErrors = {};

    // Tracking ID: required, must match pattern
    if (!data.trackingId.trim()) {
      newErrors.trackingId = 'Tracking ID is required';
    } else if (!/^SHIP-\d{4}-\d{3,}$/.test(data.trackingId)) {
      newErrors.trackingId = 'Format: SHIP-YYYY-NNN (e.g., SHIP-2024-001)';
    }

    // Origin: required, min length
    if (!data.origin.trim()) {
      newErrors.origin = 'Origin is required';
    } else if (data.origin.trim().length < 3) {
      newErrors.origin = 'Origin must be at least 3 characters';
    }

    // Destination: required, different from origin
    if (!data.destination.trim()) {
      newErrors.destination = 'Destination is required';
    } else if (data.destination.trim() === data.origin.trim()) {
      newErrors.destination = 'Destination must differ from origin';
    }

    // Weight: must be positive
    if (data.weight <= 0) {
      newErrors.weight = 'Weight must be a positive number';
    } else if (data.weight > 50000) {
      newErrors.weight = 'Weight cannot exceed 50,000 kg';
    }

    // Carrier: required
    if (!data.carrier) {
      newErrors.carrier = 'Please select a carrier';
    }

    return newErrors;
  };

  // ── Change handler ───────────────────────────────────────
  const handleChange = (
    e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>
  ) => {
    const { name, value, type } = e.target;
    const newValue = type === 'number' ? Number(value) : value;

    const newFormData = { ...formData, [name]: newValue };
    setFormData(newFormData);

    // If the field has been touched or form was submitted, validate on change
    if (touched.has(name) || submitted) {
      const newErrors = validate(newFormData);
      setErrors(prev => ({
        ...prev,
        [name]: newErrors[name as keyof ShipmentFormData] || undefined,
      }));
    }
  };

  // ── Blur handler (mark field as "touched") ───────────────
  const handleBlur = (e: React.FocusEvent<HTMLInputElement | HTMLSelectElement>) => {
    const { name } = e.target;
    setTouched(prev => new Set(prev).add(name));

    // Validate just this field on blur
    const fieldErrors = validate(formData);
    setErrors(prev => ({
      ...prev,
      [name]: fieldErrors[name as keyof ShipmentFormData] || undefined,
    }));
  };

  // ── Submit handler ───────────────────────────────────────
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitted(true);

    const validationErrors = validate(formData);
    setErrors(validationErrors);

    // If any errors exist, don't submit
    if (Object.keys(validationErrors).length > 0) {
      console.log('Validation failed:', validationErrors);
      return;
    }

    // All valid — submit to API
    console.log('Submitting valid data:', formData);
  };

  // ── Helper: should we show an error for this field? ──────
  const showError = (field: keyof ShipmentFormData): boolean =>
    !!(errors[field] && (touched.has(field) || submitted));

  return (
    <form onSubmit={handleSubmit} noValidate>
      <div>
        <label htmlFor="trackingId">Tracking ID:</label>
        <input
          id="trackingId"
          name="trackingId"
          value={formData.trackingId}
          onChange={handleChange}
          onBlur={handleBlur}
          className={showError('trackingId') ? 'input-error' : ''}
        />
        {showError('trackingId') && (
          <span className="error-message">{errors.trackingId}</span>
        )}
      </div>

      <div>
        <label htmlFor="origin">Origin:</label>
        <input
          id="origin"
          name="origin"
          value={formData.origin}
          onChange={handleChange}
          onBlur={handleBlur}
          className={showError('origin') ? 'input-error' : ''}
        />
        {showError('origin') && (
          <span className="error-message">{errors.origin}</span>
        )}
      </div>

      <div>
        <label htmlFor="destination">Destination:</label>
        <input
          id="destination"
          name="destination"
          value={formData.destination}
          onChange={handleChange}
          onBlur={handleBlur}
        />
        {showError('destination') && (
          <span className="error-message">{errors.destination}</span>
        )}
      </div>

      <div>
        <label htmlFor="weight">Weight (kg):</label>
        <input
          id="weight"
          name="weight"
          type="number"
          value={formData.weight}
          onChange={handleChange}
          onBlur={handleBlur}
        />
        {showError('weight') && (
          <span className="error-message">{errors.weight}</span>
        )}
      </div>

      <div>
        <label htmlFor="carrier">Carrier:</label>
        <select
          id="carrier"
          name="carrier"
          value={formData.carrier}
          onChange={handleChange}
          onBlur={handleBlur}
        >
          <option value="">-- Select Carrier --</option>
          <option value="MAERSK">Maersk</option>
          <option value="MSC">MSC</option>
          <option value="COSCO">COSCO</option>
        </select>
        {showError('carrier') && (
          <span className="error-message">{errors.carrier}</span>
        )}
      </div>

      <button type="submit">Create Shipment</button>
    </form>
  );
}
```

---

## Phase 5: Form Libraries

### Why Use a Library?

Writing validation, touched tracking, error display, and submission handling by hand is tedious — especially for forms with 10+ fields. Form libraries handle all of this with minimal code.

| Library | Philosophy | Re-renders | Bundle Size | Learning Curve |
|---|---|---|---|---|
| **React Hook Form** | Uncontrolled by default, hooks-based | ✅ Minimal | ~9 KB | Low |
| **Formik** | Controlled, render-props/hooks | ❌ Every keystroke | ~15 KB | Medium |
| **Final Form** | Subscription-based | ✅ Minimal | ~8 KB | Medium |

> [!tip] Recommendation
> Use **React Hook Form** + **Zod** for new projects. It's the modern standard — performant, TypeScript-friendly, and minimal boilerplate.

### Zod — Schema Validation (Like Bean Validation)

Zod lets you define validation schemas that look like Java's `@NotNull`, `@Size`, `@Pattern` annotations — but as code.

```tsx
// ── Java Bean Validation (for comparison) ──────────────────
// public class ShipmentDTO {
//     @NotBlank(message = "Required")
//     private String trackingId;
//
//     @Positive(message = "Must be positive")
//     private double weight;
//
//     @Size(min = 3, message = "At least 3 characters")
//     private String origin;
// }

// ── Zod Schema (the React/TypeScript equivalent) ───────────
import { z } from 'zod';

const shipmentSchema = z.object({
  trackingId: z
    .string()
    .min(1, 'Tracking ID is required')                // @NotBlank
    .regex(/^SHIP-\d{4}-\d{3,}$/, 'Format: SHIP-YYYY-NNN'),  // @Pattern

  weight: z
    .number()
    .positive('Weight must be positive')              // @Positive
    .max(50000, 'Cannot exceed 50,000 kg'),           // @Max

  origin: z
    .string()
    .min(3, 'Origin must be at least 3 characters'),  // @Size(min=3)

  destination: z
    .string()
    .min(3, 'Destination must be at least 3 characters'),

  carrier: z.enum(['MAERSK', 'MSC', 'COSCO', 'HAPAG'], {
    errorMap: () => ({ message: 'Please select a valid carrier' }),
  }),

  isFragile: z.boolean().default(false),

  notes: z.string().max(500, 'Max 500 characters').optional(),
});

// TypeScript type is INFERRED from the schema — no duplication!
type ShipmentFormData = z.infer<typeof shipmentSchema>;
```

### React Hook Form + Zod — Complete Example

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// ── Define schema ──────────────────────────────────────────
const shipmentSchema = z.object({
  trackingId: z.string().min(1, 'Required'),
  origin: z.string().min(3, 'At least 3 characters'),
  destination: z.string().min(3, 'At least 3 characters'),
  weight: z.number().positive('Must be positive'),
  carrier: z.enum(['MAERSK', 'MSC', 'COSCO', 'HAPAG']),
});

type ShipmentFormData = z.infer<typeof shipmentSchema>;

// ── Component ──────────────────────────────────────────────
function CreateShipmentForm() {
  const {
    register,      // Connects an input to the form (like @ModelAttribute binding)
    handleSubmit,  // Wraps your submit handler with validation
    formState: { errors, isSubmitting },  // Validation errors + loading state
    reset,         // Reset form to defaults
  } = useForm<ShipmentFormData>({
    resolver: zodResolver(shipmentSchema),  // Plug in Zod validation
    defaultValues: {
      trackingId: '',
      origin: '',
      destination: '',
      weight: 0,
      carrier: 'MAERSK',
    },
  });

  // This only runs if validation passes — like @Valid in Spring Controller
  const onSubmit = async (data: ShipmentFormData) => {
    console.log('Valid data:', data);
    // POST to your Spring Boot API...
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label>Tracking ID:</label>
        {/* register() returns { name, ref, onChange, onBlur } */}
        <input {...register('trackingId')} />
        {errors.trackingId && <span className="error">{errors.trackingId.message}</span>}
      </div>

      <div>
        <label>Origin:</label>
        <input {...register('origin')} />
        {errors.origin && <span className="error">{errors.origin.message}</span>}
      </div>

      <div>
        <label>Destination:</label>
        <input {...register('destination')} />
        {errors.destination && <span className="error">{errors.destination.message}</span>}
      </div>

      <div>
        <label>Weight (kg):</label>
        {/* valueAsNumber — parses string input to number automatically */}
        <input type="number" {...register('weight', { valueAsNumber: true })} />
        {errors.weight && <span className="error">{errors.weight.message}</span>}
      </div>

      <div>
        <label>Carrier:</label>
        <select {...register('carrier')}>
          <option value="MAERSK">Maersk</option>
          <option value="MSC">MSC</option>
          <option value="COSCO">COSCO</option>
          <option value="HAPAG">Hapag-Lloyd</option>
        </select>
        {errors.carrier && <span className="error">{errors.carrier.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create Shipment'}
      </button>
      <button type="button" onClick={() => reset()}>Reset</button>
    </form>
  );
}
```

> [!tip] Why React Hook Form + Zod?
> - **React Hook Form** handles form state, validation triggering, and submission — with minimal re-renders
> - **Zod** handles the validation logic — type-safe, composable, reusable schemas
> - Together, they replace ~100 lines of manual validation code with ~20 lines
> - The TypeScript type is inferred from the schema — **single source of truth**

---

## Phase 6: Form Submission

### Submitting to Your Spring Boot API

Here's the full lifecycle: user fills form → validates → POSTs to Spring Boot → handles response.

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useState } from 'react';

const schema = z.object({
  trackingId: z.string().min(1, 'Required'),
  origin: z.string().min(3),
  destination: z.string().min(3),
  weight: z.number().positive(),
  carrier: z.enum(['MAERSK', 'MSC', 'COSCO', 'HAPAG']),
});

type FormData = z.infer<typeof schema>;

// ── Response types (matching your Spring Boot DTOs) ────────
interface ApiSuccessResponse {
  id: string;
  trackingId: string;
  status: string;
  createdAt: string;
}

interface ApiErrorResponse {
  message: string;
  errors?: Record<string, string>;  // Field-level errors from @Valid
}

function CreateShipmentPage() {
  const [submitStatus, setSubmitStatus] = useState<'idle' | 'success' | 'error'>('idle');
  const [apiError, setApiError] = useState<string | null>(null);

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
    setError,  // Set errors on specific fields (e.g., from server validation)
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    setSubmitStatus('idle');
    setApiError(null);

    try {
      // POST to your Spring Boot @RestController
      const response = await fetch('/api/shipments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        // Handle validation errors from Spring Boot @Valid
        if (response.status === 400) {
          const errorBody: ApiErrorResponse = await response.json();

          // Map server-side field errors back to the form
          // Like how Spring MVC's BindingResult maps errors to form fields
          if (errorBody.errors) {
            Object.entries(errorBody.errors).forEach(([field, message]) => {
              setError(field as keyof FormData, {
                type: 'server',
                message,
              });
            });
          }
          return;
        }

        // Other errors (500, 403, etc.)
        throw new Error(`Server error: ${response.status}`);
      }

      const result: ApiSuccessResponse = await response.json();
      console.log('Shipment created:', result);
      setSubmitStatus('success');
      reset();  // Clear the form on success
    } catch (err) {
      setSubmitStatus('error');
      setApiError(err instanceof Error ? err.message : 'Unknown error');
    }
  };

  return (
    <div>
      <h1>Create Shipment</h1>

      {/* Success message */}
      {submitStatus === 'success' && (
        <div className="alert alert-success">
          ✅ Shipment created successfully!
        </div>
      )}

      {/* Global error message */}
      {submitStatus === 'error' && (
        <div className="alert alert-error">
          ❌ Failed to create shipment: {apiError}
        </div>
      )}

      <form onSubmit={handleSubmit(onSubmit)}>
        <div>
          <label>Tracking ID:</label>
          <input {...register('trackingId')} />
          {errors.trackingId && <span className="error">{errors.trackingId.message}</span>}
        </div>

        <div>
          <label>Origin:</label>
          <input {...register('origin')} />
          {errors.origin && <span className="error">{errors.origin.message}</span>}
        </div>

        <div>
          <label>Destination:</label>
          <input {...register('destination')} />
          {errors.destination && <span className="error">{errors.destination.message}</span>}
        </div>

        <div>
          <label>Weight:</label>
          <input type="number" step="0.1" {...register('weight', { valueAsNumber: true })} />
          {errors.weight && <span className="error">{errors.weight.message}</span>}
        </div>

        <div>
          <label>Carrier:</label>
          <select {...register('carrier')}>
            <option value="MAERSK">Maersk</option>
            <option value="MSC">MSC</option>
            <option value="COSCO">COSCO</option>
            <option value="HAPAG">Hapag-Lloyd</option>
          </select>
        </div>

        {/* Disable button during submission to prevent double-submit */}
        <button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Creating...' : 'Create Shipment'}
        </button>
      </form>
    </div>
  );
}
```

### Mapping to Spring Boot

```
React Form                          Spring Boot Controller
───────────                         ──────────────────────

<form onSubmit={handleSubmit}>      @PostMapping("/api/shipments")
  │                                 public ResponseEntity<Shipment>
  │                                 createShipment(@Valid @RequestBody ShipmentDTO dto,
  │                                                BindingResult result) {
  │                                   if (result.hasErrors()) {
  │                                     return ResponseEntity.badRequest()
Zod schema validation  ◄──────────►      .body(mapErrors(result));
  │                                   }
  │                                   Shipment saved = service.create(dto);
  fetch('/api/shipments', {          return ResponseEntity.ok(saved);
    method: 'POST',                  }
    body: JSON.stringify(data)
  })
  │
  response.ok?  ◄──────────────────  ResponseEntity status code
  │
  setError(field, message)  ◄──────  BindingResult field errors
```

### Optimistic Updates

For a snappier UX, update the UI **before** the server confirms:

```tsx
const onSubmit = async (data: FormData) => {
  // 1. Optimistically add to the local list immediately
  const tempId = `temp-${Date.now()}`;
  const optimisticShipment = { ...data, id: tempId, status: 'pending' };
  setShipments(prev => [...prev, optimisticShipment]);

  try {
    // 2. POST to server
    const response = await fetch('/api/shipments', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    const saved = await response.json();

    // 3. Replace optimistic entry with real server data
    setShipments(prev =>
      prev.map(s => s.id === tempId ? saved : s)
    );
  } catch (err) {
    // 4. Rollback on failure — remove the optimistic entry
    setShipments(prev => prev.filter(s => s.id !== tempId));
    setApiError('Failed to create shipment. Please try again.');
  }
};
```

---

## Phase 7: Best Practices

### Form Development Checklist

| Practice | Why | Example |
|---|---|---|
| ✅ Use **controlled components** for important forms | Single source of truth, easy debugging | `value={state} onChange={handler}` |
| ✅ Use **form libraries** for 5+ fields | Less boilerplate, better UX | React Hook Form + Zod |
| ✅ Validate on **client AND server** | Client: UX, Server: security | Zod + `@Valid` in Spring |
| ✅ Show **clear, specific error messages** | "Required" < "Tracking ID is required" | Per-field errors near the field |
| ✅ **Disable submit** during loading | Prevent double-submission | `disabled={isSubmitting}` |
| ✅ **Show loading state** | User knows something is happening | Spinner, "Creating..." text |
| ✅ **Handle all error cases** | 400 (validation), 401 (auth), 500 (server) | Try/catch with status checks |
| ✅ Use **`noValidate`** on `<form>` | Prevent browser's ugly default validation | `<form noValidate>` |
| ❌ Don't validate **only on client** | Users can bypass JS, use curl, etc. | Always validate in Spring Boot too |
| ❌ Don't store **sensitive data** in state longer than needed | Security risk | Clear password fields after submit |

### Accessibility (a11y) Essentials

```tsx
// ✅ ACCESSIBLE form field
<div>
  <label htmlFor="trackingId">Tracking ID:</label>
  <input
    id="trackingId"                          // Matches label's htmlFor
    name="trackingId"
    aria-required="true"                      // Screen reader: "required"
    aria-invalid={!!errors.trackingId}        // Screen reader: "invalid"
    aria-describedby="trackingId-error"       // Links to error message
  />
  {errors.trackingId && (
    <span id="trackingId-error" role="alert"> {/* Screen reader announces */}
      {errors.trackingId.message}
    </span>
  )}
</div>
```

> [!warning] Don't Skip Accessibility
> Forms are the primary interaction point for many users. Always:
> - Associate `<label>` with inputs via `htmlFor`/`id`
> - Use `aria-invalid` and `aria-describedby` for error states
> - Support keyboard navigation (Tab, Enter to submit)
> - Test with a screen reader at least once

### The Full Form Architecture for Production

```
┌───────────────────────────────────────────────────────────────────┐
│                         React Frontend                            │
│                                                                   │
│  ┌──────────────┐    ┌─────────────┐    ┌──────────────────────┐ │
│  │ Zod Schema   │───→│ React Hook  │───→│ onSubmit handler     │ │
│  │ (validation) │    │ Form        │    │ - fetch() POST       │ │
│  └──────────────┘    │ (state,     │    │ - Error mapping      │ │
│                      │  errors,    │    │ - Success handling   │ │
│                      │  submit)    │    │ - Optimistic update  │ │
│                      └─────────────┘    └──────────┬───────────┘ │
│                                                     │             │
└─────────────────────────────────────────────────────┼─────────────┘
                                                      │ HTTP POST
                                                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Spring Boot Backend                         │
│                                                                  │
│  ┌────────────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ @RestController │──→│ @Valid DTO   │──→│ @Service         │  │
│  │ @PostMapping    │   │ Bean Validation │ │ Business Logic   │  │
│  └────────────────┘   └──────────────┘   └──────────────────┘  │
│                                                                  │
│  Response: 200 OK (success) | 400 Bad Request (validation errors)│
└──────────────────────────────────────────────────────────────────┘
```

---

## Further Reading

- [[Components and Props]] — Building the components that contain forms
- [[Hooks]] — `useState`, `useEffect`, `useRef` used throughout forms
- [[State and Lifecycle]] — Understanding when forms re-render
- [[Angular Forms]] — Compare with Angular's template-driven and reactive forms
