# 01 — Reactive state, events and stabilization

Design direction, September 2026. This document describes required UI semantics, not an implemented signal engine. The current Clef specification defines `Signal`, `Memo`, `Effect` and `Batch` as a surface over Observable/Incremental. Fidelity.UI should consume that contract instead of introducing an unrelated native runtime table.

| Concept | UI role |
|---|---|
| Signal | Settable current state with change detection |
| Memo / Incremental | Pure, demand-driven cached derivation |
| Effect | An owned side-effecting sink, with defined execution and cleanup |
| Observable / event | Occurrences whose multiplicity/order must be preserved unless explicitly transformed |
| Batch | Local stabilization boundary; not rollback or a distributed transaction |
| Selector | Read-only projection of domain state with an explicit equality policy |
| Resource | Owned asynchronous operation with pending/result/error and stale-completion policy |

Cold is the default construction posture for proposed UI components and resources. It has several distinct meanings at the primitive level: `Cold<'T>` defers repeatable work without caching; `Lazy<'T>` computes and caches once; `Incremental<'T>` adds dependencies, invalidation and cutoff. `Cold<Incremental<'T>>` defers graph construction and dependency registration as well as result evaluation. UI syntax should preserve these contracts without requiring authors to wrap every control manually.

The existing `Effect.create` contract registers an active, always-demanded sink. A cold component therefore defers that call until owned activation; constructing its description does not create an effect and then suppress its promised behavior. Pure stale nodes without observation remain unevaluated. Render, validation, background service and preparation sinks establish demand according to their own lifetimes. First presentation is the default activation point, but explicit preparation can admit an owned instance earlier.

Reactive callbacks are ordinary closures. FFI function pointers belong at the platform boundary. The compiler must distinguish actual reactive reads from handles captured only to write, pass elsewhere, or invoke later. Lexical capture alone is not a complete dependency analysis. Static dependencies can specialize; data-dependent branches and keyed instances need supported dynamic realization or a clear diagnostic.

Proposed usage of the specified signal vocabulary:

```fsharp
let count = Signal.create 0
let doubled = Memo.create (fun () -> Signal.get count * 2)
Signal.update count (fun n -> n + 1)
```

A mounted binding must preserve a source or deferred read. Reading a signal into a string during setup produces a snapshot unless a supported transformation explicitly creates a binding. The component body is not automatically the correct subscriber for every property it constructs.

Before target parity is claimed, resolve and test read-after-write, reads within nested batches, initial effect timing, effect ordering, reentrancy, writes during effects, failure and cycles. Logical stabilization and frame presentation are separate: consistent state can be available before the next paint. Async suspension does not silently extend a local batch.

Demand is not the same as pixel visibility. Layout, focus, accessibility and form submission can require work for content that is offscreen. A suspended visual projection may release paint demand while retaining editing state and a separately required validation observer. Specify retention and demand release explicitly; cold work can still retain captured data or cached resources.

Applications should be able to choose how much work stays ready:

| Policy | What remains ready | Cost moved between background and request time |
|---|---|---|
| On demand | Cold description; other state only if separately owned | Graph activation and required computation occur on request |
| Retain cached state | Existing instance and last cached result, possibly stale | Avoids reconstruction; invalidated derivations still need validation/recomputation |
| Keep a projection current | A background/service observer continues demanding selected derived values | Pays ongoing update cost to reduce switching latency |
| Prepare selected view stages | An admitted preparation scope demands controls, measurement or render preparation within known constraints | Pays additional memory/work; presentation still checks current constraints and revisions |

These policies can compose across one application; they are not mutually exclusive global evaluation modes. A time-series service can ingest into bounded history and keep a windowed projection current while the corresponding chart's layout and painting remain inactive. Switching views attaches to that current projection and demands the remaining stages. Input/event ingestion is a separate lifetime: keeping a derived value cold must not silently lose samples the domain promised to retain.

An eager operational policy can be expressed by keeping an explicit observer active. It preserves the demand-driven core: the background observer supplies demand even when no surface displays its value. Releasing a foreground observer does not release the service's observation. A preparation scope needs its own owner, retention/eviction policy and bounded scheduling budget; attaching its prepared instance must not repeat setup or duplicate subscriptions. Fresh data alone does not mean that text shaping, native buffers or DOM layout are already prepared.

The hardware/unikernel deployment profile should supply concrete memory, execution and I/O budgets for these choices. Measure first activation, cached-but-stale activation and switching to a kept-current projection, including background load and frame deadlines. Specify the allowed freshness lag and coherent revision of background snapshots. Reading the latest accepted snapshot and requesting a fresh value that may need computation or waiting are different contracts. Hardware control helps define the envelope; bounded queues, scheduling, device completion and admission still determine whether the target latency is met.

Preparation also needs explicit priority, error and cancellation policies. Anticipated navigation must not execute business commands. Device/network access or asynchronous validation can run early only under the admitted effect policy, while pure projection work can be warmed without inventing a user action. Background budgets must leave room for interactive and other required work.

Cutoff suppresses propagation attributable to an unchanged input. It must not clear an independent invalidation of the same dependent from another input. Diamond graphs and multi-source batches are necessary conformance cases. An effect's `unit` result must not be used to suppress required side effects through ordinary value cutoff.

Signals and actors have a useful structural correspondence, but different contracts. A local owner may contain many signals. Crossing an owner/thread/process boundary uses typed messages or versioned projections. Remote updates have pending/failure/stale semantics; no synchronous distributed getter/setter is promised.

Every mounted component/area has a logical lifetime. Removal detaches observers, invalidates pending work, unregisters handlers and releases retained resources after outstanding users finish. Whole-actor arena retirement alone is too coarse for repeatedly mounted branches. Native scopes/pools and JavaScript host allocation realize the same logical cleanup differently.

A store adapter may expose typed selectors and explicit field operations. Ordinary record access must not be described as deep-reactive proxy access without specified compiler/library support. Large collections need identity and delta contracts; structural comparison has a cost proportional to the data actually examined.

The [review](08_ui_model_reconsideration.md) details the open semantic questions and the prior-art comparison. None of this requires a CE layout surface.
