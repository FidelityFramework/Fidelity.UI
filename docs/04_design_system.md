# 04 — Semantic behavior and design system

Design direction, September 2026. These facilities are planned; the current scaffold does not implement a headless control library or theme engine.

Keep control behavior separate from appearance. Portable controls describe roles, states, actions, focus relationships and input behavior. Browser realization maps that intent to semantic HTML/ARIA where appropriate; native realization needs platform accessibility integration. ARIA attributes themselves are not a universal accessibility model.

Provide a replaceable reference control kit. Authors should be able to adjust tokens, replace one field's presentation, or compose a different control while retaining the behavior appropriate to that control. Fable.Ripple.Form.Plain is useful prior art for this progression: its field constructors and `listWith` reuse form machinery beneath custom markup. Its current form core still returns DOM items, so Fidelity needs a deeper portable boundary rather than a renamed browser renderer.

For forms, distinguish typed behavior from its presentation. A field owns raw editable input, parsing, a valid domain result or errors, dirty/touched state, validation status and reset/commit actions. Raw text can be temporarily invalid without corrupting the domain model. Semantic bindings carry label/help/error relationships, focus requests and enabled/read-only behavior to either native controls or DOM. Replacement presentation must preserve those obligations; custom drawing or markup does not inherit accessibility correctness automatically.

Define field/list behavior as cold descriptions with explicit owned activation. Allocate editing state and activate subscriptions or asynchronous validators when the logical field scope is admitted. Redraw, reflow and presentation replacement preserve that scope unless the application requests a reset. Removing the logical field retires it; detaching a presentation can instead leave the scope retained by an explicit owner. A delayed validation result must match both the owner generation and current input revision before publication. Cancellation of a request and rejection of an obsolete result are separate responsibilities.

A form-list controller should own stable logical keys, separately updated item payloads and positions, add/remove actions, aggregate results and child cleanup. Its presentation consumes those bindings. Object reference identity is insufficient for refreshed immutable records or network snapshots; specify duplicate keys and remove/reinsert behavior. A field or list need not have a dedicated reactive area or thread. Long-lived kiosk/MCU checks must bound retained metadata as well as subscriptions through repeated insertion and removal.

Design tokens describe color, spacing, typography, density and state variants. They may lower to CSS, native retained values or embedded constants. A theme change invalidates the stages affected by each token: color may repaint, while font/spacing can change measurement and layout. Theme switching is not necessarily a paint-only operation.

A visual designer can use alternate presentations for selection and drag handles. Editable design documents additionally need persistent identities, typed property/layout metadata and validated edit/undo operations; opaque render callbacks alone do not supply them. Keep that metadata an optional authoring capability so small embedded applications do not require a general runtime schema or designer engine.

A reactive area's semantics and hit geometry must agree with the accepted visual/layout revision. Headless behavior includes more than a state machine: text editing requires IME/composition, clipboard and selection; dialogs require focus/modality; mobile adds virtual keyboards, gestures, safe areas and lifecycle behavior.

Start with buttons, text/output, a real editable field, simple layout and a keyed list. Prove keyboard/touch activation, disabled state, focus retention, accessible naming and disposal before expanding the control catalog. Custom native drawing is the primary direction, so these responsibilities cannot be delegated implicitly to a browser.

Portable capability profiles declare which behaviors are available. HTML-specific document features and native/device integrations remain explicit extensions. Missing required behavior should be diagnosed or have an authored alternative, rather than silently changing semantics.

DaisyUI, Kobalte/Radix and native platform conventions can inform tokens and behavior. No claim of automatic CSS-class conversion, universal visual identity or inherited accessibility compliance is made. See the [architectural review](08_ui_model_reconsideration.md).
