# 06 - Prior Art: Signals-to-Native in the Wild

## The Convergent Insight

The idea of applying SolidJS-style fine-grained reactivity to native UI rendering is not unique to Fidelity.UI. Multiple teams have independently arrived at the same conclusion: the signal model is a better substrate for UI than virtual DOM diffing, and it translates naturally beyond the browser.

What differs across these projects is the language, the rendering approach, the platform scope, and the degree to which signals drive rendering (vs. just state management). Fidelity.UI's unique position is the combination of F# type system, actor-model substrate, MLIR-native compilation, and a migration path from a production web stack (WRENStack).

## The Spectrum of Adoption

Projects fall along a spectrum of how deeply SolidJS's model is adopted:

| Depth | Approach | Examples |
|-------|----------|----------|
| **Full** | Signals drive both state AND rendering. No virtual DOM, no diffing. | SolidJS, Leptos, Floem, KiteUI |
| **Partial** | Signals for state management, but rendering still uses VirtualDOM diffing | Dioxus |
| **Inspired** | View-tree diffing (SwiftUI/Elm model) with some signal-like patterns | Xilem |
| **Implicit** | Signal-equivalent primitives under different names (@State, @Observable) | SwiftUI |

Fidelity.UI targets the **Full** end: signals are actors, actors drive rendering, no diffing anywhere.

## Project Survey

### SolidJS - The Origin

