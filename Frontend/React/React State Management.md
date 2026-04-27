---
tags:
  - react
  - state-management
  - redux
  - context-api
  - frontend
  - phase3
created: 2025-07-18
---

# React State Management

> *"In a distributed warehouse, every picker must see the same inventory count — otherwise, chaos. The same is true for your UI components and shared state."*

---

## Phase 1 — When Local State Isn't Enough

In [[State and Lifecycle]], we learned that `useState` gives each component its own private state — like a warehouse clerk keeping a personal notepad. That works fine when the data stays local.

But what happens when **multiple components** across different parts of the tree need the **same data**?

> [!info] Backend Analogy
> Think of microservices. Each service has its own database for local concerns. But when the **Shipment Service**, **Tracking Service**, and **Billing Service** all need the current shipment status, you introduce a **shared database** or a **message broker** (Kafka, RabbitMQ). React state management is that shared infrastructure for your UI.

### The Problem: Distant Components Need the Same Data

```
┌─────────────────────────────────────────────┐
│                   <App>                     │
│                                             │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │  <Sidebar>   │    │  <MainContent>   │   │
│  │              │    │                  │   │
│  │  Needs user  │    │  ┌────────────┐  │   │
│  │  info + theme│    │  │ <Shipments>│  │   │
│  │              │    │  │            │  │   │
│  └──────────────┘    │  │ Needs user │  │   │
│                      │  │ info + theme│  │   │
│                      │  └────────────┘  │   │
│                      └──────────────────┘   │
└─────────────────────────────────────────────┘
```

Both `<Sidebar>` and `<Shipments>` need `user` and `theme` — but they're in completely different branches of the component tree. With just `useState`, the only way to share data is **prop drilling**.

---

## Phase 2 — Prop Drilling: The Pain

**Prop drilling** means passing data through every intermediate component between the source and the consumer — even if those middle components don't use the data at all.

> [!warning] Backend Analogy
> Imagine a customs document that must pass through 5 departments (receiving → inspection → classification → valuation → clearance) but only **receiving** and **clearance** actually read it. The middle departments just carry the paperwork forward. That's prop drilling.

### ASCII Diagram: Props Passed Through 5 Levels

```
Level 0   <App>                  ← owns `user` state
            │
            │  props: { user }
            ▼
Level 1   <Dashboard>            ← doesn't use `user`, just passes it
            │
            │  props: { user }
            ▼
Level 2   <ShipmentPanel>        ← doesn't use `user`, just passes it
            │
            │  props: { user }
            ▼
Level 3   <ShipmentList>         ← doesn't use `user`, just passes it
            │
            │  props: { user }
            ▼
Level 4   <ShipmentCard>         ← doesn't use `user`, just passes it
            │
            │  props: { user }
            ▼
Level 5   <AssignedCarrier>      ← FINALLY uses `user` to show carrier name
```

### Code Example: The Prop Drilling Nightmare

```jsx
// ❌ Every level must accept and forward `user` — even if it doesn't care

function App() {
  const [user, setUser] = useState({ name: "Leela", role: "dispatcher" });
  return <Dashboard user={user} />;
}

function Dashboard({ user }) {
  // Dashboard doesn't use `user` — just a postman
  return <ShipmentPanel user={user} />;
}

function ShipmentPanel({ user }) {
  // ShipmentPanel doesn't use `user` either
  return <ShipmentList user={user} />;
}

function ShipmentList({ user }) {
  // Still just passing it along...
  return <ShipmentCard user={user} />;
}

function ShipmentCard({ user }) {
  return <AssignedCarrier user={user} />;
}

function AssignedCarrier({ user }) {
  // FINALLY — the actual consumer
  return <p>Assigned to: {user.name}</p>;
}
```

### Why Prop Drilling Hurts

