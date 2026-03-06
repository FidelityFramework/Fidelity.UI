# 05 - WRENStack to Fidelity.UI Migration Path

## The Two-Track Strategy

WRENStack and Fidelity.UI are not competitors. They are two tracks of the same strategy:

- **WRENStack**: Ship applications today using proven web technology with native performance
- **Fidelity.UI**: Build toward a fully native UI model with no web runtime dependency

The migration path is designed so that WRENStack applications can progressively adopt Fidelity.UI as the native rendering backends mature, without rewriting application logic.

## What Transfers Directly

### Signal Primitives

The signal API is designed to be isomorphic between Partas.Solid (WRENStack) and Fidelity.UI:

| Partas.Solid (Web) | Fidelity.UI (Native) |
|--------------------|--------------------|
| `createSignal(value)` | `createSignal value` |
| `createMemo(() => ...)` | `createMemo (fun () -> ...)` |
| `createEffect(() => ...)` | `createEffect (fun () -> ...)` |
| `createStore(obj)` | `createStore value` |
| `createContext(default)` | `createContext defaultValue` |
| `createResource(source, fetcher)` | `createResource source fetcher` |
| `batch(() => ...)` | `batch (fun () -> ...)` |

The semantics are identical. The implementation changes (JavaScript reactive runtime vs Prospero/Olivier actors), but the developer-facing API is the same.

### Store Patterns

TanStack Store patterns used in WRENStack map to Fidelity.UI stores:

| TanStack Store (Web) | Fidelity.UI Store (Native) |
|---------------------|-----------------------|
| `new Store(initial)` | `createStore initial` |
| `store.state` | `store.Value` |
| `store.setState(updater)` | `setStore updater` |
| `useStore(store, selector)` | `useStore store selector` |
| `new Derived(options)` | `createMemo` with store dependency |

### Component Structure

Components follow the same "run once, subscribe to signals" model:

```fsharp
// WRENStack (Partas.Solid, compiles to SolidJS)
[<SolidComponent>]
let Counter () =
    let count, setCount = createSignal 0
    div(class' = "card") {
        span() { $"Count: {count()}" }
        button(onClick = fun _ -> setCount.Invoke(fun c -> c + 1)) { "+" }
    }

// Fidelity.UI (compiles to native)
[<Component>]
let Counter () =
    let count, setCount = createSignal 0
    Card() {
        Label($"Count: {count()}")
        Button("+", fun () -> setCount (fun c -> c + 1))
    }
```

The structure is the same: create signals, compose widgets, read signals in widget properties. What changes is:
- HTML elements (`div`, `span`, `button`) become Fidelity.UI widgets (`Card`, `Label`, `Button`)
- CSS classes (`class' = "card"`) become design system styles (`.style(Card.Default)`)
- Partas.Solid compilation target (JavaScript) becomes Firefly compilation target (native)

### Application State

The application state layer (stores, context, effects) transfers completely. Business logic that manages state through signals doesn't change when the rendering backend changes.

## What Changes

### Visual Layer

HTML + CSS becomes Fidelity.UI widgets + design tokens. This is the primary migration surface:

| WRENStack | Fidelity.UI |
|-----------|-------------|
| `div(class' = "flex flex-col gap-4")` | `Stack(direction = Vertical, gap = 4)` |
| `button(class' = "btn btn-primary")` | `Button("text").style(Btn.Primary)` |
| `input(class' = "input input-bordered")` | `TextInput().style(Input.Bordered)` |
| `div(class' = "card bg-base-200")` | `Card().background(theme.surface)` |
| Tailwind utility classes | Design tokens + semantic styles |
| DaisyUI component classes | Headless primitives + design system |

### Rendering Target

The WebView disappears. Instead of:
```
F# → Fable → JavaScript → SolidJS → Browser DOM → WebView → Compositor
```

The path becomes:
```
F# → Firefly → Native Binary → Signal-Actor Reactor → Render Regions → Compositor
```

### IPC Boundary

In WRENStack, frontend and backend communicate via BAREWire over WebSocket. In Fidelity.UI, everything is the same process. No IPC, no serialization, no WebSocket. Signals in the UI directly reference application state.

## Migration Strategy

### Phase 1: Shared State Layer

Extract application state (stores, business logic, data management) into a shared module that doesn't depend on UI rendering:

```
src/
├── Shared/
│   ├── State.fs      # Stores, business logic (works with both targets)
│   └── Protocol.fs   # Types (already shared in WRENStack)
├── Frontend/
│   └── App.fs        # WRENStack UI (Partas.Solid)
└── Native/
    └── App.fs        # Fidelity.UI (when ready)
```

### Phase 2: Component-by-Component

Migrate individual components from Partas.Solid to Fidelity.UI. Since both use the same signal model, the state management code transfers directly - only the widget tree changes.

### Phase 3: Full Native

When all components are migrated, remove the WebView dependency. The application is now pure native, compiled entirely by Firefly.

## Atelier: The Migration Case Study

Atelier (the Fidelity IDE) is built on WRENStack today. As Fidelity.UI matures, Atelier will be the first application to migrate. This creates a tight feedback loop:

1. Atelier uses WRENStack (current)
2. Fidelity.UI development begins, informed by Atelier's needs
3. Atelier migrates components progressively
4. Fidelity.UI improves based on migration experience
5. Atelier becomes fully native

This self-hosting loop ensures Fidelity.UI is battle-tested on a complex, real-world application before being offered to external developers.

## Navigation

- Previous: [04_design_system.md](./04_design_system.md)
- Next: [06_prior_art.md](./06_prior_art.md): Survey of signals-to-native frameworks
