# 04 - Design System: Tokens, Themes, and Headless Primitives

## Design Goal

Separate UI **behavior** from UI **appearance**. A Dialog component knows how to manage focus, trap keyboard navigation, and handle escape-to-close. How it *looks* - colors, corners, shadows, spacing - is defined by the design system. Swapping themes changes the visual language without touching component logic.

This is the Kobalte/Radix + DaisyUI model: headless primitives provide behavior, design tokens provide appearance.

## Headless Primitives

### What "Headless" Means

A headless component provides:
- State management (open/closed, selected/unselected, focused/unfocused)
- Keyboard navigation (arrow keys, tab, escape, enter)
- Accessibility semantics (roles, labels, live regions)
- Focus management (trapping, restoration, initial focus)

A headless component does **not** provide:
- Colors, backgrounds, borders
- Spacing, sizing, layout
- Typography
- Animations

### Primitive Catalog

The following headless primitives are planned, inspired by Kobalte and Radix:

| Primitive | Behavior |
|-----------|----------|
| **Dialog** | Modal/non-modal overlay, focus trap, escape-to-close, backdrop click |
| **Select** | Dropdown, keyboard selection, typeahead, multi-select variant |
| **Menu** | Popup menu, nested submenus, keyboard navigation, trigger variants |
| **Tabs** | Tab list + tab panels, keyboard left/right, lazy/eager panel rendering |
| **Accordion** | Expandable sections, single/multiple open, keyboard navigation |
| **Popover** | Anchored floating content, positioning logic, dismiss on outside click |
| **Toast** | Notification queue, auto-dismiss timer, pause-on-hover, stacking |
| **Tooltip** | Delayed show/hide, positioning, accessible description |
| **Checkbox** | Checked/unchecked/indeterminate, group management |
| **RadioGroup** | Single selection within group, keyboard up/down |
| **Switch** | Toggle with on/off state |
| **Slider** | Range input, keyboard step, min/max/step |
| **Combobox** | Input + listbox, filtering, async search |

### Headless Usage Pattern

```fsharp
// The headless primitive provides behavior via signals
Dialog.Root(open' = isOpen, onOpenChange = setIsOpen) {
    Dialog.Trigger() {
        // Whatever you want the trigger to look like
        Button("Open Settings")
    }
    Dialog.Portal() {
        Dialog.Overlay()
            .background(theme.overlay)
        Dialog.Content()
            .background(theme.surface)
            .cornerRadius(12)
            .padding(24)
            .shadow(elevation = 4) {
            Dialog.Title() { Label("Settings") }
            Dialog.Description() { Label("Adjust your preferences") }
            // ... dialog content ...
            Dialog.Close() { Button("Done") }
        }
    }
}
```

The headless `Dialog` manages:
- Open/close state via the `isOpen` signal
- Focus trap (Tab cycles within dialog content)
- Escape key closes the dialog
- Click on overlay closes the dialog
- Focus returns to trigger on close
- Accessibility: `role="dialog"`, `aria-labelledby`, `aria-describedby`

The `.background()`, `.cornerRadius()`, `.padding()`, `.shadow()` modifiers come from the design system, not the dialog primitive.

## Design Tokens

### What Are Tokens?

Design tokens are the atomic values of a visual design system:

```fsharp
type DesignTokens = {
    // Colors
    Primary: Color
    OnPrimary: Color
    Surface: Color
    OnSurface: Color
    Background: Color
    Error: Color
    Outline: Color

    // Spacing
    SpaceXs: float   // 4
    SpaceSm: float   // 8
    SpaceMd: float   // 16
    SpaceLg: float   // 24
    SpaceXl: float   // 32

    // Typography
    FontFamily: string
    FontSizeBody: float
    FontSizeHeading: float
    FontWeightNormal: int
    FontWeightBold: int
    LineHeight: float

    // Shape
    RadiusSm: float   // 4
    RadiusMd: float   // 8
    RadiusLg: float   // 16
    RadiusFull: float  // 9999

    // Elevation
    Shadow1: Shadow
    Shadow2: Shadow
    Shadow3: Shadow
}
```

### Token Hierarchy

Tokens are organized in layers:

1. **Primitive tokens**: Raw values (`blue500 = Color.hex "#3B82F6"`)
2. **Semantic tokens**: Role-based references (`primary = blue500`, `error = red500`)
3. **Component tokens**: Component-specific defaults (`buttonBackground = primary`, `buttonRadius = radiusMd`)

### Theme Definition

A theme is a complete set of semantic tokens:

```fsharp
let lightTheme = {
    Primary = Color.hex "#3B82F6"
    OnPrimary = Color.hex "#FFFFFF"
    Surface = Color.hex "#FFFFFF"
    OnSurface = Color.hex "#1F2937"
    Background = Color.hex "#F9FAFB"
    // ...
}

let darkTheme = {
    Primary = Color.hex "#60A5FA"
    OnPrimary = Color.hex "#1E3A5F"
    Surface = Color.hex "#1F2937"
    OnSurface = Color.hex "#F9FAFB"
    Background = Color.hex "#111827"
    // ...
}
```

### Theme Delivery

Themes are delivered via the Context system (hierarchical signal propagation):

```fsharp
let ThemeContext = createContext lightTheme

// At the app root
ThemeContext.Provider(currentTheme()) {
    App()
}

// In any component
let theme = useContext ThemeContext
Label("Hello").color(theme.Primary)
```

When `currentTheme` changes (e.g., user toggles dark mode), every component that reads from `ThemeContext` is notified via the signal-actor graph. Only the visual properties that reference theme tokens repaint.

## Semantic Classes

Inspired by DaisyUI's approach of semantic class names over utility classes:

```fsharp
// Instead of specifying every visual property:
Button("Click me")
    .background(theme.primary)
    .color(theme.onPrimary)
    .padding(8, 16)
    .cornerRadius(theme.radiusMd)
    .fontSize(theme.fontSizeBody)

// Apply a semantic style:
Button("Click me")
    .style(Btn.Primary)

// Or with size variants:
Button("Click me")
    .style(Btn.Primary)
    .size(Btn.Lg)
```

Semantic styles are defined as token compositions in the design system:

```fsharp
module Btn =
    let Primary = Style [
        Background theme.primary
        Color theme.onPrimary
        Padding (theme.spaceSm, theme.spaceMd)
        CornerRadius theme.radiusMd
        FontSize theme.fontSizeBody
        FontWeight theme.fontWeightBold
    ]

    let Lg = SizeStyle [
        Padding (theme.spaceMd, theme.spaceLg)
        FontSize theme.fontSizeHeading
    ]
```

## Accessibility

### Native Accessibility

Unlike web ARIA attributes, native accessibility varies by platform:

| Platform | Accessibility API |
|----------|------------------|
| Linux | ATK / AT-SPI2 |
| macOS | NSAccessibility protocol |
| Windows | UI Automation (UIA) |

Headless primitives define accessibility **intent** (this is a dialog, this is a button, this label describes that input). The platform backend translates intent to platform-specific accessibility API calls.

### Keyboard Navigation

Headless primitives implement standard keyboard patterns:
- **Dialog**: Tab cycles within, Escape closes
- **Menu**: Arrow keys navigate, Enter selects, Escape closes
- **Tabs**: Left/Right arrows switch tabs
- **Select**: Arrow keys navigate options, Enter selects, typing filters

These patterns are part of the headless primitive, not the design system.

## Navigation

- Previous: [03_rendering_backends.md](./03_rendering_backends.md)
- Next: [05_wren_migration.md](./05_wren_migration.md): WRENStack to native migration path