| Problem                  | Impact                                                  |
| ------------------------ | ------------------------------------------------------- |
| Verbose code             | Every middle component has props it never reads          |
| Fragile refactoring      | Rename a prop? Change it in 6 files                     |
| Tight coupling           | Components know about data they don't use               |
| Hard to test             | Middle components need mock props just to render         |
| Scales terribly          | Add a new shared value? Thread it through the entire tree|

> [!tip] Rule of Thumb
> If you're passing a prop through **3+ levels** without any component in the middle using it, you need a state management solution.

---

## Phase 3 — Context API: React's Built-in Solution

The **Context API** is React's answer to prop drilling. It lets you broadcast data to any component in the tree — no matter how deep — without passing props through every level.

> [!info] Backend Analogy
> Context is like a **service registry** (Eureka, Consul). Any microservice can look up configuration (database URL, feature flags) from the registry directly, instead of having it passed through a chain of intermediaries.

### How Context Works

```
┌──────────────────────────────────────────────┐
│             Context.Provider                 │
│          (wraps part of the tree)            │
│                                              │
│   value = { theme: "dark", user: {...} }     │
│                                              │
│   ┌────────────┐   ┌─────────────────┐       │
│   │ <Sidebar>  │   │ <MainContent>   │       │
│   │            │   │                 │       │
│   │ useContext  │   │  ┌───────────┐ │       │
│   │ ──► theme  │   │  │<Shipments>│ │       │
│   │            │   │  │           │ │       │
│   └────────────┘   │  │useContext │ │       │
│                     │  │──► user  │ │       │
│                     │  └───────────┘ │       │
│                     └─────────────────┘       │
└──────────────────────────────────────────────┘

No prop drilling! Components reach directly into the Context.
```

### Step-by-Step: Creating and Using Context

```
Step 1: createContext()        ← Define what the context holds
Step 2: <Context.Provider>     ← Wrap tree, supply the value
Step 3: useContext(Context)    ← Any descendant reads the value
```

---

### Example 1: ThemeContext (Light/Dark Toggle)

This is the "Hello World" of Context — a theme toggle that every component can access.

#### 1. Create the Context

```jsx
// src/context/ThemeContext.jsx
import { createContext, useState, useContext } from "react";

// Create with a default value (used if no Provider is found above)
const ThemeContext = createContext({
  theme: "light",
  toggleTheme: () => {},
});

// Custom provider component — keeps state logic encapsulated
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  const toggleTheme = () => {
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// Custom hook — cleaner than importing ThemeContext everywhere
export function useTheme() {
  return useContext(ThemeContext);
}
```

#### 2. Wrap Your App

```jsx
// src/App.jsx
import { ThemeProvider } from "./context/ThemeContext";
import Dashboard from "./Dashboard";

function App() {
  return (
    <ThemeProvider>
      <Dashboard />
    </ThemeProvider>
  );
}
```

#### 3. Consume Anywhere — No Prop Drilling

```jsx
// src/components/Header.jsx — 3 levels deep, doesn't matter
import { useTheme } from "../context/ThemeContext";

function Header() {
  const { theme, toggleTheme } = useTheme();

  return (
    <header className={theme === "dark" ? "bg-gray-900" : "bg-white"}>
      <h1>Freight Dashboard</h1>
      <button onClick={toggleTheme}>
        Switch to {theme === "light" ? "🌙 Dark" : "☀️ Light"} Mode
      </button>
    </header>
  );
}
```

✅ No props passed through intermediate components
✅ Any component calls `useTheme()` to get theme + toggle
✅ State logic is encapsulated in `ThemeProvider`

---

### Example 2: AuthContext (Login/Logout + User Info)

A more realistic example — authentication state shared across the app.

> [!tip] Backend Parallel
> This is like Spring Security's `SecurityContextHolder` — any controller can call `SecurityContextHolder.getContext().getAuthentication()` without it being passed as a method parameter. That's exactly what AuthContext does for React.

