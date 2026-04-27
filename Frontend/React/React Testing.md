---
tags:
  - react
  - testing
  - jest
  - rtl
  - frontend
  - phase4
created: 2025-07-18
---

# React Testing

> *"Quality is not an act, it is a habit." — Aristotle*

Think of testing your React components the same way you'd test a warehouse receiving dock. You don't care **how** the conveyor belt works internally — you care that packages arrive at the right bay, with the right labels, in the right condition. React testing follows the same principle: test what the **user sees and does**, not the internal wiring.

---

## Phase 1 — Testing Philosophy

### Test Behaviour, Not Implementation

If you've written JUnit tests for a Spring Boot REST controller, you already understand this instinct. You test the **HTTP response** — status code, body, headers — not whether some private method inside the service layer was called with the right argument.

React testing works the same way:

```
┌─────────────────────────────────────────────────────┐
│              BACKEND (Spring Boot)                   │
│                                                      │
│   ✅  Test: POST /api/shipments returns 201          │
│   ❌  Test: shipmentService.validate() was called    │
│                                                      │
├─────────────────────────────────────────────────────┤
│              FRONTEND (React)                        │
│                                                      │
│   ✅  Test: "Shipment Created" message appears       │
│   ❌  Test: useState was called with correct value   │
│                                                      │
└─────────────────────────────────────────────────────┘
```

> [!tip] The Golden Rule
> If a user can't observe it, don't test it. Users don't inspect React state — they read text, click buttons, and fill forms. Test **those** things.

### Why This Matters

Imagine you refactor a component from `useState` to `useReducer`. If your tests check state internals, they all break — even though the UI didn't change at all. That's like a warehouse QA test failing because you swapped a forklift brand, even though the packages still arrive on time.

Behaviour-focused tests survive refactors. Implementation-focused tests don't.

---

## Phase 2 — The Testing Pyramid

Just like in Java, frontend testing follows a pyramid. More unit tests at the base, fewer E2E tests at the top.

```
                        ╱╲
                       ╱  ╲
                      ╱ E2E╲          ← Slow, expensive, few
                     ╱──────╲            (Cypress / Playwright)
                    ╱        ╲
                   ╱Integration╲      ← Moderate speed, moderate count
                  ╱────────────╲        (RTL + MSW)
                 ╱              ╲
                ╱    Unit Tests  ╲    ← Fast, cheap, many
               ╱──────────────────╲     (Jest + RTL)
              ╱                    ╲
             ╱──────────────────────╲
```

### Mapping to Java Equivalents

| Layer         | Java / Spring Boot            | React / Frontend               |
| ------------- | ----------------------------- | ------------------------------ |
| **Unit**      | JUnit + Mockito               | Jest + React Testing Library   |
| **Integration** | `@SpringBootTest` + MockMvc | RTL + MSW (Mock Service Worker) |
| **E2E**       | Selenium / RestAssured        | Cypress / Playwright           |

> [!info] Logistics Analogy
> Think of the pyramid like freight inspection levels:
> - **Unit** = Checking each package label individually (fast, done for every item)
> - **Integration** = Checking that packages flow correctly through the sorting facility
> - **E2E** = Running a full shipment from origin warehouse to destination dock

---

## Phase 3 — Jest: The Test Runner

Jest is to JavaScript what **JUnit is to Java**. It's the test runner, assertion library, and mocking framework all in one.

### Setup

If you used Create React App or Vite with the test plugin, Jest is already configured:

```bash
# Create React App — Jest is built in
npx create-react-app my-logistics-app --template typescript
npm test   # runs Jest in watch mode

# Vite — install vitest (Jest-compatible API)
npm install -D vitest @testing-library/react @testing-library/jest-dom
```

### Core Concepts: describe, it/test, expect

```typescript
// freightUtils.test.ts
// Think of 'describe' as a test class, 'it' as a test method

describe('calculateFreightCost', () => {

  it('should calculate cost for standard ground shipping', () => {
    // Arrange
    const weight = 500;   // kg
    const distance = 200; // km
    const ratePerKgKm = 0.05;

    // Act
    const cost = calculateFreightCost(weight, distance, ratePerKgKm);

    // Assert
    expect(cost).toBe(5000);
  });

  it('should return 0 for zero weight', () => {
    expect(calculateFreightCost(0, 200, 0.05)).toBe(0);
  });

  it('should throw for negative weight', () => {
    expect(() => calculateFreightCost(-10, 200, 0.05)).toThrow('Weight must be positive');
  });
});
```

