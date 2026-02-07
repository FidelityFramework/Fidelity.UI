# 00 - Architecture: The Signal-Actor Reactive Model

## The Core Insight

Signals and actors are isomorphic abstractions:

| Signal Concept | Actor Concept |
|---------------|---------------|
| Signal value changes | Actor receives message |
| Dependency tracking (who reads me?) | Subscription graph (who listens to me?) |
| Derived/memo (computed from signals) | Projection actor (transforms upstream messages) |
| Effect (side effect on change) | Actor with side effects (IO, rendering) |
| Batch (coalesce updates) | Mailbox batching (process N messages, one render) |
| Store (structured reactive state) | Stateful actor (maintains model, emits deltas) |
| `untrack` (opt out of reactivity) | Fire-and-forget (no reply expected) |

In both models, the fundamental primitive is: **a value changes, and only the things that declared interest in that value get notified.** No central diffing. No broadcast. No polling. The dependency graph *is* the message routing topology.

Fidelity.UI does not build a separate reactivity runtime. It builds UI semantics on top of Prospero/Olivier, the Fidelity actor infrastructure.

## Why Not Virtual DOM?

The virtual DOM (React, Fabulous) works by:
1. Re-running the entire view function on any state change
2. Producing a complete tree description
3. Diffing the new tree against the old tree
4. Applying only the differences to the real rendering surface

This is a broadcast-then-diff architecture. Every state change triggers a full re-evaluation, and the framework discovers what changed after the fact. It works - React proved it works at scale - but it has fundamental costs:

- **O(n) re-evaluation** where n is the tree size, even if only one leaf changed
- **Diffing overhead** that grows with tree complexity
- **Memory pressure** from maintaining two complete trees
- **Latency** from the re-evaluate → diff → apply pipeline

For a browser where DOM manipulation is the bottleneck, virtual DOM amortizes that cost. For a native rendering surface where pixel manipulation is cheap, it's pure overhead.

## The Signal-Actor Alternative

Fidelity.UI inverts the model:

1. **Components run once.** The component function executes exactly once, establishing signal subscriptions.
2. **Signals track subscribers.** When a signal is read inside a reactive context, it registers the reader as a subscriber.
3. **Changes propagate directly.** When a signal value changes, it notifies exactly its subscribers - no intermediate tree, no diffing.
4. **Layout is incremental.** Only layout regions whose input signals changed recalculate geometry.
5. **Rendering is regional.** Only pixel regions whose visual signals changed repaint.

This is the SolidJS model, transplanted from browser DOM to native compositor, and backed by Prospero/Olivier actors instead of a JavaScript reactive runtime.

### How It Works

```
User taps a button
    ↓
Button's onClick handler calls setCount(count + 1)
    ↓
count signal notifies its 2 subscribers:
    ├── Label component (displays count) → repaints 1 text region
    └── ProgressBar component (width = count * 10) → recalculates 1 layout region, repaints
    ↓
2 regions updated. Everything else untouched.
```

Compare to virtual DOM:
```
User taps a button
    ↓
setCount(count + 1) triggers full re-render
    ↓
Entire view function re-executes (Label, ProgressBar, Header, Footer, Sidebar...)
    ↓
New virtual tree produced
    ↓
Diff against old tree discovers: Label text changed, ProgressBar width changed
    ↓
Apply 2 changes to real rendering surface
    ↓
Same 2 regions updated. But the framework did O(n) work to discover that.
```

## Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│  Developer API                                          │
│  F# Computation Expressions + Phantom Type Safety       │
├─────────────────────────────────────────────────────────┤
│  Component Model                                        │
│  Components, Signals, Stores, Effects, Context          │
├─────────────────────────────────────────────────────────┤
│  Headless Primitives                                    │
│  Dialog, Select, Menu, Tabs, Toast, Popover             │
│  (Behavior + accessibility, no visual opinion)          │
├─────────────────────────────────────────────────────────┤
│  Design System                                          │
│  Tokens, Themes, Semantic Classes                       │
│  (Visual opinion, swappable)                            │
├─────────────────────────────────────────────────────────┤
│  Layout Engine                                          │
│  Stack, Flex, Grid + Constraint Solver                  │
├─────────────────────────────────────────────────────────┤
│  Signal-Actor Reactor (Prospero/Olivier)                │
│  Dependency graph, batched notifications, scheduling    │
├─────────────────────────────────────────────────────────┤
│  Rendering Backend (via Fidelity.Platform)              │
│  Wayland (Linux) | Quartz/Metal (macOS) | D3D12 (Win)  │
└─────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

**Developer API**: F# computation expressions (`Component { ... }`, `Stack() { ... }`), phantom type markers for compile-time widget validation, signal primitives (`createSignal`, `createMemo`, `createEffect`). This layer defines what developers write.

**Component Model**: Runtime representation of components, signal subscription management, store architecture (global + local), context/environment propagation. This layer manages the reactive graph.

**Headless Primitives**: UI behavior patterns (focus trapping, keyboard navigation, open/close state machines, selection models) without visual rendering. Inspired by Kobalte/Radix. A `Dialog` knows how to manage focus and escape-key; how it looks is the design system's job.

**Design System**: Hierarchical design tokens (colors, spacing, typography, elevation), theme definitions, semantic classes (`btn`, `card`, `input`). Inspired by DaisyUI/Tailwind's token-based approach. Themes are data, not code - swapping a theme changes the visual system without changing components.

**Layout Engine**: Geometry computation. Stack (vertical/horizontal), Flex (proportional), Grid (two-dimensional). Each layout node subscribes to the signals that affect its geometry (children's sizes, padding, gaps). Layout is incremental - only dirty subtrees recalculate.

**Signal-Actor Reactor**: The Prospero/Olivier substrate. Manages the dependency graph, batches signal notifications (so setting 5 signals in one handler triggers one update pass, not five), and schedules rendering work.

**Rendering Backend**: Platform-specific pixel output. Receives "repaint region X" commands and executes them via the platform compositor. Defined in Fidelity.Platform, not in Fidelity.UI.

## Comparison with Studied Frameworks

| Aspect | Fabulous | ReactiveElmish | SolidJS (web) | Fidelity.UI |
|--------|----------|----------------|---------------|-------------|
| Reactivity | Virtual DOM diff | Elmish + selective bind | Fine-grained signals | Signal-Actor |
| Components | `WidgetBuilder<'msg, 'marker>` struct | ViewModel + Store | Functions run once | CE + phantom types |
| State | MVU `'model` or `ComponentContext` | `ReactiveElmishStore` | `createSignal`/`createStore` | Actor-backed signals |
| Styling | Per-widget scalar attrs | XAML + bindings | CSS + Tailwind | Design tokens |
| Rendering | Platform widget tree | Avalonia | Browser DOM | Native compositor |
| Runtime | .NET | .NET + ReactiveUI | JavaScript | Firefly native |

Fidelity.UI takes the developer ergonomics of SolidJS (fine-grained signals, components run once), the type safety patterns of Fabulous (phantom markers, computation expressions), the store architecture of ReactiveElmish/TanStack (global + local, selective subscriptions), and the headless component philosophy of Kobalte - all backed by native compilation and actor-model reactivity.

## Navigation

- Previous: [README](../README.md)
- Next: [01_signal_system.md](./01_signal_system.md): Fine-grained reactivity primitives
