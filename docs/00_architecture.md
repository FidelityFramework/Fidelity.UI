# 00 — Architecture: shared semantics and native reactive areas

Design direction, September 2026. The described engine is not implemented by the current Fidelity.UI scaffold. The [review](08_ui_model_reconsideration.md) records the evidence and open decisions.

Fidelity.UI should provide one semantic UI model with quiet functional composition as its default. Optional CEs elaborate to the same construction operations; separate workflow builders may express resource acquisition or asynchronous work. Neither syntax determines threading.

Cold, incremental composition is the architectural starting point. Constructing a UI description must not mount controls, subscribe to producers or start validation. Explicit owned activation admits demand, normally at first mount or deliberately earlier through a service/preparation scope; subsequent invalidation recomputes demanded stale work. Description storage can still have a cost. This is a policy for the UI contract, not a claim that every ordinary Clef expression has non-strict evaluation.

Keep cold construction distinct from incremental evaluation: `Incremental<'T>` caches and invalidates a demanded result; `Cold<Incremental<'T>>` can defer constructing its graph at all. Mounted effects are active sinks until their owner retires them. Removing visual demand does not by itself cancel a separately owned service or an acknowledged command.

Demand can be kept active without surface visibility. A background observer can maintain a current time-series projection for fast view switching; an explicit preparation scope can warm selected view stages. Retaining a stale cache, maintaining current derived data and preparing layout/paint are different resource choices. Budget background work and retained memory alongside first-demand latency using the deployment's hardware and scheduling envelope. The [reactive-state draft](01_signal_system.md) records these policies.

The primary native realization is a new reactive-area engine. An area is a mounted visual subtree with stable identity, an owned scope, reactive inputs, incoming constraints and retained output. It organizes incremental measurement, layout, painting and composition. It need not have its own thread, actor, native surface or framebuffer.

Four structures have different responsibilities:

| Structure | Responsibility |
|---|---|
| Semantic/control tree | Controls, composition, focus, input and accessibility relationships |
| Dependency graph | State dependencies, invalidation, demand, equality/cutoff |
| Spatial composition/damage structure | Geometry, overlap, clipping, cached output and affected pixels |
| Execution ownership graph | Serialized mutation, optional workers, supervision and communication |

A shared input can affect several areas. A layout change can propagate to parents and siblings. Independent graph computations do not establish independent pixel writes when areas overlap. Explicit contracts connect these structures.

The portable semantic operations include components/scopes, static and reactive properties, events, conditional branches, keyed collections, resources, areas and execution-domain boundaries. The compiler should preserve their identities, reads, effects and lifetimes through the PSG/codata and portable middle end. A full interpreted tree or a particular intermediate dialect is not required; runtime dynamic instances and retained state still need representation.

Clef's signals are a surface over its Observable/Incremental contract. State and event semantics remain distinct. Owner-local graphs stabilize coherent state; actors or equivalent domains own substantial work and communicate through explicit messages/projections. An actor can contain many reactive nodes. Batching controls local stabilization; mailbox coalescing is a separate delivery policy.

Native realization schedules affected layout and paint work and presents accepted output. Browser realization maps portable controls/areas to DOM content and lets the browser perform its admitted layout and painting. Native memory regions and pointers remain backend details, unavailable in portable JavaScript-facing code. Shared semantics do not imply identical allocation mechanisms.

Embedded profiles admit bounded topology, queues, storage and dynamic content. Desktop/mobile profiles can admit richer capabilities. A target diagnoses unavailable required behavior or requires an authored alternative; unsupported capabilities must not silently approximate the API contract.

State discipline is independent of rendering: local signals and optional pure reducers with selective projections can coexist. Network state is a local projection of explicitly replicated domain state, not a synchronous signal spanning machines.

The first implementation experiment should compare fine leaf bindings, pure area recomputation and a hybrid. Mount/setup lifetime remains stable while a pure area update may run repeatedly. Choose update granularity from memory, scheduling, layout and paint measurements rather than making “components run once” an absolute ban on recomputation.

See [reactive semantics](01_signal_system.md), [component model](02_component_model.md), and [area rendering](03_rendering_backends.md).

## Compiler planning alignment — 2026-09-20

[Composer M-01](../../Composer/docs/PRDs/M-01-DialectAdmission.md) treats the piped
FP and CE forms here as triangulation for complete language expression. UI
quotations and declarations contribute target, ownership, demand and execution
facts to Baker; Alex witnesses the settled graph in the form appropriate to the
selected native or browser backend. No rendering-specific semantic dialect is
implied. Numeric selection governs geometry/arithmetic eligibility and precision;
parallel layout/paint also requires access, publication, completion and progress
contracts under the actual display/platform profile.

Planned acceptance pairs serial and parallel results, bounded-device resource
failures, target capability changes, stale work after invalidation and ownership
retirement. Source-related CCS diagnostics and design-time evidence must agree
with compilation. HelloWayland's working CPU renderer is an oracle, not a proof
of all targets or the completed actor model. Native Fidelity.UI remains primary;
an optional Farscape/LVGL adapter implements the shared interface. The
[waypoints](../../Composer/docs/Language_Coverage_Waypoints.md) record coordinated
revisions; this synchronization adds no engine implementation or test evidence.
