# 07 — Pure domain transitions and selective UI state

Design direction, September 2026. Reducers and local signals are complementary options. No production WREN implementation of the full proposed hybrid is claimed.

An Elmish-style pure transition can produce a new immutable domain model and commands. Typed selectors expose read-only projections to the mounted incremental UI. Changing a selected value invalidates its dependent bindings or area stages. This does not require rerunning every view or diffing a complete visual tree.

```text
command / event → serialized domain transition → model revision
                                           → selective projections
                                           → owned reactive areas
```

Local interaction state can belong to the component/area: draft text, expanded state or transient animation. Business authority and persistent state belong to the relevant domain. A large application can use several domain stores with explicit communication instead of one global reducer.

A selector adapter needs a defined equality policy, coherent publication of related fields, disposal and revision identity. Structural sharing and collection deltas can reduce work, but comparisons and selectors still have costs. Do not describe a whole-model structural comparison as automatically constant or bounded independently of data size.

Retained collections distinguish key identity from payload updates. Same-key replacement must update current data while preserving the intended local control state. Removal retires the item scope and prevents late async work from modifying a replacement instance.

The state adapter does not replace the area engine. Native areas still decide which measure/layout/paint/composition stages to recompute. DOM realization still needs correct reactive bindings and ownership. A pure reducer is a state discipline, not a rendering algorithm.

Cross-domain and cross-machine state uses messages, snapshots and deltas with explicit authority and failure semantics. Local batching is not a distributed transaction. Coalescing latest telemetry can be valid; coalescing repeated commands may lose user actions. Reconnect, stale data and pending/rejected commands need explicit policy.

Use reducer adapters where traceability, replay and domain invariants justify them. Keep small local components simple. The same quiet functional authoring and optional CEs should work with either choice. See the [review](08_ui_model_reconsideration.md).