```jsx
// src/context/AuthContext.jsx
import { createContext, useState, useContext } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = async (email, password) => {
    // In real app: call your Spring Boot /api/auth/login endpoint
    const response = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email, password }),
    });
    const userData = await response.json();
    setUser(userData);   // { id, name, email, role }
  };

  const logout = () => {
    setUser(null);
    // Also clear tokens, call /api/auth/logout, etc.
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, isLoggedIn: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
}
```

#### Using AuthContext in Components

```jsx
// src/components/Navbar.jsx
import { useAuth } from "../context/AuthContext";

function Navbar() {
  const { user, logout, isLoggedIn } = useAuth();

  return (
    <nav>
      {isLoggedIn ? (
        <>
          <span>Welcome, {user.name} ({user.role})</span>
          <button onClick={logout}>Logout</button>
        </>
      ) : (
        <a href="/login">Sign In</a>
      )}
    </nav>
  );
}
```

```jsx
// src/pages/ShipmentPage.jsx — deeply nested, still has access
import { useAuth } from "../context/AuthContext";

function ShipmentPage() {
  const { user } = useAuth();

  if (user?.role !== "dispatcher") {
    return <p>Access denied. Dispatchers only.</p>;
  }

  return <h2>Shipment Management for {user.name}</h2>;
}
```

---

### Context API Limitations

> [!warning] When NOT to Use Context
> Context has a critical limitation: **when the Provider's value changes, every component that consumes that context re-renders** — even if the specific piece of data they use hasn't changed.

```
Provider value changes (e.g., user.lastLogin updates)
    │
    ├──► <Sidebar>     re-renders  (only uses theme)    ❌ Wasteful
    ├──► <Header>      re-renders  (only uses user.name) ❌ Wasteful
    └──► <ShipmentList> re-renders (uses user.role)      ✅ Needed
```

| ✅ Good For                          | ❌ Bad For                           |
| ------------------------------------ | ------------------------------------ |
| Theme (changes rarely)               | Shopping cart (changes often)         |
| Auth/user info (changes on login)    | Real-time tracking coordinates       |
| Locale/language                      | Form state with many fields           |
| Feature flags                        | WebSocket message streams             |

> [!tip] Rule of Thumb
> If data changes **more than once per second**, don't put it in Context. Use Redux, Zustand, or local state with selective subscriptions.

---

## Phase 4 — Redux Toolkit: The Industry Standard

When your app grows beyond a few shared values — when you have **complex state transitions**, **async API calls**, and **many interconnected slices of data** — you need Redux.

> [!info] Backend Analogy — This One Is Important
> Think of Redux through the lens of a relational database:
>
> | Redux Concept | Backend Equivalent                                 |
> | ------------- | -------------------------------------------------- |
> | **Store**     | Centralized database for your entire UI state      |
> | **Slice**     | A database table (shipments, users, notifications) |
> | **Reducer**   | A stored procedure — takes current state + action, returns new state |
> | **Action**    | An API request — `{ type: "shipment/updateStatus", payload: {...} }` |
> | **Selector**  | A SQL query — extracts specific data from the store |
> | **Dispatch**  | Calling the API — `dispatch(updateStatus(shipmentId, "delivered"))` |
> | **Thunk**     | A service method that does async work before dispatching |

### Redux Data Flow

```
  Component                     Store
  ─────────                     ─────
      │                           │
      │  dispatch(action)         │
      │ ─────────────────────►    │
      │                           │
      │                     ┌─────┴─────┐
      │                     │  Reducer   │
      │                     │            │
      │                     │ old state  │
      │                     │ + action   │
      │                     │ = new state│
      │                     └─────┬─────┘
      │                           │
      │    useSelector(selector)  │
      │ ◄─────────────────────    │
      │                           │
      │  Component re-renders     │
      │  with new data            │
      ▼                           ▼
```

### Setting Up Redux Toolkit