> [!tip] If you know JUnit, you know Jest
> `describe` ≈ test class, `it`/`test` ≈ `@Test` method, `expect` ≈ assertions.

### Common Matchers

```typescript
// Exact equality — like assertEquals
expect(status).toBe('delivered');

// Deep equality for objects/arrays — like assertThat with Hamcrest
expect(shipment).toEqual({
  id: 'SHIP-001',
  status: 'in-transit',
  origin: 'Melbourne',
  destination: 'Sydney'
});

// Check if array/string contains a value
expect(warehouses).toContain('Brisbane');
expect(trackingId).toContain('SHIP');

// Check truthiness
expect(isDelivered).toBeTruthy();
expect(errorMessage).toBeFalsy();

// Numeric comparisons
expect(freightCost).toBeGreaterThan(0);
expect(deliveryDays).toBeLessThanOrEqual(5);

// Check if function throws
expect(() => validateWeight(-1)).toThrow();
expect(() => validateWeight(-1)).toThrow('Invalid weight');
```

### Mocking with Jest

Mocking in Jest is like Mockito in Java — you replace real dependencies with controlled fakes.

```typescript
// jest.fn() — create a mock function (like Mockito.mock())
const mockOnShipmentSelect = jest.fn();

// Call it
mockOnShipmentSelect('SHIP-001');

// Assert it was called
expect(mockOnShipmentSelect).toHaveBeenCalledWith('SHIP-001');
expect(mockOnShipmentSelect).toHaveBeenCalledTimes(1);
```

```typescript
// jest.mock() — mock an entire module (like @MockBean)
jest.mock('../services/shipmentApi', () => ({
  fetchShipments: jest.fn().mockResolvedValue([
    { id: 'SHIP-001', status: 'delivered' },
    { id: 'SHIP-002', status: 'in-transit' },
  ]),
}));
```

```typescript
// jest.spyOn() — spy on a real method (like Mockito.spy())
const consoleSpy = jest.spyOn(console, 'error').mockImplementation();

// ... run code that logs errors ...

expect(consoleSpy).toHaveBeenCalledWith('Shipment not found');
consoleSpy.mockRestore();
```

### Full Example: Testing a Utility Function

```typescript
// src/utils/freightCalculator.ts
export function calculateFreightCost(
  weightKg: number,
  distanceKm: number,
  ratePerKgKm: number
): number {
  if (weightKg < 0) throw new Error('Weight must be positive');
  if (distanceKm < 0) throw new Error('Distance must be positive');
  return weightKg * distanceKm * ratePerKgKm;
}

export function getShippingTier(cost: number): string {
  if (cost <= 100) return 'economy';
  if (cost <= 500) return 'standard';
  return 'express';
}
```

```typescript
// src/utils/freightCalculator.test.ts
import { calculateFreightCost, getShippingTier } from './freightCalculator';

describe('freightCalculator', () => {

  describe('calculateFreightCost', () => {
    it('calculates cost correctly', () => {
      expect(calculateFreightCost(100, 50, 0.10)).toBe(500);
    });

    it('returns 0 when weight is 0', () => {
      expect(calculateFreightCost(0, 100, 0.05)).toBe(0);
    });

    it('throws for negative weight', () => {
      expect(() => calculateFreightCost(-5, 100, 0.05))
        .toThrow('Weight must be positive');
    });

    it('throws for negative distance', () => {
      expect(() => calculateFreightCost(10, -100, 0.05))
        .toThrow('Distance must be positive');
    });
  });

  describe('getShippingTier', () => {
    it('returns economy for cost <= 100', () => {
      expect(getShippingTier(50)).toBe('economy');
      expect(getShippingTier(100)).toBe('economy');
    });

    it('returns standard for cost 101-500', () => {
      expect(getShippingTier(250)).toBe('standard');
      expect(getShippingTier(500)).toBe('standard');
    });

    it('returns express for cost > 500', () => {
      expect(getShippingTier(1000)).toBe('express');
    });
  });
});
```

---

## Phase 4 — React Testing Library (RTL)

RTL is the **core library** for testing React components. While Jest is the runner and asserter, RTL renders your components and lets you query the DOM the way a real user would.

> [!info] Philosophy
> *"The more your tests resemble the way your software is used, the more confidence they can give you."* — Kent C. Dodds (RTL creator)

