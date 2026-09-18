# 03 — The native reactive-area engine

Design direction, September 2026. Building a new Clef-native reactive-area engine is the primary native objective. HelloWayland supplies experimental presentation and parallel raster evidence; it is not this engine. LVGL chiefly informs the component model, and also offers rendering/resource lessons; its runtime and internal architecture are not mandated.

An area has mounted identity, a disposal scope, reactive inputs, incoming constraints and retained outputs. Its outputs may include measurement/layout, paint commands, hit regions and accessibility semantics. An area is not necessarily a framebuffer, actor, OS surface or rectangular clip.

```text
input/messages → stabilize demanded work → affected measure/arrange
              → affected paint/semantics → versioned render work
              → raster/composition → owner-affine presentation
```

The dependency graph identifies changed computations; spatial damage identifies pixels that must be refreshed. These are separate. Text/font changes may invalidate parent and sibling layout. Fill color normally affects paint. Transform/opacity can be composition-only only when the retained layer/group representation preserves the correct visual semantics.

Area descriptions begin cold. Owned activation, normally at first mount, admits demand for selected outputs; an input change marks dependent work stale without requiring evaluation of every unused branch. Frame preparation demands the necessary visual result, while layout, input and accessibility may have additional consumers. Suspension can release selected demand and keep retained state under a bounded policy. It must not silently stop an independently owned producer or background observer.

An application can keep derived data current for an inactive view, or explicitly prepare some of its measure/paint work in advance. These are separate choices: a current chart projection does not imply prepared geometry or pixels. Preparation must run under an admitted owner and resource budget, preserve platform affinity, and record relevant data/constraint/surface revisions. Switching to the view can reuse compatible results and demand the remaining work. Resize or resource eviction may invalidate that preparation; measure this path alongside first-use latency and active-frame work.

Parent constraints and child measurements need a staged, bounded layout protocol. Each result records the applicable constraint/scene revision. Avoid arbitrary cyclic reactive fixed-point layout. Fixed or independently constrained panels are useful early parallel boundaries.

Compare direct leaf bindings, pure area recomputation and hybrid retained controls/display output. An area update can rerun pure calculations without remounting its controls. Fine-grained graphs save some computation but cost nodes, edges and scheduling; area recomputation trades those costs against repeated work.

For an initial concurrent experiment, versioned immutable area results are easier to reject atomically than partially applied patch streams. Share retained resources rather than copying every output. Ordered changesets are another candidate for large content, but need base-version checks, ordering and recovery. Couple visual, hit-test and semantic metadata to compatible revisions.

Damage must cover old and new visual bounds, clipping, transforms, shadows, overlap and z-order. Removal and movement must erase old pixels. Bound rectangle/tile accumulation and allow conservative merging or full-area redraw. Cache output where useful; do not require one full-size buffer per area.

Fades, eased movement and state transitions should use the same component properties on native and DOM targets. A cold motion description acquires clock/frame demand only when its owner activates a transition. Sample progress from elapsed monotonic time rather than assuming a fixed frame rate. Define duration/easing, retargeting from the currently presented value, cancellation versus completion, and suspension/resumption policy. A finished or retired transition releases its clock demand; an explicit repeating animation remains active work. Motion changes presentation values without rewriting the authoritative application target on each frame. See [LVGL animation](https://lvgl.io/docs/open/9.4/details/main-modules/animation.html) for useful timing, path and lifecycle concepts; Fidelity's ownership contract remains its own.

Width or padding animation can invalidate measurement and sibling layout; color can invalidate paint. Group opacity or transforms may reuse cached content only if clipping, blending and retained-layer semantics permit it. A layer costs memory and may need rerasterization when its content changes. Bound layer count/bytes, uploads and pending frames; never allocate a texture per component by default. DOM realization may use browser animation facilities where their timing and cancellation can preserve the contract. GPU completion and compositor release still govern buffer reuse after a transition is cancelled.

Execution can progress through explicit tiers:

| Tier | Placement |
|---|---|
| Cooperative | Independently owned areas scheduled on one executor; useful baseline and MCU realization |
| Worker jobs | Pure calculations or proven disjoint borrowed raster writes, committed by the presentation owner |
| Independent domains | Owned graphs exchanging bounded versioned snapshots/messages |
| Process isolation | Transferable data/display output with resource leases and an explicit composition protocol |

Visual nesting does not prove dependency isolation or disjoint output pixels. Overlapping alpha content requires ordered composition. Unsupported placement policies require a diagnostic or explicitly selected fallback.

An accepted result must match its mount generation and relevant input/layout/surface revisions. Cancellation is insufficient to prove no worker still accesses memory. Buffers become reusable only after both producer work and external display/GPU/DMA use have retired. HelloWayland's distinct worker-join and compositor-release handling is relevant evidence.

Platform integration supplies windows, surfaces, input and native rendering facilities: Wayland on Linux; appropriate AppKit/Core Animation/Metal integration on macOS; Win32 and a selected rendering stack on Windows. Mobile adds its own lifecycle, input and presentation contracts. These are target plans, not implemented backend coverage in this repository. Custom drawing also requires text shaping/editing, focus and accessibility integration.

The browser adapter maps admitted areas/controls to DOM structures. The browser performs layout and painting; a native damage rectangle does not map to a guaranteed browser repaint boundary. Shared API parity concerns behavior, identity, binding and lifecycle within declared capabilities.

Embedded realization needs bounded nodes/edges/queues, partial buffers, display flush completion, device-specific pixel formats and measured resource limits. It must not require a desktop thread pool or a full framebuffer per area.

The shared component language admits different product profiles:

| Proposed profile | Presentation policy | Evidence required |
|---|---|---|
| STM32H7 instrument | Immediate state changes; decorative UI motion omitted by product choice | Bounded controls/paint alongside separately active audio; this omission makes no claim that the chip cannot animate |
| Sweet Potato KeyStation panel | Selected fades and eased transitions on the selected Waveshare 7.9inch HDMI LCD, SKU 17916 | Accepted portrait scanout and logical/input rotation, native Meson display and restricted Mali rendering, measured layer memory and latency |
| GPU desktop | Richer motion and retained composition within declared limits | Backend capabilities, frame pacing, resource completion and reduced-motion behavior under load |
| WREN | The same authored transition intent through admitted browser facilities | Compatible interruption, lifetime and timing behavior in the bundled browser/WebView artifact |

Omitting decorative UI motion does not disable live meter updates, sensor filtering or DSP pitch/control smoothing; those serve separate application behavior and timing contracts.

The [Sweet Potato design](09_sweet_potato_keystation.md) selects a CPU reference renderer, native Meson HDMI/input integration and then a restricted Mali-450 renderer for the proposed KeyStation unikernel. Linux supplies a separate hosted route and hardware reference. Video decoding, display scanout and GPU drawing have distinct drivers and acceptance gates. These profiles vary presentation capability and budgets while preserving component identity, actions and application semantics.

The decisive native acceptance case combines telemetry, local editing, text-driven resizing, moved/removed translucent content and repeated disposal. Compare partial-update output with forced full redraw; instrument avoided measurement/paint/raster work and verify stale-result rejection, queue bounds and buffer reuse. See the [review](08_ui_model_reconsideration.md).