```bash
npm install @reduxjs/toolkit react-redux
```

---

### Full Example: Shipment State Management

Let's build a real-world shipment management slice — the kind you'd see in a logistics dashboard.

#### 1. Define the Slice

```jsx
// src/store/shipmentSlice.js
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";

// Async thunk — like a @Service method that calls an external API
export const fetchShipments = createAsyncThunk(
  "shipments/fetchAll",
  async (_, { rejectWithValue }) => {
    try {
      const response = await fetch("/api/shipments");
      if (!response.ok) throw new Error("Failed to fetch");
      return await response.json(); // returns shipment[]
    } catch (err) {
      return rejectWithValue(err.message);
    }
  }
);

// Initial state — like your database schema
const initialState = {
  items: [],           // shipment[]
  loading: false,      // is a request in-flight?
  error: null,         // last error message
  selectedId: null,    // currently selected shipment
};

const shipmentSlice = createSlice({
  name: "shipments",

  initialState,

  // Synchronous reducers — like stored procedures
  reducers: {
    addShipment: (state, action) => {
      // action.payload = { id, origin, destination, status, carrier }
      state.items.push(action.payload);
    },

    updateStatus: (state, action) => {
      // action.payload = { id, status }
      const shipment = state.items.find((s) => s.id === action.payload.id);
      if (shipment) {
        shipment.status = action.payload.status;
      }
    },

    removeShipment: (state, action) => {
      // action.payload = shipment id
      state.items = state.items.filter((s) => s.id !== action.payload);
    },

    selectShipment: (state, action) => {
      state.selectedId = action.payload;
    },

    clearError: (state) => {
      state.error = null;
    },
  },

  // Handle async thunk lifecycle — pending, fulfilled, rejected
  extraReducers: (builder) => {
    builder
      .addCase(fetchShipments.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchShipments.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchShipments.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  },
});

// Export actions (these are your "API endpoints")
export const {
  addShipment,
  updateStatus,
  removeShipment,
  selectShipment,
  clearError,
} = shipmentSlice.actions;

// Export selectors (these are your "SQL queries")
export const selectAllShipments = (state) => state.shipments.items;
export const selectShipmentById = (id) => (state) =>
  state.shipments.items.find((s) => s.id === id);
export const selectLoading = (state) => state.shipments.loading;
export const selectError = (state) => state.shipments.error;
export const selectDelivered = (state) =>
  state.shipments.items.filter((s) => s.status === "delivered");
export const selectInTransit = (state) =>
  state.shipments.items.filter((s) => s.status === "in_transit");

// Export reducer (plugs into the store)
export default shipmentSlice.reducer;
```

#### 2. Configure the Store

```jsx
// src/store/index.js
import { configureStore } from "@reduxjs/toolkit";
import shipmentReducer from "./shipmentSlice";

// configureStore = like Spring's ApplicationContext wiring everything together
export const store = configureStore({
  reducer: {
    shipments: shipmentReducer,
    // Add more slices here as your app grows:
    // users: userReducer,
    // notifications: notificationReducer,
    // warehouses: warehouseReducer,
  },
});
```

#### 3. Provide the Store to Your App

```jsx
// src/main.jsx
import { Provider } from "react-redux";
import { store } from "./store";
import App from "./App";

// Provider wraps the app — like Spring's ApplicationContext
ReactDOM.createRoot(document.getElementById("root")).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

#### 4. Use in Components with `useSelector` and `useDispatch`

```jsx
// src/components/ShipmentDashboard.jsx
import { useSelector, useDispatch } from "react-redux";
import { useEffect } from "react";
import {
  fetchShipments,
  updateStatus,
  removeShipment,
  selectAllShipments,
  selectLoading,
  selectError,
  selectInTransit,
} from "../store/shipmentSlice";