Think of RTL as a warehouse inspector who can only use their **eyes and hands** — they read labels, press buttons, fill clipboards. They can't open the conveyor belt's control panel and inspect the circuit board.

### The `screen` Object

After rendering a component, `screen` gives you access to the rendered DOM — like a window into the browser.

```typescript
import { render, screen } from '@testing-library/react';
import { ShipmentCard } from './ShipmentCard';

test('renders shipment info', () => {
  render(<ShipmentCard trackingId="SHIP-001" status="in-transit" />);

  // screen is the rendered output — query it like a user would
  const heading = screen.getByText('SHIP-001');
  expect(heading).toBeInTheDocument();
});
```

### Query Priority Order

RTL provides multiple query methods. Use them in this priority order — prefer accessible queries that mirror how real users (including screen readers) find elements:

```
┌──────────────────────────────────────────────────────────┐
│  QUERY PRIORITY (use the highest available)              │
│                                                          │
│  1. getByRole        ← Best. Accessible to all users     │
│  2. getByLabelText   ← Great for form fields             │
│  3. getByPlaceholder ← OK for inputs without labels      │
│  4. getByText        ← Good for buttons, headings, text  │
│  5. getByDisplayValue← For filled form elements          │
│  6. getByAltText     ← For images                        │
│  7. getByTitle       ← Rarely used                       │
│  8. getByTestId      ← Last resort escape hatch          │
└──────────────────────────────────────────────────────────┘
```

> [!warning] Avoid `getByTestId` Unless Necessary
> Adding `data-testid` attributes everywhere is like putting barcodes on the **inside** of packages — sure, you can scan them, but a customer will never find them. Prefer queries that a user could actually use.

### Query Examples

```typescript
// ── getByRole ──
// Finds elements by their ARIA role (button, heading, textbox, etc.)
screen.getByRole('button', { name: /submit shipment/i });
screen.getByRole('heading', { level: 2 });

// ── getByLabelText ──
// Finds form inputs by their associated <label>
screen.getByLabelText('Tracking ID');
screen.getByLabelText(/destination/i);

// ── getByPlaceholderText ──
screen.getByPlaceholderText('Enter weight in kg');

// ── getByText ──
// Finds elements by visible text content
screen.getByText('Shipment Created Successfully');
screen.getByText(/in-transit/i);  // regex for case-insensitive

// ── getByTestId ── (last resort)
screen.getByTestId('shipment-status-badge');
```

### Query Variants: get vs query vs find

| Variant     | No Match  | 1 Match | 1+ Match | Async? |
| ----------- | --------- | ------- | -------- | ------ |
| `getBy`     | ❌ Throws | ✅      | ❌ Throws | No     |
| `queryBy`   | `null`    | ✅      | ❌ Throws | No     |
| `findBy`    | ❌ Throws | ✅      | ❌ Throws | ✅ Yes  |
| `getAllBy`   | ❌ Throws | ✅ []   | ✅ []     | No     |
| `queryAllBy` | `[]`     | ✅ []   | ✅ []     | No     |
| `findAllBy`  | ❌ Throws | ✅ []   | ✅ []     | ✅ Yes  |

> [!tip] When to Use Which
> - **`getBy`** — element should be there right now (default choice)
> - **`queryBy`** — asserting something is NOT in the DOM
> - **`findBy`** — element appears after async operation (API call, setTimeout)

### User Events

RTL provides `userEvent` to simulate real user interactions — clicks, typing, selecting:

```typescript
import userEvent from '@testing-library/user-event';

test('clicking track button shows tracking info', async () => {
  const user = userEvent.setup();
  render(<ShipmentTracker />);

  // Click a button
  await user.click(screen.getByRole('button', { name: /track shipment/i }));

  // Type into an input
  await user.type(screen.getByLabelText('Tracking ID'), 'SHIP-001');

  // Select a dropdown option
  await user.selectOptions(
    screen.getByLabelText('Shipping Method'),
    'express'
  );

  // Clear and retype
  await user.clear(screen.getByLabelText('Weight'));
  await user.type(screen.getByLabelText('Weight'), '250');
});
```

### Async Utilities: waitFor and findBy

When components fetch data or update asynchronously, use `waitFor` or `findBy`:

```typescript
import { render, screen, waitFor } from '@testing-library/react';

test('loads shipment list from API', async () => {
  render(<ShipmentList />);

  // findBy waits for the element to appear (default timeout: 1000ms)
  const shipment = await screen.findByText('SHIP-001');
  expect(shipment).toBeInTheDocument();

  // waitFor retries until the assertion passes
  await waitFor(() => {
    expect(screen.getAllByRole('row')).toHaveLength(5);
  });
});
```

> [!info] Logistics Analogy
> `findBy` is like waiting at the receiving dock — you know the truck is coming, you just don't know exactly when. The test keeps checking until the shipment (element) arrives, or times out.

---

## Phase 5 — Component Testing Examples

### Testing a Component with Props

```tsx
// src/components/ShipmentCard.tsx
interface ShipmentCardProps {
  trackingId: string;
  status: 'pending' | 'in-transit' | 'delivered' | 'cancelled';
  origin: string;
  destination: string;
}

export function ShipmentCard({ trackingId, status, origin, destination }: ShipmentCardProps) {
  return (
    <div className="shipment-card" data-testid="shipment-card">
      <h3>{trackingId}</h3>
      <span className={`status-badge status-${status}`}>{status}</span>
      <p>{origin} → {destination}</p>
    </div>
  );
}
```

```tsx
// src/components/ShipmentCard.test.tsx
import { render, screen } from '@testing-library/react';
import { ShipmentCard } from './ShipmentCard';

describe('ShipmentCard', () => {

  it('renders shipment card with tracking ID', () => {
    render(<ShipmentCard trackingId="SHIP-001" status="in-transit"
      origin="Melbourne" destination="Sydney" />);

    expect(screen.getByText('SHIP-001')).toBeInTheDocument();
    expect(screen.getByText('in-transit')).toBeInTheDocument();
  });

  it('displays route information', () => {
    render(<ShipmentCard trackingId="SHIP-002" status="pending"
      origin="Brisbane" destination="Perth" />);

    expect(screen.getByText('Brisbane → Perth')).toBeInTheDocument();
  });

  it('applies correct status class', () => {
    render(<ShipmentCard trackingId="SHIP-003" status="delivered"
      origin="Adelaide" destination="Darwin" />);

    const badge = screen.getByText('delivered');
    expect(badge).toHaveClass('status-delivered');
  });
});
```

### Testing User Interactions

```tsx
// src/components/ShipmentActions.tsx
interface ShipmentActionsProps {
  trackingId: string;
  onCancel: (id: string) => void;
  onTrack: (id: string) => void;
}

export function ShipmentActions({ trackingId, onCancel, onTrack }: ShipmentActionsProps) {
  return (
    <div>
      <button onClick={() => onTrack(trackingId)}>Track</button>
      <button onClick={() => onCancel(trackingId)}>Cancel Shipment</button>
    </div>
  );
}
```

```tsx
// src/components/ShipmentActions.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ShipmentActions } from './ShipmentActions';

describe('ShipmentActions', () => {

  it('calls onTrack with tracking ID when Track is clicked', async () => {
    const user = userEvent.setup();
    const mockOnTrack = jest.fn();
    const mockOnCancel = jest.fn();

    render(
      <ShipmentActions
        trackingId="SHIP-001"
        onTrack={mockOnTrack}
        onCancel={mockOnCancel}
      />
    );

    await user.click(screen.getByRole('button', { name: /track/i }));

    expect(mockOnTrack).toHaveBeenCalledWith('SHIP-001');
    expect(mockOnTrack).toHaveBeenCalledTimes(1);
    expect(mockOnCancel).not.toHaveBeenCalled();
  });

  it('calls onCancel when Cancel Shipment is clicked', async () => {
    const user = userEvent.setup();
    const mockOnCancel = jest.fn();

    render(
      <ShipmentActions
        trackingId="SHIP-005"
        onTrack={jest.fn()}
        onCancel={mockOnCancel}
      />
    );

    await user.click(screen.getByRole('button', { name: /cancel shipment/i }));

    expect(mockOnCancel).toHaveBeenCalledWith('SHIP-005');
  });
});
```

### Testing Conditional Rendering

```tsx
// src/components/DeliveryStatus.tsx
interface DeliveryStatusProps {
  status: 'pending' | 'in-transit' | 'delivered' | 'cancelled';
}

export function DeliveryStatus({ status }: DeliveryStatusProps) {
  return (
    <div>
      {status === 'delivered' && <p>✅ Package has been delivered</p>}
      {status === 'cancelled' && <p>❌ Shipment was cancelled</p>}
      {status === 'in-transit' && <p>🚚 Package is on its way</p>}
      {status === 'pending' && <p>📦 Awaiting pickup</p>}
    </div>
  );
}
```

