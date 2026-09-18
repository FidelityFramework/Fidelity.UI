# 06 — Lessons from prior art

Research alignment, September 2026. These are references to study, not adopted dependencies. The [full review](08_ui_model_reconsideration.md) includes exact source locations and limits of the inspected snapshots.

| Reference | Lesson to retain | Constraint to avoid importing by accident |
|---|---|---|
| Fable.Ripple | Quiet signal-aware composition; structural switching; optional signal CE; keyed scopes | Eager live DOM, ambient global ownership/scheduling, mount without an unmount lifetime |
| Fable.Ripple.Form / Plain | Reusable field/list behavior beneath replaceable presentation | The form render contract still returns DOM items; list caching uses local item identity |
| Jimmy Byrd's ColdTasks / IcedTasks | Explicitly startable, repeatable descriptions of work; foundational influence on Fidelity's cold default | Cold tasks alone provide neither cached incremental results nor dependency invalidation |
| Partas.Solid | Typed facade and compiler transformation into JSX/Solid; useful output fixtures | Fable-specific plugin and Solid runtime semantics are not a native backend |
| Fabulous | Typed composition, reducers, local components, subscriptions and renderer separation | Its component diffing, positional identity assumptions and CE shape are not requirements |
| ReactiveElmish.Avalonia | Pure reducers plus selective retained-view bindings; explicit dispatcher ownership | Avalonia/.NET mechanics and UI-thread policy are target-specific |
| FSharp.Data.Adaptive | Demand-driven graph evaluation, dynamic dependencies, collection deltas, CE/combinators | Weak-reference/GC lifetime and .NET locking are not MCU defaults |
| Jimmy Byrd's WebSharper ViewCE | Source adaptation and applicative composition of reactive values | `View<'T>` is a changing value, not a visual element; builder syntax does not establish mounted lifetime |
| AdaptiveSlop | Pull-oriented computation, explicit graph ownership and functional terminal views | Collaborative experimental code; inspected TUI timers conflict with its newer core ownership rule |
| Solid | Fine-grained bindings, reactive owners and framework compilation | Effect timing, batches and cleanup must be adapted rather than assumed identical |
| LVGL | Embedded control/rendering scope, partial buffers, flush completion and thread ownership | A C wrapper or internal clone is not the chosen new native-area architecture |

Ripple already offers both quiet DOM calls and a signal CE. That is evidence that surface notation and reactive semantics are separable. Its DOM constructors create live objects immediately; a clean native/portable API should instead establish owned mounting. Its keyed row reuse also illustrates why key identity and current payload updates need separate contracts.

Maxime's description of Plain as something to use and outgrow sharpens the design-system direction: replace tokens, field presentation or list markup while retaining the appropriate controller behavior. `Form.listWith` is particularly useful ownership prior art. Fidelity's shared field contract should preserve editing, validation and keyed lifetimes across native/DOM presentations; Ripple.Form's `DomItem` callbacks do not themselves establish that portable boundary. See the [design-system draft](04_design_system.md).

The framework author's ColdTasks influence is a separate, foundational commitment: descriptions start cold and `Incremental<'T>` supplies demand-driven reuse with invalidation. [IcedTasks' builder guide](https://www.jimmybyrd.me/IcedTasks/Explanations/Choosing-a-builder.html) distinguishes deferred start from caching and scheduling. Clef already distinguishes `Cold`, `Lazy` and `Incremental`, including deferred graph construction through `Cold<Incremental<'T>>`. Evaluate reference libraries against that contract; eager activation is not the default to preserve.

Partas CEs are compiler input. Solid supplies its reactive behavior. Ordinary constructor calls can also be compiler-recognized, so native reach does not make CE syntax mandatory. A CE may still improve scoped resources or dependent work; compare those use cases rather than only trivial layouts.

Fabulous supports local components and local MVU, not just one global application model. ReactiveElmish demonstrates that reducer updates need not rebuild the entire view. Adaptive shows why large incremental collections deserve richer semantics than replacing a scalar list value.

Signals are not a guarantee of universally lower cost. Fine-grained graphs consume nodes, edges, equality checks and scheduling work. Native areas may profit from recomputing cheap pure paint descriptions while retaining expensive control state. Compare leaf, area and hybrid strategies on measured workloads.

This document intentionally avoids ranking an expansive framework catalog by unverified speed, maturity or novelty claims. The relevant architectural question is whether each borrowed pattern helps define identity, invalidation, lifetime, scheduling and target behavior for Fidelity's new engine.

Jimmy's [published ViewCE proposal](https://github.com/dotnet-websharper/ui/issues/263) and [TypeSafeViewEngine](https://github.com/TheAngryByrd/TypeSafeViewEngine) are attributable references for reactive CE composition and typed form paths respectively. [AdaptiveSlop](https://github.com/TheAngryByrd/AdaptiveSlop) includes his initial/TUI work and substantial later core work by Angel Munoz. These findings favor evaluating visual, reactive and area-workflow CEs separately. The full review records source pins, attribution and integration limits; no claim is made that the quoted chat identifies one particular implementation.
