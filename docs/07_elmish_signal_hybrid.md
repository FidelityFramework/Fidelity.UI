# 07 - The Elmish-Signal Hybrid: Coherent State, Surgical Reactivity

## Origin: WRENStack as Proof of Concept

The hybrid model emerged from production experience with WRENStack, where Partas.Solid (SolidJS bindings for Fable) and TanStack solid-store were combined to solve a fundamental tension:

- **Elmish** gives coherent, traceable state transitions — but its virtual-DOM-style view rebuild is O(n)
- **Signals** give surgical, fine-grained reactivity — but distributed signal state becomes untraceable spaghetti at scale

WRENStack resolved this by using TanStack Store as a **predicate buffer** between the Elmish loop and the signal graph. The Elmish update function remains the single source of truth for state transitions. The store observes the model, structurally compares what changed, and selectively fires only the signals representing real deltas. The UI subscribes to signals, not to the model directly.

## The Architecture

```
  ┌─────────────────────────────────────┐
  │  Message                            │
  │  User action, async result, timer   │
  └──────────────┬──────────────────────┘
                 ↓
  ┌─────────────────────────────────────┐
  │  Elmish Update                      │
  │  Model → Message → Model            │
  │  Pure function. Testable. Traceable. │
  └──────────────┬──────────────────────┘
                 ↓
  ┌─────────────────────────────────────┐
  │  Store (Predicate Buffer)           │
  │  Structural diff of model fields    │
  │  Only changed paths fire signals    │
  └──────────────┬──────────────────────┘
                 ↓
  ┌─────────────────────────────────────┐
  │  Signal Graph                       │
  │  Fine-grained subscriptions         │
  │  Memos, effects, derived values     │
  └──────────────┬──────────────────────┘
                 ↓
  ┌─────────────────────────────────────┐
  │  Render Regions                     │
  │  Only affected regions repaint      │
  └─────────────────────────────────────┘
```

### Why the Store Layer Matters

Without the store, you have two unpalatable choices:

1. **Pure Elmish**: The update function produces a new model. The view function rebuilds the entire widget tree. A diffing layer discovers what changed. This works (Fabulous, Elm, React) but the O(n) view rebuild and diffing pass are pure overhead for native rendering.

2. **Pure Signals**: Fine-grained and surgical, but state transitions are scattered across signal setters. Debugging "how did we get here" means tracing a graph of interdependent signal mutations. At scale, this becomes the same problem as imperative state management.

The store bridges them:

- **Upstream** (Elmish → Store): The store receives the new model and performs structural comparison. It knows `model.User.Name` changed but `model.User.Avatar` didn't. This is a single, bounded comparison — not a tree diff.
- **Downstream** (Store → Signals): Only the `userName` signal fires. The `userAvatar` signal stays silent. Every subscriber of `userName` gets notified; nothing else does.

The store is not a cache. It's a **change predicate** — it answers "did this specific path change?" and converts Elmish's coarse-grained model replacement into the fine-grained signal deltas that the rendering layer needs.

## One Programming Model, Three Targets

The critical design decision: the developer-facing API is the **same** across all rendering backends. The CE surface, the state management pattern, the component structure — these are invariant. Only the rendering substrate changes.

```
                    ┌──────────────┐
                    │  Application │
                    │  Elmish Loop │
                    │  Store       │
                    │  Components  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │  WRENStack │ │  Partas    │ │ Fidelity.UI│
     │  WebView   │ │  Solid     │ │  Native    │
     │  Electron  │ │  CF Pages  │ │  Wayland   │
     │  Tauri     │ │  Workers   │ │  Quartz    │
     └────────────┘ └────────────┘ │  D3D12     │
                                   └────────────┘
```

| Layer | WRENStack | Partas.Solid | Fidelity.UI |
|-------|-----------|--------------|-------------|
| **State** | Elmish + TanStack Store | Elmish + TanStack Store | Elmish + Actor-backed Store |
| **Signals** | SolidJS reactive runtime | SolidJS reactive runtime | Prospero/Olivier actors |
| **Components** | Partas CEs (div, span) | Partas CEs (div, span) | Fidelity CEs (Stack, Label) |
| **Rendering** | Browser DOM via WebView | Browser DOM via CF Workers | Native compositor |
| **Compilation** | Fable → JavaScript | Fable → JavaScript | Composer → MLIR → native |

The Elmish loop and store predicates are **identical code** across all three. The component CEs differ in their widget vocabulary (HTML elements vs native widgets) but share the same composition patterns. A developer who writes WRENStack applications today already knows the Fidelity.UI programming model — they just need to swap widget names.

## Fabulous CE Ergonomics, Signal Propagation

Fabulous demonstrated that F# computation expressions are the right developer surface for declarative UI. The `WidgetBuilder<'msg, 'marker>` pattern — using phantom type markers for compile-time widget validation — is genuinely excellent API design. Fidelity.UI adopts this approach:

```fsharp
// Fabulous-style CE composition
[<Component>]
let TodoApp () =
    let model, dispatch = useElmish init update

    Stack(direction = Vertical, gap = 8) {
        Label($"Items: {model.Count}")     // signal subscription, not view rebuild
        for item in model.Items ->
            TodoItem(item, dispatch)
        TextInput(onSubmit = fun text ->
            dispatch (AddItem text))
    }
```

The difference from Fabulous: this component body runs **once**. `model.Count` and `model.Items` are signal reads, not value accesses. When `Count` changes, only the `Label` region updates. When `Items` changes, only the list region updates. The `TextInput` is untouched in both cases. No view function re-execution. No tree diffing.

The CE surface feels like Fabulous. The propagation model is SolidJS. The substrate is Prospero actors. The compilation target is native MLIR. And the application code is the same code that runs on WRENStack today.

## Integration with Signal-Actor Substrate

In the Fidelity.UI architecture (see [00_architecture.md](./00_architecture.md)), signals are actors. The Elmish-Signal hybrid maps naturally onto this:

| Hybrid Component | Actor Model Realization |
|-----------------|------------------------|
| Elmish dispatch loop | Stateful actor (receives Messages, produces Models) |
| Store (predicate buffer) | Projection actor (receives Model, emits path deltas) |
| Signal | Actor with subscriber list (receives delta, notifies subscribers) |
| Memo | Derived actor (recomputes on dependency notification) |
| Effect | Actor with side effects (IO, logging, external system) |
| Render region | Leaf actor (receives signal, repaints pixel buffer) |

The entire pipeline — from user input to pixel output — is actors sending messages. No separate reactive runtime. No event bus. No observer pattern bolted on. The actor model IS the reactive model. Prospero/Olivier provides the scheduling, batching, and lifecycle management.

## Design Principle: Don't Make the Good the Enemy of the Perfect

Pure signals (SolidJS style) are simpler to explain and implement. Pure Elmish (MVU style) is more coherent and testable. The hybrid is more complex — it has three layers (Elmish, Store, Signals) where pure approaches have one or two.

The complexity is justified because:

1. **It's proven.** WRENStack applications in production use this exact pattern via Partas.Solid + TanStack Store. The hybrid isn't theoretical.
2. **It migrates.** The same code runs on WebView today and native tomorrow. A pure-signals approach would require rewriting WRENStack application state management. A pure-Elmish approach would require virtual DOM diffing in native rendering.
3. **It scales.** Small apps can use signals directly (skip the Elmish loop). Large apps get Elmish's coherence without its O(n) rendering cost. The store layer is opt-in, not mandatory.

The store layer is the key enabler. Without it, you choose between coherence and efficiency. With it, you get both.

## Navigation

- Previous: [06_prior_art.md](./06_prior_art.md)