```tsx
// src/components/DeliveryStatus.test.tsx
import { render, screen } from '@testing-library/react';
import { DeliveryStatus } from './DeliveryStatus';

describe('DeliveryStatus', () => {

  it('shows delivered message', () => {
    render(<DeliveryStatus status="delivered" />);
    expect(screen.getByText(/package has been delivered/i)).toBeInTheDocument();
    expect(screen.queryByText(/cancelled/i)).not.toBeInTheDocument();
  });

  it('shows in-transit message', () => {
    render(<DeliveryStatus status="in-transit" />);
    expect(screen.getByText(/package is on its way/i)).toBeInTheDocument();
  });

  it('shows pending message', () => {
    render(<DeliveryStatus status="pending" />);
    expect(screen.getByText(/awaiting pickup/i)).toBeInTheDocument();
  });

  it('shows cancelled message', () => {
    render(<DeliveryStatus status="cancelled" />);
    expect(screen.getByText(/shipment was cancelled/i)).toBeInTheDocument();
  });
});
```

---

## Phase 6 — Testing Hooks

Custom hooks can be tested in isolation using `renderHook` from `@testing-library/react`.

> [!info] Think of It Like Testing a Service Layer
> In Spring Boot, you test `@Service` classes independently by mocking their dependencies. `renderHook` does the same for custom React hooks — runs them outside a component so you can inspect their return values.

```tsx
// src/hooks/useShipments.ts
import { useState, useEffect } from 'react';

interface Shipment {
  id: string;
  status: string;
  destination: string;
}

export function useShipments() {
  const [shipments, setShipments] = useState<Shipment[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch('/api/shipments')
      .then(res => res.json())
      .then(data => {
        setShipments(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, []);

  return { shipments, loading, error };
}
```

```tsx
// src/hooks/useShipments.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useShipments } from './useShipments';

// Mock the global fetch
beforeEach(() => {
  global.fetch = jest.fn();
});

afterEach(() => {
  jest.restoreAllMocks();
});

describe('useShipments', () => {

  it('returns shipments after successful fetch', async () => {
    const mockData = [
      { id: 'SHIP-001', status: 'delivered', destination: 'Sydney' },
      { id: 'SHIP-002', status: 'in-transit', destination: 'Perth' },
    ];

    (global.fetch as jest.Mock).mockResolvedValue({
      json: () => Promise.resolve(mockData),
    });

    const { result } = renderHook(() => useShipments());

    // Initially loading
    expect(result.current.loading).toBe(true);
    expect(result.current.shipments).toEqual([]);

    // After fetch completes
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.shipments).toHaveLength(2);
    expect(result.current.shipments[0].id).toBe('SHIP-001');
    expect(result.current.error).toBeNull();
  });

  it('returns error on fetch failure', async () => {
    (global.fetch as jest.Mock).mockRejectedValue(
      new Error('Network error')
    );

    const { result } = renderHook(() => useShipments());

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBe('Network error');
    expect(result.current.shipments).toEqual([]);
  });
});
```

---

## Phase 7 — Testing Forms

Forms are the bread and butter of logistics apps — booking shipments, entering addresses, filling customs declarations. Here's how to test them thoroughly.

```tsx
// src/components/ShipmentForm.tsx
import { useState } from 'react';

interface ShipmentFormProps {
  onSubmit: (data: { trackingId: string; weight: number; destination: string }) => void;
}

export function ShipmentForm({ onSubmit }: ShipmentFormProps) {
  const [trackingId, setTrackingId] = useState('');
  const [weight, setWeight] = useState('');
  const [destination, setDestination] = useState('');
  const [errors, setErrors] = useState<Record<string, string>>({});

  const validate = () => {
    const newErrors: Record<string, string> = {};
    if (!trackingId.trim()) newErrors.trackingId = 'Tracking ID is required';
    if (!weight || Number(weight) <= 0) newErrors.weight = 'Weight must be greater than 0';
    if (!destination.trim()) newErrors.destination = 'Destination is required';
    return newErrors;
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors = validate();
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    setErrors({});
    onSubmit({ trackingId, weight: Number(weight), destination });
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="trackingId">Tracking ID</label>
        <input id="trackingId" value={trackingId}
          onChange={e => setTrackingId(e.target.value)} />
        {errors.trackingId && <span role="alert">{errors.trackingId}</span>}
      </div>
      <div>
        <label htmlFor="weight">Weight (kg)</label>
        <input id="weight" type="number" value={weight}
          onChange={e => setWeight(e.target.value)} />
        {errors.weight && <span role="alert">{errors.weight}</span>}
      </div>
      <div>
        <label htmlFor="destination">Destination</label>
        <input id="destination" value={destination}
          onChange={e => setDestination(e.target.value)} />
        {errors.destination && <span role="alert">{errors.destination}</span>}
      </div>
      <button type="submit">Create Shipment</button>
    </form>
  );
}
```

