# 05 — WREN and native convergence

Design direction, September 2026. Both WREN and native UI work are experimental. The objective is a shared semantic API with a new native reactive-area realization and a browser/DOM realization, rather than preserving one experiment as a production architecture.

WrenHello currently compiles F#/Partas.Solid through Fable, Solid and Vite, embeds the generated HTML, and runs it in a Composer-native WebKitGTK host. Commands and events cross a bounded ASCII script-message bridge. Its BAREWire/WebSocket path and Composer Clef-to-JavaScript/JSX path remain proposed. The current example does not establish Windows/macOS/mobile host coverage or a distributed signal system.

Current HelloWayland uses a separate typed native CPU path, bypassing this Fidelity.UI scaffold. It demonstrates owner-affine presentation, scoped borrowed pixel work and synchronous parallel rasterization. It does not demonstrate independently scheduled reactive areas.

Shared application work can include pure domain types/transitions, typed commands, selectors, portable semantic components and resource policies. Existing HTML/CSS-dependent views and Partas compilation behavior do not become portable merely by changing widget names. Target-specific assumptions must be made explicit.

Two browser realizations deserve bounded comparison:

- A Solid adapter realizing admitted state/binding/lifetime semantics through Solid primitives and compilation.
- Clef-owned incremental behavior with direct DOM construction/update operations.

Ripple is a source reference for the second, not an adopted runtime. Existing WREN packaging helps exercise both; it does not decide their semantic suitability. Neither path currently proves full Clef lowering or parity.

Use a native-led area contract and the same nontrivial authored components in both hosts. Validate keyed identity/current payload, conditional scope disposal, effect ordering, async races, events and final rendered behavior. Browser layout/paint mechanics may differ while preserving admitted control behavior. Native damage regions are not browser repaint promises.

Cold activation and demand are conformance requirements for either browser realization. Constructing a description must not create DOM nodes or start subscriptions/validation; a mount or explicit preparation owner activates the appropriate sinks. An adapter must preserve undemanded derivations and inactive branches without importing a library's eager setup as Fidelity's default. It must also support background observation that deliberately keeps selected values current while their views are inactive. Test discarded descriptions, separate instances, prepared-instance attachment, released demand and reacquisition through the final embedded artifact.

An application may keep a core in-process, use native workers, host a WebView, or communicate with remote services. Moving to native does not inherently remove IPC. Cross-domain state remains versioned projections and commands; multiple windows/documents have distinct lifetimes and heaps unless an explicit service shares data.

Component-by-component mixed rendering is a separate interoperability capability. It needs host composition, focus/input/accessibility routing and ownership. Shared signal terminology alone does not implement it. Prefer a small side-by-side conformance app before promising incremental production migration.

Close/reopen should distinguish workload, subscription and view lifetime. A new observer receives a snapshot and subsequent updates. Packaging several pages into one asset bundle does not establish shared state or window lifecycle.

See the [review](08_ui_model_reconsideration.md) for the experimental sequence and the source evidence. This direction replaces earlier claims of unchanged application code or identical runtime semantics without adapter validation.