- **Repository**: [github.com/solidjs/solid](https://github.com/solidjs/solid)
- **Language**: JavaScript/TypeScript
- **Platforms**: Web (DOM)

The framework that proved fine-grained reactivity could outperform virtual DOM diffing at scale. SolidJS's `createSignal`, `createMemo`, `createEffect` are the vocabulary that all projects below draw from. Components are setup functions that run once; signal reads establish subscriptions; only the specific DOM nodes that depend on a signal update when it changes.

**What Fidelity.UI takes**: The complete reactive primitive API, the "components run once" model, the control flow components (Show, For, Switch/Match), and the mental model that the framework compiles away.

### Leptos - SolidJS in Rust

- **Repository**: [github.com/leptos-rs/leptos](https://github.com/leptos-rs/leptos)
- **Language**: Rust
- **Platforms**: Web (WebAssembly + SSR)

The most faithful SolidJS port in any language. Leptos proved that SolidJS's signal model translates cleanly to a systems language with ownership semantics. Signals are `Copy + 'static` in Rust, providing ergonomics competitive with the JavaScript original. A signal change updates a single DOM text node or attribute - no diffing, no virtual DOM.

**What Fidelity.UI takes**: Proof that fine-grained reactivity works in a compiled, statically-typed language without a garbage collector. The reactive_graph crate demonstrates clean separation of the reactive core from the rendering layer.

### Floem - Signals Meet Native Rendering

- **Repository**: [github.com/lapce/floem](https://github.com/lapce/floem)
- **Language**: Rust
- **Platforms**: Desktop (Windows, macOS, Linux)

Built by the Lapce editor team for their own needs. Floem is the closest existing precedent to what Fidelity.UI aims to be: SolidJS-style signals driving native desktop rendering without a browser. The view tree is constructed once (no rebuild on state change), and reactive signals surgically update only affected render output. Rendering is via wgpu using Vello or vger for vector graphics, with tiny-skia as CPU fallback. Layout via Taffy (Flexbox + Grid).

**What Fidelity.UI takes**: Validation that the signal-to-native model works in production (Lapce editor). The architectural pattern of signals → dirty regions → GPU render. The lesson that Flexbox/Grid layout algorithms can serve native UIs well.

**Where Fidelity.UI differs**: Floem is Rust-only with no migration path from a web stack. Its reactive system is self-contained rather than building on an actor substrate. No headless primitive / design system separation.

### Dioxus - Signals + VirtualDOM Hybrid

- **Repository**: [github.com/DioxusLabs/dioxus](https://github.com/DioxusLabs/dioxus)
- **Language**: Rust
- **Platforms**: Web, Desktop (WebView), Mobile, SSR

Dioxus adopted `use_signal` for state management (replacing `use_state`/`use_ref`) but retained a React-inspired virtual DOM for rendering. Signals drive reactivity - knowing *what* changed - but the framework still diffs a virtual tree to determine *how* to update the output. This is closer to React + signals than to SolidJS.

Notably, Dioxus desktop renders via WebView (using wry/Tauri's WebView wrapper), making it architecturally very similar to WRENStack: signals for state, HTML/CSS for rendering, a system WebView as the bridge to native. Dioxus also ships a component library modeled after shadcn/ui and Radix Primitives, providing headless-style accessible components (Dialog, Select, Menu, Tabs, etc.) within a WebView-rendered context. An experimental WGPU-based native renderer (Blitz) is in progress.

**What Fidelity.UI takes**: The cross-platform ambition (web, desktop, mobile from one codebase). Dioxus's experience with WebView-based desktop rendering validates the WRENStack architecture as a viable shipping strategy. The shadcn/Radix-modeled component library is a concrete reference for Fidelity.UI's planned headless primitives - proof that the Kobalte/Radix pattern translates beyond the browser.

**Where Fidelity.UI differs**: Fidelity.UI rejects virtual DOM diffing entirely. Signals drive rendering directly, not just state management. The actor model makes this possible - each render region is an actor that subscribes to its dependencies. Critically, WRENStack has an explicit migration path to Fidelity.UI (shared signal API, component-by-component migration), while Dioxus's move from WebView to Blitz is a clean break with no progressive migration story.

### Xilem - View-Tree Diffing with GPU Rendering

- **Repository**: [github.com/linebender/xilem](https://github.com/linebender/xilem)
- **Language**: Rust
- **Platforms**: Desktop (Windows, macOS, Linux)

Raph Levien's research framework. Xilem uses a view-tree diffing approach (closer to SwiftUI/Elm than SolidJS) with Vello for GPU-accelerated vector rendering. The user returns a declarative view tree each cycle; the framework diffs against the previous tree and applies minimal mutations to retained Masonry widgets.

**What Fidelity.UI takes**: Vello's GPU rendering architecture is relevant research for Fidelity.UI's Stage 2+ GPU rendering path. Masonry's approach to accessible widget trees (via AccessKit) is informative for native accessibility. Parley's text rendering research is valuable.

**Where Fidelity.UI differs**: Xilem's diffing model (rebuild + diff) is exactly what Fidelity.UI avoids. The "components run once" model with signal subscriptions eliminates the need for tree diffing entirely.

### KiteUI - SolidJS in Kotlin Multiplatform

- **Repository**: [github.com/lightningkite/kiteui](https://github.com/lightningkite/kiteui)
- **Language**: Kotlin
- **Platforms**: Web (semantic HTML), Android, iOS, JVM

The most explicit adoption outside JavaScript. KiteUI describes itself as "based on Solid JS" and uses fine-grained reactivity with automatic dependency tracking. Uniquely, it renders to each platform's native view system (not a custom rendering engine), producing semantic HTML on web and native views on mobile. The reactivity system also tracks loading states automatically.

**What Fidelity.UI takes**: Proof that the SolidJS model works in a JVM/multiplatform language. KiteUI's approach to automatic loading state tracking is worth studying for Resource primitive design. The semantic rendering strategy (native views per platform, not canvas) aligns with Fidelity.UI's approach of platform-native rendering backends.

**Where Fidelity.UI differs**: KiteUI targets mobile-first (Android/iOS) while Fidelity.UI targets desktop-first (Wayland/Quartz/Win32). KiteUI relies on platform widget trees; Fidelity.UI uses region-based rendering with custom drawing.

### SwiftUI - The Implicit Precedent

- **Platforms**: Apple (iOS, macOS, watchOS, tvOS, visionOS)
- **Language**: Swift

SwiftUI doesn't call them "signals," but `@State`, `@Binding`, `@Observable`, and `@Environment` are functionally equivalent. A `@State` variable is a signal. A `@Binding` is a shared signal reference. `@Observable` (Observation framework, iOS 17+) provides fine-grained property tracking similar to SolidJS stores. `@Environment` is context propagation.

SwiftUI's body is re-evaluated on state change (closer to Xilem's diffing model than SolidJS's run-once model), but the framework performs structural identity matching to minimize actual view updates - achieving similar practical outcomes through a different mechanism.

**What Fidelity.UI takes**: The modifier pattern (`.font()`, `.foregroundColor()`, `.padding()`) as a proven UX for declarative styling. The `@Environment` / context propagation model for themes and configuration. Proof that a signal-equivalent model can power a production native UI framework at massive scale.

**Where Fidelity.UI differs**: SwiftUI is Apple-only, closed-source, and uses view-body re-execution rather than true fine-grained subscriptions. Fidelity.UI's actor model enables genuine run-once components with surgical signal-driven updates.

## Comparison Matrix

| | SolidJS | Leptos | Floem | Dioxus | Xilem | KiteUI | SwiftUI | **Fidelity.UI** |
|---|---|---|---|---|---|---|---|---|
| **Language** | JS/TS | Rust | Rust | Rust | Rust | Kotlin | Swift | **F#** |
| **Signal model** | Original | Faithful port | Inspired by | State only | None (diff) | Faithful port | Implicit | **Actor-backed** |
| **Components run once** | Yes | Yes | Yes | No (re-render) | No (re-render) | Yes | No (re-render) | **Yes** |
| **Rendering** | DOM | WASM/DOM | wgpu/Vello | WebView/WASM | Vello/Masonry | Platform native | Apple native | **Region-based** |
| **Native rendering** | No | No | Yes | Partial | Yes | Yes | Yes | **Yes** |
| **Cross-platform** | Web only | Web only | Desktop | Web+Desktop+Mobile | Desktop | Web+Mobile | Apple only | **Desktop (3 OS)** |
| **Headless primitives** | No (Kobalte) | No (3rd party) | No | Yes (shadcn/Radix) | No | No | No | **Planned** |
| **Design system** | No (Tailwind) | No (Tailwind) | No | No | No | Partial | Yes | **Planned** |
| **Type-safe modifiers** | No | Partial | No | No | No | No | Yes | **Phantom types** |
| **Compiler integration** | Babel plugin | Rust proc macros | None | Rust proc macros | None | None | Swift compiler | **Firefly MLIR** |
| **Actor substrate** | No | No | No | No | No | No | No | **Prospero/Olivier** |
| **Migration from web** | N/A | N/A | No | Partial | No | Partial | No | **WRENStack** |

## Lessons Extracted

### What the Field Validates

1. **Fine-grained reactivity outperforms virtual DOM diffing** - proven by SolidJS benchmarks, adopted by Leptos, Floem, KiteUI
2. **The signal model translates to statically-typed, compiled languages** - Leptos (Rust), KiteUI (Kotlin), both successful
3. **Signals can drive native rendering without a browser** - Floem in production (Lapce editor)
4. **"Components run once" works** - adopted by every framework using full signal model
5. **Cross-platform is achievable** - Dioxus and KiteUI demonstrate multi-target from single codebase

### What No One Has Done Yet

1. **Actor-model substrate for reactivity** - Every framework builds a custom reactive runtime. None uses an actor system as the reactive foundation. Fidelity.UI's signal-actor isomorphism is novel.
2. **Compiler-level integration** - Firefly can apply coeffect analysis, escape analysis, and memory model optimization to reactive code because signals are compiled, not interpreted.
3. **Headless primitives for true native rendering** - Dioxus has shadcn/Radix-modeled components, but they render through a WebView. No framework with a native rendering path (Floem, Xilem, KiteUI) has the Kobalte/Radix separation of behavior from appearance.
4. **Migration path from production web stack** - No native framework provides a progressive migration from an existing web application framework.
5. **F# type system for UI** - Phantom type markers, computation expressions, and algebraic data types applied to component composition.

## Navigation

- Previous: [05_wren_migration.md](./05_wren_migration.md)
- Next: [07_elmish_signal_hybrid.md](./07_elmish_signal_hybrid.md): Elmish-Signal hybrid programming model
- Back to: [README](../README.md)