```tsx
// src/components/ShipmentForm.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { ShipmentForm } from './ShipmentForm';

describe('ShipmentForm', () => {

  it('submits form with valid data', async () => {
    const user = userEvent.setup();
    const mockSubmit = jest.fn();
    render(<ShipmentForm onSubmit={mockSubmit} />);

    // Fill in all fields
    await user.type(screen.getByLabelText('Tracking ID'), 'SHIP-100');
    await user.type(screen.getByLabelText('Weight (kg)'), '250');
    await user.type(screen.getByLabelText('Destination'), 'Sydney');

    // Submit the form
    await user.click(screen.getByRole('button', { name: /create shipment/i }));

    // Verify onSubmit was called with correct data
    expect(mockSubmit).toHaveBeenCalledWith({
      trackingId: 'SHIP-100',
      weight: 250,
      destination: 'Sydney',
    });
  });

  it('shows validation errors for empty fields', async () => {
    const user = userEvent.setup();
    render(<ShipmentForm onSubmit={jest.fn()} />);

    // Submit without filling anything
    await user.click(screen.getByRole('button', { name: /create shipment/i }));

    // All three error messages should appear
    expect(screen.getByText('Tracking ID is required')).toBeInTheDocument();
    expect(screen.getByText('Weight must be greater than 0')).toBeInTheDocument();
    expect(screen.getByText('Destination is required')).toBeInTheDocument();
  });

  it('shows weight error for invalid weight', async () => {
    const user = userEvent.setup();
    render(<ShipmentForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText('Tracking ID'), 'SHIP-200');
    await user.type(screen.getByLabelText('Weight (kg)'), '-5');
    await user.type(screen.getByLabelText('Destination'), 'Perth');

    await user.click(screen.getByRole('button', { name: /create shipment/i }));

    expect(screen.getByText('Weight must be greater than 0')).toBeInTheDocument();
    expect(screen.queryByText('Tracking ID is required')).not.toBeInTheDocument();
  });

  it('does not call onSubmit when validation fails', async () => {
    const user = userEvent.setup();
    const mockSubmit = jest.fn();
    render(<ShipmentForm onSubmit={mockSubmit} />);

    await user.click(screen.getByRole('button', { name: /create shipment/i }));

    expect(mockSubmit).not.toHaveBeenCalled();
  });

  it('clears errors after successful submission', async () => {
    const user = userEvent.setup();
    render(<ShipmentForm onSubmit={jest.fn()} />);

    // First submit with errors
    await user.click(screen.getByRole('button', { name: /create shipment/i }));
    expect(screen.getAllByRole('alert')).toHaveLength(3);

    // Fill in valid data and resubmit
    await user.type(screen.getByLabelText('Tracking ID'), 'SHIP-300');
    await user.type(screen.getByLabelText('Weight (kg)'), '100');
    await user.type(screen.getByLabelText('Destination'), 'Darwin');
    await user.click(screen.getByRole('button', { name: /create shipment/i }));

    // Errors should be gone
    expect(screen.queryAllByRole('alert')).toHaveLength(0);
  });
});
```

---

## Phase 8 — Mocking API Calls with MSW

**Mock Service Worker (MSW)** intercepts HTTP requests at the network level — your component thinks it's talking to a real server. This is far superior to `jest.mock()` for API testing.