function ShipmentDashboard() {
  const dispatch = useDispatch();

  // Selectors — like SQL queries against the store
  const shipments = useSelector(selectAllShipments);
  const loading = useSelector(selectLoading);
  const error = useSelector(selectError);
  const inTransit = useSelector(selectInTransit);

  // Fetch on mount — like @PostConstruct in Spring
  useEffect(() => {
    dispatch(fetchShipments());
  }, [dispatch]);

  const handleMarkDelivered = (id) => {
    dispatch(updateStatus({ id, status: "delivered" }));
  };

  const handleRemove = (id) => {
    dispatch(removeShipment(id));
  };

  if (loading) return <p>Loading shipments...</p>;
  if (error) return <p className="error">Error: {error}</p>;

  return (
    <div>
      <h2>Shipment Dashboard ({shipments.length} total)</h2>
      <p>{inTransit.length} currently in transit</p>

      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>Origin</th>
            <th>Destination</th>
            <th>Carrier</th>
            <th>Status</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {shipments.map((s) => (
            <tr key={s.id}>
              <td>{s.id}</td>
              <td>{s.origin}</td>
              <td>{s.destination}</td>
              <td>{s.carrier}</td>
              <td>{s.status}</td>
              <td>
                <button onClick={() => handleMarkDelivered(s.id)}>
                  ✅ Delivered
                </button>
                <button onClick={() => handleRemove(s.id)}>
                  ❌ Remove
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

### RTK Query — API Data Fetching Made Easy

RTK Query is Redux Toolkit's data-fetching solution. It auto-generates hooks for your API endpoints — caching, loading states, refetching — all handled.

> [!info] Backend Analogy
> RTK Query is like **Spring Data REST** or **Spring Data JPA Repositories**. You define your entity and endpoint, and it generates the CRUD operations (queries, mutations, caching) automatically. No manual `fetch` + `useState` + `useEffect` boilerplate.

```jsx
// src/store/shipmentApi.js
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const shipmentApi = createApi({
  reducerPath: "shipmentApi",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),

  // Tags enable automatic cache invalidation — like @CacheEvict in Spring
  tagTypes: ["Shipment"],

  endpoints: (builder) => ({
    // GET /api/shipments — like a JpaRepository.findAll()
    getShipments: builder.query({
      query: () => "/shipments",
      providesTags: ["Shipment"],
    }),

    // GET /api/shipments/:id — like JpaRepository.findById()
    getShipmentById: builder.query({
      query: (id) => `/shipments/${id}`,
      providesTags: (result, error, id) => [{ type: "Shipment", id }],
    }),

    // POST /api/shipments — like JpaRepository.save()
    createShipment: builder.mutation({
      query: (newShipment) => ({
        url: "/shipments",
        method: "POST",
        body: newShipment,
      }),
      invalidatesTags: ["Shipment"], // auto-refetches the list
    }),

    // PATCH /api/shipments/:id/status
    updateShipmentStatus: builder.mutation({
      query: ({ id, status }) => ({
        url: `/shipments/${id}/status`,
        method: "PATCH",
        body: { status },
      }),
      invalidatesTags: (result, error, { id }) => [{ type: "Shipment", id }],
    }),
  }),
});

// Auto-generated hooks — use these in your components
export const {
  useGetShipmentsQuery,
  useGetShipmentByIdQuery,
  useCreateShipmentMutation,
  useUpdateShipmentStatusMutation,
} = shipmentApi;
```

#### Using RTK Query Hooks in a Component

```jsx
// src/components/ShipmentList.jsx
import {
  useGetShipmentsQuery,
  useUpdateShipmentStatusMutation,
} from "../store/shipmentApi";

function ShipmentList() {
  // This single hook handles: fetch, loading, error, caching, refetch
  const { data: shipments, isLoading, error } = useGetShipmentsQuery();

  const [updateStatus] = useUpdateShipmentStatusMutation();

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error loading shipments</p>;

  return (
    <ul>
      {shipments.map((s) => (
        <li key={s.id}>
          {s.origin} → {s.destination} [{s.status}]
          <button onClick={() => updateStatus({ id: s.id, status: "delivered" })}>
            Mark Delivered
          </button>
        </li>
      ))}
    </ul>
  );
}
```

> [!tip] Compare the Approaches
> ```
> Manual fetch + useState + useEffect    RTK Query
> ─────────────────────────────────      ──────────────────────
> 15-25 lines of boilerplate             1 line: useGetShipmentsQuery()
> Manual loading/error states            Auto-managed
> No caching                             Built-in cache + dedup
> Manual refetch logic                   Auto refetch on mutation
> ```

---

## Phase 5 — Zustand: The Lightweight Alternative

**Zustand** (German for "state") is a minimal state management library. If Redux is a full warehouse management system, Zustand is a well-organized clipboard — lightweight, effective, no ceremony.

```bash
npm install zustand
```

### Same Shipment Store — in Zustand

```jsx
// src/store/useShipmentStore.js
import { create } from "zustand";

const useShipmentStore = create((set, get) => ({
  // State
  shipments: [],
  loading: false,
  error: null,

  // Actions — defined right alongside state (no actions/reducers split)
  fetchShipments: async () => {
    set({ loading: true, error: null });
    try {
      const response = await fetch("/api/shipments");
      const data = await response.json();
      set({ shipments: data, loading: false });
    } catch (err) {
      set({ error: err.message, loading: false });
    }
  },

  addShipment: (shipment) => {
    set((state) => ({
      shipments: [...state.shipments, shipment],
    }));
  },

  updateStatus: (id, status) => {
    set((state) => ({
      shipments: state.shipments.map((s) =>
        s.id === id ? { ...s, status } : s
      ),
    }));
  },

  removeShipment: (id) => {
    set((state) => ({
      shipments: state.shipments.filter((s) => s.id !== id),
    }));
  },

  // Computed values — like a getter in your @Entity
  getInTransit: () => {
    return get().shipments.filter((s) => s.status === "in_transit");
  },

  getDelivered: () => {
    return get().shipments.filter((s) => s.status === "delivered");
  },
}));

export default useShipmentStore;
```

### Using Zustand in a Component

```jsx
// src/components/ShipmentDashboard.jsx
import { useEffect } from "react";
import useShipmentStore from "../store/useShipmentStore";

function ShipmentDashboard() {
  // Subscribe to specific slices — only re-renders when THESE change
  const shipments = useShipmentStore((state) => state.shipments);
  const loading = useShipmentStore((state) => state.loading);
  const fetchShipments = useShipmentStore((state) => state.fetchShipments);
  const updateStatus = useShipmentStore((state) => state.updateStatus);

  useEffect(() => {
    fetchShipments();
  }, [fetchShipments]);

  if (loading) return <p>Loading...</p>;

  return (
    <ul>
      {shipments.map((s) => (
        <li key={s.id}>
          {s.origin} → {s.destination} [{s.status}]
          <button onClick={() => updateStatus(s.id, "delivered")}>
            ✅ Delivered
          </button>
        </li>
      ))}
    </ul>
  );
}
```

### Zustand vs Redux Toolkit — Side by Side

```
                    Redux Toolkit              Zustand
                    ─────────────              ───────
Setup               configureStore +           create() — one function
                    Provider wrapper

Boilerplate         Slices, actions,           State + actions in one object
                    selectors, thunks

Re-render control   useSelector picks          Selector function in hook
                    specific state             (same concept, less wiring)

Async               createAsyncThunk           Just use async/await in actions
                    with extraReducers

DevTools            Built-in                   Plugin available

Bundle size         ~11 KB                     ~1.5 KB

Learning curve      Medium                     Low
```

### When to Choose Zustand Over Redux

✅ **Use Zustand when:**
- Your app is small-to-medium sized
- You want minimal boilerplate
- You don't need middleware, time-travel debugging, or strict action logging
- You prefer a "just works" API

❌ **Stick with Redux when:**
- Large team needs strict patterns and conventions
- You need middleware (logging, analytics, error reporting)
- You want RTK Query for API data management
- You need time-travel debugging with Redux DevTools

---

## Phase 6 — Decision Matrix

> [!tip] Quick Reference
> Use this table to pick the right tool for the job — like choosing between LTL, FTL, and parcel shipping based on your cargo size.

| Scenario                                  | Solution           | Why                                            |
| ----------------------------------------- | ------------------ | ---------------------------------------------- |
| 2-3 nearby components share state         | **Lift state up**  | Simplest — move state to common parent         |
| Theme, locale, auth (changes rarely)      | **Context API**    | Built-in, no dependencies, perfect for globals |
| Complex app with many state slices        | **Redux Toolkit**  | Scalable, strict patterns, great DevTools      |
| Medium app, want simplicity               | **Zustand**        | Tiny, fast, minimal boilerplate                |
| Heavy server-state (CRUD APIs)            | **RTK Query**      | Auto caching, refetch, mutation invalidation   |
| Real-time data (WebSocket, live tracking) | **Zustand + subs** | Lightweight subscriptions, easy to integrate   |

### Logistics Analogy for Each Solution

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Lift State Up    = Two desks in the same office sharing    │
│                     a clipboard. Just pass it over.         │
│                                                             │
│  Context API      = A warehouse bulletin board. Everyone    │
│                     can read it, updates are infrequent.    │
│                                                             │
│  Redux Toolkit    = A centralized TMS (Transport Mgmt      │
│                     System). Structured, audited, handles   │
│                     complex workflows with strict rules.    │
│                                                             │
│  Zustand          = A shared spreadsheet. Quick to set up,  │
│                     everyone can read/write, but no audit   │
│                     trail or strict access control.         │
│                                                             │
│  RTK Query        = An EDI integration. Define the message  │
│                     format once, and data flows in/out      │
│                     automatically with caching.             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 7 — Comparison With Angular

If you've looked at [[Angular State Management]], the concepts map directly:

| React                        | Angular Equivalent                |
| ---------------------------- | --------------------------------- |
| Context API                  | Angular Services with `@Injectable` |
| Redux Toolkit                | NgRx (Store, Actions, Reducers, Effects) |
| RTK Query                    | NgRx Entity + NgRx Effects        |
| Zustand                      | Akita / Elf                       |
| `useSelector`                | `store.select()`                  |
| `useDispatch`                | `store.dispatch()`                |
| `createSlice`                | `createReducer` + `createAction`  |
| `createAsyncThunk`           | NgRx Effects (`createEffect`)     |

> [!info] Key Difference
> Angular services are **singleton by default** — injected via DI, they naturally act as shared state. React has **no built-in DI**, so you need Context, Redux, or Zustand to achieve the same thing.

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│              REACT STATE MANAGEMENT CHEAT SHEET              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  LOCAL STATE         useState / useReducer                   │
│  SHARED (simple)     Lift state up to parent                 │
│  SHARED (global)     Context API                             │
│  COMPLEX STATE       Redux Toolkit (createSlice)             │
│  API DATA            RTK Query (createApi)                   │
│  LIGHTWEIGHT         Zustand (create)                        │
│                                                              │
│  ─── WHEN TO UPGRADE ───                                     │
│                                                              │
│  Props through 3+ levels?         → Context or Redux         │
│  Multiple slices of related data? → Redux Toolkit            │
│  Just need a shared counter?      → Zustand or Context       │
│  Heavy CRUD with caching?         → RTK Query                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Cross-References

- [[Hooks]] — `useState`, `useReducer`, `useContext` deep dive
- [[State and Lifecycle]] — local component state fundamentals
- [[Angular State Management]] — NgRx, services, and how Angular handles shared state
