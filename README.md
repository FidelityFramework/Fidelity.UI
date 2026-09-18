# Fidelity.UI

**A shared declarative UI model for a new Clef-native reactive-area engine and browser/WebView realization.**

Status: experimental implementation and architectural design. The source in this repository is a small descriptor/rendering scaffold. It does not implement the architecture described below. WrenHello and HelloWayland are experimental hosts with useful, bounded implementation evidence.

The September 2026 direction favors quiet functional composition: functions, lists, typed bindings and modifiers. Optional computation expressions can express the same construction semantics or make scoped workflows easier to write. Concurrency is defined by ownership and execution contracts, independently of layout syntax.

Cold construction and demand-driven incremental work are the default design posture. A description becomes live through explicit owned activation, normally at first mount; a service or preparation scope can deliberately establish demand earlier. A background observer can keep a time-series projection current while its visual areas remain inactive. Unobserved derivations need not run merely because their inputs changed. Activation, observation and disposal must survive native and JavaScript lowering. This follows Fidelity's `Incremental<'T>` foundation, rather than adopting eager activation from a reference library.

The primary native direction is a **reactive-area engine**. An area owns mounted identity, reactive inputs, layout/paint results and a disposal scope. Changes invalidate the affected stages of measurement, layout, painting and composition. Areas need not correspond one-to-one with actors, threads, memory arenas, framebuffers or OS surfaces. Leaf bindings, area recomputation and hybrid update policies remain experiments to compare.

The portable layer describes controls, layout, events, identity and resource lifetimes. Native realization produces retained visual output and damage updates; browser realization maps the same admitted semantics to DOM content. Native damage rectangles are not a promise about browser repaint boundaries. HTML/CSS and device-specific features remain explicit capabilities.

Clef's `Signal`, `Memo` and `Effect` surface is specified over `Observable` and `Incremental`. Signals represent changing state; events/commands represent occurrences. Actors can own graphs and connect execution domains, but mailbox coalescing does not itself provide coherent signal stabilization. The implementation and conformance work for this contract remains substantial.

Fabulous, ReactiveElmish.Avalonia, Partas.Solid, Fable.Ripple and FSharp.Data.Adaptive are research references. They are not adopted dependencies. In particular, Ripple's quiet syntax is valuable inspiration without making its eager DOM construction or global scheduler the native implementation model. LVGL chiefly informs the widget/component, style and event model; Solid supplies complementary component-composition lessons. Native controls and layouts should be as easy to compose as web components, including optional declarative motion admitted by each product's capabilities and budgets.

**Current source scope:** constructors and modifiers create flat descriptors; containers currently retain only child counts; generic child retention/layout is absent. SVG presentation and experimental splash composition exist, while label/box paths are console placeholders. The current HelloWayland executable uses its own typed rendering path rather than this scaffold. No production UI or cross-target API parity is claimed.

The [architectural review](docs/08_ui_model_reconsideration.md) gives the source evidence, alternatives, native-area model and narrowing plan. The standing design documents are aligned with that direction:

- [Architecture](docs/00_architecture.md)
- [Reactive semantics](docs/01_signal_system.md)
- [Components and authoring](docs/02_component_model.md)
- [Reactive areas and rendering](docs/03_rendering_backends.md)
- [Behavior and design system](docs/04_design_system.md)
- [WREN and native convergence](docs/05_wren_migration.md)
- [Lessons from prior art](docs/06_prior_art.md)
- [Reducers and selective state](docs/07_elmish_signal_hybrid.md)

Next evidence: a native control panel with independently changing areas, text-driven resize, overlapping content and correct partial redraw; then the same semantic components in a DOM host, independent execution domains and a bounded physical embedded target. These are planned acceptance gates, not completed capabilities.

Dual-licensed under Apache 2.0 and a commercial license. See [LICENSE](LICENSE) and [Commercial.md](Commercial.md).