```
┌──────────────────────────────────────────────────────────────┐
│  jest.mock() approach:                                       │
│                                                              │
│    Component ──→ [MOCK MODULE] ──✗ fetch never executes      │
│    ⚠ Tests the mock, not the real fetch logic                │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  MSW approach:                                               │
│                                                              │
│    Component ──→ fetch() ──→ [MSW INTERCEPT] ──→ mock data   │
│    ✅ Full fetch pipeline runs, only the network is mocked   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

> [!tip] Logistics Analogy
> `jest.mock()` is like replacing the entire truck with a cardboard cutout. MSW is like rerouting the truck to a local warehouse with pre-staged inventory — the truck still drives, loads, and unloads.

### MSW Setup

```bash
npm install -D msw
```

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  // Mock GET /api/shipments
  http.get('/api/shipments', () => {
    return HttpResponse.json([
      { id: 'SHIP-001', status: 'delivered', destination: 'Sydney' },
      { id: 'SHIP-002', status: 'in-transit', destination: 'Melbourne' },
      { id: 'SHIP-003', status: 'pending', destination: 'Brisbane' },
    ]);
  }),

  // Mock POST /api/shipments
  http.post('/api/shipments', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json(
      { id: 'SHIP-NEW', ...body, status: 'pending' },
      { status: 201 }
    );
  }),

  // Mock error scenario
  http.get('/api/shipments/:id', ({ params }) => {
    if (params.id === 'INVALID') {
      return HttpResponse.json(
        { error: 'Shipment not found' },
        { status: 404 }
      );
    }
    return HttpResponse.json({
      id: params.id, status: 'in-transit', destination: 'Perth'
    });
  }),
];
```

```typescript
// src/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```typescript
// src/setupTests.ts (or jest.setup.ts)
import { server } from './mocks/server';

beforeAll(() => server.listen());           // Start intercepting
afterEach(() => server.resetHandlers());    // Reset any per-test overrides
afterAll(() => server.close());             // Clean up
```

### Using MSW in Tests

```tsx
// src/components/ShipmentList.test.tsx
import { render, screen } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { server } from '../mocks/server';
import { ShipmentList } from './ShipmentList';

describe('ShipmentList', () => {

  it('renders shipments from API', async () => {
    render(<ShipmentList />);

    // Wait for async data to load
    expect(await screen.findByText('SHIP-001')).toBeInTheDocument();
    expect(screen.getByText('SHIP-002')).toBeInTheDocument();
    expect(screen.getByText('SHIP-003')).toBeInTheDocument();
  });

  it('shows error message on API failure', async () => {
    // Override handler for this specific test
    server.use(
      http.get('/api/shipments', () => {
        return HttpResponse.json(
          { error: 'Internal Server Error' },
          { status: 500 }
        );
      })
    );

    render(<ShipmentList />);

    expect(await screen.findByText(/error loading shipments/i))
      .toBeInTheDocument();
  });

  it('shows empty state when no shipments exist', async () => {
    server.use(
      http.get('/api/shipments', () => {
        return HttpResponse.json([]);
      })
    );

    render(<ShipmentList />);

    expect(await screen.findByText(/no shipments found/i))
      .toBeInTheDocument();
  });
});
```

### Why MSW > jest.mock for API Tests

| Aspect              | `jest.mock()`              | **MSW**                        |
| ------------------- | -------------------------- | ------------------------------ |
| What's mocked       | The import/module          | The network request            |
| Fetch logic tested  | ❌ No                      | ✅ Yes                          |
| Error handling      | Must mock manually         | ✅ Real HTTP status codes       |
| Reusable handlers   | ❌ Per-file mocks           | ✅ Shared handler definitions   |
| Works in browser    | ❌ No                      | ✅ Yes (for dev/storybook)      |
| Closest Java equiv. | Mockito `when().thenReturn()` | WireMock                    |

---

## Phase 9 — Snapshot Testing

Snapshot testing captures a component's rendered output and compares it to a stored reference. If the output changes, the test fails.

```tsx
import { render } from '@testing-library/react';
import { ShipmentCard } from './ShipmentCard';

