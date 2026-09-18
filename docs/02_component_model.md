# 02 — Components, quiet composition and optional CEs

Design direction, September 2026. Examples are proposed notation, not executable APIs in the current scaffold.

Quiet function/list composition is the preferred starting point. Typed properties and modifiers should prevent invalid combinations. A portable semantic vocabulary describes intent; an HTML extension can expose DOM-specific features without making HTML the native object model.

```fsharp
let counter =
    Ui.component (fun () ->
        let count = Signal.create 0
        Ui.column [
            Ui.button [
                on.activate (fun _ -> Signal.update count (fun n -> n + 1))
                Ui.text "Count"
            ]
            Ui.output count
        ])
```

`Ui.component` describes a cold factory for an owned instance, normally activated at first mount. Its local state belongs to that logical identity. An explicit preparation scope may activate the instance earlier; later presentation attaches to it without repeating setup. `Ui.output` receives a reactive source through a read-only binding contract; a plain eagerly computed value remains a snapshot. Explicit `textSignal`/property-binding operations may precede convenient overloads in an implementation.

The default mount should establish an owner and return an unmount lifetime. Description construction is cold: it must not create live platform objects, active subscriptions, timers or validation requests. Explicit earlier activation belongs to a service or preparation owner, with separate lifetime from its presentation attachment. Independent mounts of a reusable description create distinct local state unless an existing shared/prepared instance is explicit. Source closures are natural event handlers; their foreign callback adaptation must preserve captures and lifetime. Current experimental host restrictions are not a completed generic callback implementation.

Functions and lists can compose cold descriptions. Their arguments must themselves preserve deferred work: passing an already-started operation cannot make it cold. A CE's `Delay`/`Run` must likewise return the intended deferred description instead of activating it as the block completes. Keeping a plan is not necessarily allocation-free, and cold construction alone neither caches a result nor defines mounted identity.

Setup runs once per activated logical identity. Pure area projection/layout/paint functions can run again when invalidated and demanded. Local control state must survive those updates according to identity, not according to incidental expression position. This separates instance lifetime from the native area's chosen recomputation granularity and from each presentation attachment.

Activation establishes demand for the required outputs; it need not force every descendant or inactive branch. An invalidated pure projection runs when an admitted consumer needs it, including a background observer keeping data current. Visibility-based suspension is a policy, not unconditional disposal: focus, layout, accessibility, submission or explicit preparation may still demand work. Replacing a presentation also need not recreate its field state or validation owner.

| Operation | Required meaning |
|---|---|
| Static property | A fixed value |
| Reactive property | A source/deferred read and typed target setter |
| Conditional branch | Condition plus deferred branch ownership; explicit retention/disposal policy |
| Keyed collection | Stable key plus a separately updateable current item payload |
| Area | Stable visual identity, scope, constraints and stage outputs |
| Execution boundary | Placement/transfer/admission policy, independent of visual nesting |

Ordinary `if`/`for` during construction and dynamic conditional/keyed operations are different. A compiler may make dynamic syntax concise, but must preserve dependency, identity and disposal semantics. Duplicate keys, same-key replacements and index-dependent state need defined behavior.

An optional layout CE should elaborate to the same semantic construction operations. A CE may also be especially useful for resource acquisition, cancellation, scratch lifetimes or dependent calculations. Keep workflow meanings explicit: reactive `let!`, acquiring a mounted resource and awaiting async work should not be accidentally conflated.

Normal lexical `use` can dispose a resource when a setup function returns. Mounted ownership needs an explicit registration or a builder whose lifetime behavior is specified. Likewise, `and!` can express independent inputs without promising threads.

The same function API can describe an area:

```fsharp
Ui.area "telemetry" (fun () ->
    Ui.column [ Ui.output temperature; Ui.output pressure; controls ])
```

Placement may later be requested through a policy. A local captured factory does not automatically become a valid cross-process closure. Typed transferable data, owner affinity and source availability remain separate obligations.

Compare function and CE forms on forms, keyed editing, resources and concurrent areas before freezing either facade. Readability and diagnostics matter as much as line count. See the [review](08_ui_model_reconsideration.md).
