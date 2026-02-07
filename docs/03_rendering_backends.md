# 03 - Rendering Backends: Wayland, Quartz, D3D12

## Design Goal

Fidelity.UI components describe *what* to render. The rendering backend determines *how* pixels reach the screen. Backends are defined in Fidelity.Platform, not in Fidelity.UI, maintaining the Fidelity principle that platform details do not leak into application code.

## Rendering Model

Fidelity.UI uses a **region-based rendering model**. The widget tree is divided into rectangular render regions. Each region:

1. Subscribes to the signals that affect its visual output
2. Maintains a pixel buffer (or GPU texture) for its content
3. Repaints only when one of its subscribed signals changes
4. Submits its buffer to the compositor for display

This is fundamentally different from both immediate-mode rendering (repaint everything every frame) and retained-mode widget toolkits (platform widgets with their own rendering). It's closest to how modern compositor-based systems actually work: surfaces with content, composed by the window manager.

## Platform Backends

### Linux: Wayland

Wayland is the native display protocol on modern Linux. The application communicates directly with the compositor (Hyprland, Sway, GNOME/Mutter, KDE/KWin).

**Surface model**: Each top-level window is a `wl_surface`. Subsurfaces can be used for independent render regions (e.g., a video player or GPU-rendered component within a UI).

**Buffer management**: Shared memory buffers via `wl_shm` (CPU rendering) or DMA-BUF for GPU-rendered content. Fidelity.Platform provides `memfd_create` + `ftruncate` + `mmap` for shared memory allocation.

**Input**: Wayland delivers input events (pointer, keyboard, touch) through `wl_seat`. Events flow into the signal-actor reactor as messages.

**Key protocols**:
- `xdg-shell`: Window management (toplevel, popup)
- `wl_shm`: Shared memory pixel buffers
- `wp-fractional-scale`: HiDPI support
- `wp-viewporter`: Efficient scaling
- `xdg-decoration`: Server-side window decorations

### macOS: Quartz / Metal

macOS uses Quartz (Core Graphics) for 2D rendering and Metal for GPU-accelerated rendering.

**Surface model**: `NSWindow` + `NSView` hierarchy. Each render region can be a `CALayer` backed by a Metal texture for GPU rendering.

**Buffer management**: `CGBitmapContext` for CPU rendering, `MTLTexture` for GPU. Core Animation composites layers automatically.

**Input**: `NSEvent` / `NSResponder` chain delivers input events.

**Key APIs**:
- Core Graphics: 2D vector/raster rendering
- Metal: GPU-accelerated rendering
- Core Animation: Layer compositing
- AppKit: Window and event management

### Windows: D3D12 / Win32

Windows uses D3D12 (or D3D11) for GPU rendering and GDI/Direct2D for software fallback.

**Surface model**: `HWND` windows with `IDXGISwapChain` for GPU-rendered content.

**Buffer management**: `ID3D12Resource` textures for GPU rendering, `ID2D1RenderTarget` for Direct2D.

**Input**: Win32 message loop (`WM_MOUSEMOVE`, `WM_KEYDOWN`, etc.) dispatched into the signal-actor reactor.

**Key APIs**:
- D3D12: GPU-accelerated rendering
- Direct2D: Hardware-accelerated 2D
- DirectWrite: Text rendering
- Win32: Window management, message loop

## Rendering Pipeline

```
Signal changes
    ↓
Signal-Actor reactor notifies subscribed render regions
    ↓
Dirty regions re-execute their render functions
    ↓
Render functions produce drawing commands (rects, text, images, paths)
    ↓
Drawing commands execute against platform rendering API
    ↓
Platform buffer submitted to compositor
    ↓
Compositor displays on screen
```

### CPU Rendering Path (Stage 1)

For initial implementation and fallback:

1. Render regions paint into shared memory pixel buffers
2. Buffers are submitted to Wayland via `wl_shm` / to Quartz via `CGBitmapContext` / to Win32 via GDI
3. The compositor composites and displays

This is functional, portable, and sufficient for most UI content.

### GPU Rendering Path (Stage 2+)

For performance-critical content (large lists, animations, visualizations):

1. Render regions paint into GPU textures
2. Textures are submitted via DMA-BUF (Wayland) / Metal (macOS) / D3D12 (Windows)
3. The compositor zero-copies the GPU texture to the display

## Text Rendering

Text rendering is a platform-specific concern handled by Fidelity.Platform:

| Platform | Text Engine | Font Loading |
|----------|-------------|-------------|
| Linux | FreeType + HarfBuzz | fontconfig |
| macOS | Core Text | Font Manager |
| Windows | DirectWrite | System fonts |

Fidelity.UI's text components specify font family, size, weight, and color. The platform backend resolves these to native font handles and renders glyphs.

## HiDPI

All layout is in logical pixels. The rendering backend applies the platform's scale factor:

- **Wayland**: `wp-fractional-scale` provides the scale factor
- **macOS**: `NSScreen.backingScaleFactor`
- **Windows**: Per-monitor DPI via `GetDpiForWindow`

Render buffers are allocated at physical pixel dimensions. Layout math is in logical pixels. The framework handles the conversion.

## Navigation

- Previous: [02_component_model.md](./02_component_model.md)
- Next: [04_design_system.md](./04_design_system.md): Tokens, themes, headless primitives