test('ShipmentCard matches snapshot', () => {
  const { container } = render(
    <ShipmentCard trackingId="SHIP-001" status="delivered"
      origin="Melbourne" destination="Sydney" />
  );
  expect(container).toMatchSnapshot();
});
```

### When to Use ✅ and When to Avoid ❌

```
┌────────────────────────────────────────────────────────────────┐
│  ✅  USE SNAPSHOTS FOR:                                        │
│      • Stable, rarely-changing UI (footer, logo, static page)  │
│      • Catching unintended changes in markup structure          │
│      • Quick smoke tests for simple presentational components   │
│                                                                │
│  ❌  AVOID SNAPSHOTS FOR:                                      │
│      • Components with dynamic data (dates, IDs, random)       │
│      • Frequently updated components (snapshot churn)           │
│      • Complex components (massive snapshots are unreadable)    │
│      • As a substitute for proper behavioural tests             │
└────────────────────────────────────────────────────────────────┘
```

> [!warning] Snapshot Pitfall
> Large snapshots become rubber-stamp reviews — developers just run `jest --updateSnapshot` without actually reviewing the diff. Use snapshots sparingly and keep them small.

---

## Phase 10 — Testing Best Practices

| #  | Practice                                                  | Status |
| -- | --------------------------------------------------------- | ------ |
| 1  | Test what users see and do, not internal state            | ✅      |
| 2  | Use `getByRole` and `getByLabelText` as primary queries   | ✅      |
| 3  | Test component internals (`useState`, `useEffect` calls)  | ❌      |
| 4  | Use `userEvent` over `fireEvent` for realistic simulation | ✅      |
| 5  | Write one assertion per test when possible                | ✅      |
| 6  | Hardcode `data-testid` on every element "just in case"    | ❌      |
| 7  | Use MSW for API mocking over `jest.mock(fetch)`           | ✅      |
| 8  | Test error states and loading states, not just happy path | ✅      |
| 9  | Use snapshot tests as primary testing strategy            | ❌      |
| 10 | Clean up after each test (`afterEach`, `cleanup`)         | ✅      |
| 11 | Test accessibility (roles, labels, ARIA attributes)       | ✅      |
| 12 | Couple tests tightly to CSS class names or DOM structure  | ❌      |

### Quick Reference: Jest & RTL Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│  JEST ASSERTIONS                                            │
│                                                             │
│  expect(x).toBe(y)              Strict equality             │
│  expect(x).toEqual(y)           Deep equality               │
│  expect(x).toBeTruthy()         Truthy check                │
│  expect(fn).toThrow()           Exception thrown             │
│  expect(fn).toHaveBeenCalled()  Mock was invoked             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  RTL QUERIES                                                │
│                                                             │
│  screen.getByRole('button')     By ARIA role                │
│  screen.getByText(/hello/i)     By visible text             │
│  screen.getByLabelText('Name')  By form label               │
│  screen.findByText('Loaded')    Async — waits for element   │
│  screen.queryByText('Gone')     Returns null if missing     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  JEST-DOM MATCHERS                                          │
│                                                             │
│  .toBeInTheDocument()           Element exists in DOM        │
│  .toHaveTextContent('text')     Contains text                │
│  .toBeVisible()                 Element is visible           │
│  .toBeDisabled()                Element is disabled          │
│  .toHaveClass('active')         Has CSS class                │
│  .toHaveValue('100')            Input has value              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparison: React Testing vs Angular Testing

If you also work with Angular, here's how testing concepts map:

| Concept            | React (Jest + RTL)                     | Angular (Jasmine + TestBed)                 |
| ------------------ | -------------------------------------- | ------------------------------------------- |
| Test runner        | Jest / Vitest                          | Karma + Jasmine                             |
| Component render   | `render(<Component />)`                | `TestBed.createComponent(Component)`        |
| Query DOM          | `screen.getByText()`                   | `fixture.debugElement.query(By.css())`      |
| Detect changes     | Automatic                              | `fixture.detectChanges()` (manual)          |
| Mock services      | MSW or `jest.mock()`                   | `TestBed.overrideProvider()`                |
| User interaction   | `userEvent.click()`                    | `element.triggerEventHandler('click')`      |
| Async testing      | `findBy*`, `waitFor`                   | `fakeAsync`, `tick`, `flush`                |
| Snapshot testing   | `toMatchSnapshot()`                    | Limited support                             |

> [!tip] Key Difference
> React Testing Library encourages testing from the **user's perspective** by default. Angular's `TestBed` gives you more access to component internals, which can tempt you into testing implementation details. Both can produce great tests — just be mindful of what you're asserting.

For more on Angular testing, see [[Angular Testing]].

---

## Related Notes

- [[Components and Props]] — Building the components you'll test
- [[Hooks]] — Custom hooks and how to test them with `renderHook`
- [[Event Handling and Forms]] — The form patterns tested in Phase 7
- [[Angular Testing]] — Angular's TestBed approach compared to RTL
