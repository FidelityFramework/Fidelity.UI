# 01 - Signal System: Fine-Grained Reactivity Primitives

## Design Goal

Provide the same reactive primitives that SolidJS developers know - `createSignal`, `createMemo`, `createEffect`, `createStore` - but backed by the Prospero/Olivier actor model instead of a JavaScript runtime, and compiled to native code by Firefly.

A developer who has used Partas.Solid in the WRENStack should recognize the API immediately.

## Primitive Catalog

### Signal: Reactive Value

The fundamental primitive. A value that tracks its readers and notifies them on change.

```fsharp
// Create a signal with an initial value
let count, setCount = createSignal 0

// Read (inside a reactive context, this registers a subscription)
let current = count()

// Write (notifies all subscribers)
setCount 5

// Update from previous value
setCount (fun prev -> prev + 1)
```

**Implementation**: A signal is an Olivier actor that holds a value and maintains a subscriber set. Reading the signal inside a tracked context (component body, effect, memo) adds the tracking context to the subscriber set. Setting the signal dispatches notifications to all subscribers.

### Memo: Derived Computation

A cached computation that re-evaluates only when its dependencies change.

```fsharp
let count, setCount = createSignal 0
let doubled = createMemo (fun () -> count() * 2)

// doubled() returns 0
setCount 5
// doubled() returns 10 (recomputed because count changed)
// If count hasn't changed since last read, returns cached value
```

**Implementation**: A memo is a signal-actor that subscribes to upstream signals. When notified of a change, it re-evaluates its function, and only if the result differs from the cached value does it notify its own subscribers. This prevents cascading updates through chains of derived values.

### Effect: Side Effect on Change

Runs arbitrary code when tracked dependencies change.

```fsharp
let name, setName = createSignal "world"

createEffect (fun () ->
    // This re-runs whenever name() changes
    Platform.Console.writeLine $"Hello, {name()}!"
)
```

**Implementation**: An effect is a subscriber-actor that re-runs its function when notified. Effects are scheduled after the current batch completes, preventing interleaving of state updates and side effects.

### Store: Structured Reactive State

Deep-reactive state for complex data structures. Path-based updates allow fine-grained notification.

```fsharp
type TodoItem = { Id: int; Title: string; Done: bool }
type AppState = { Todos: TodoItem list; Filter: string }

let store, setStore = createStore { Todos = []; Filter = "all" }

// Read (fine-grained - only subscribes to accessed paths)
let filter = store.Filter

// Update (only notifies subscribers of the changed path)
setStore.Path.Map(_.Filter).Update("active")

// Deep update
setStore.Path.Map(_.Todos).Find(fun t -> t.Id = 1).Map(_.Done).Update(true)
```

**Implementation**: A store wraps a value in a proxy that tracks access paths. When a path is read in a reactive context, only that specific path is subscribed. When a path is updated, only subscribers to that path (or ancestor paths) are notified.

### Batch: Coalesced Updates

Multiple signal updates in one handler trigger a single notification pass.

```fsharp
batch (fun () ->
    setFirstName "John"
    setLastName "Doe"
    setAge 30
)
// Subscribers notified once, not three times
```

**Implementation**: Batching defers subscriber notifications until the batch function completes, then runs a single notification pass. This maps directly to Prospero/Olivier mailbox coalescing.

### Context: Hierarchical State

Provide values to a subtree without prop drilling.

```fsharp
let ThemeContext = createContext defaultTheme

// Provider wraps a subtree
ThemeContext.Provider(darkTheme) {
    MyComponent()  // can access theme
    OtherComponent()  // can access theme
}

// Consumer reads from nearest provider
let theme = useContext ThemeContext
```

**Implementation**: Context is stored in the environment hierarchy (conceptually similar to Fabulous's `EnvironmentContext`). Context values are themselves signals - when a context value changes, all consumers in the subtree are notified.

### Resource: Async Data with Lifecycle

Load async data with built-in loading/error states.

```fsharp
let userId, setUserId = createSignal 1

let user = createResource
    (fun () -> userId() |> Some)
    (fun id _ -> fetchUser id)

// user.state: Unresolved | Pending | Ready | Refreshing | Errored
// user.current: the loaded value (undefined while loading)
// user.latest: keeps previous value while refreshing
```

**Implementation**: A resource is a compound actor that manages an async operation lifecycle. It subscribes to its source signal, triggers the fetcher on change, and exposes loading/error/data states as signals.

## Reactive Scope Rules

### Tracking Contexts

Signal reads are only tracked inside reactive contexts:
- **Component bodies** (during initial execution)
- **createMemo** computations
- **createEffect** bodies
- **createResource** source functions

Outside these contexts, signal reads return the current value without establishing subscriptions.

### Untracking

```fsharp
let value = untrack (fun () -> someSignal())
// Reads someSignal's value without subscribing
```

### Cleanup

Effects can register cleanup functions that run before re-execution:

```fsharp
createEffect (fun () ->
    let subscription = eventSource.subscribe (fun e -> handle e)
    onCleanup (fun () -> subscription.Dispose())
)
```

## Relationship to Prospero/Olivier

The signal system is not a separate runtime bolted onto the actor model. It *is* the actor model with UI-specific semantics:

| Signal Primitive | Olivier Implementation |
|-----------------|----------------------|
| `createSignal` | Stateful actor with subscriber set |
| `createMemo` | Projection actor (subscribes upstream, notifies downstream) |
| `createEffect` | Terminal actor (subscribes, runs side effects) |
| `createStore` | Hierarchical stateful actor with path-based routing |
| `batch` | Mailbox coalescing / transaction boundary |
| `createContext` | Environment actor scoped to subtree |
| `createResource` | Compound actor managing async lifecycle |

This means Fidelity.UI applications can freely mix UI signals with non-UI actors. A WebSocket connection actor can dispatch messages that update UI signals. A background computation actor can emit progress signals. The reactive graph is unified.

## Navigation

- Previous: [00_architecture.md](./00_architecture.md)
- Next: [02_component_model.md](./02_component_model.md): Composition DSL and phantom types
