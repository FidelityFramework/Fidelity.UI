# 02 - Component Model: Composition DSL and Phantom Types

## Design Goal

Provide a component composition API that:
1. Feels natural to F# developers (computation expressions, pipe operators)
2. Feels familiar to WRENStack/Partas.Solid developers (signal-reactive, components run once)
3. Provides stronger type safety than web frameworks (phantom types, compile-time widget validation)
4. Compiles to zero-overhead native code via Firefly

## Components Run Once

This is the foundational principle borrowed from SolidJS and diverging from React/Fabulous:

```fsharp
// This function executes ONCE when the component mounts.
// It establishes signal subscriptions. It does NOT re-execute on state changes.
[<Component>]
let Counter () =
    let count, setCount = createSignal 0

    // The Stack and its children are created once.
    // Signal reads (count()) establish fine-grained subscriptions.
    Stack() {
        Label($"Count: {count()}")          // Re-renders label text when count changes
        Button("Increment", fun () ->
            setCount (fun c -> c + 1))      // Updates signal, triggers subscriptions
    }
```

The component body is a setup function, not a render function. It runs once, wires up the reactive graph, and returns a widget tree with embedded signal subscriptions.

## Phantom Type Markers

Inspired by Fabulous's `WidgetBuilder<'msg, 'marker>` pattern but adapted for the signal model.

Widget types carry phantom markers that constrain which modifiers are valid:

```fsharp
// Marker hierarchy
type IWidget = interface end
type ITextWidget = inherit IWidget
type IContainerWidget = inherit IWidget
type IInputWidget = inherit IWidget

// Label has ITextWidget marker
// .fontSize() is only available on ITextWidget descendants
// Calling .fontSize() on a Stack is a compile error
```

This catches invalid property application at compile time rather than runtime:

```fsharp
Label("Hello")
    .fontSize(16)        // Compiles: Label is ITextWidget
    .color(theme.primary) // Compiles: color is on IWidget

Stack() {
    Label("Hello")
}
    .gap(8)              // Compiles: gap is on IContainerWidget
    .fontSize(16)        // COMPILE ERROR: Stack is not ITextWidget
```

## Computation Expression Builders

### Container Builder

Containers use F# computation expressions for child composition:

```fsharp
Stack(direction = Vertical, gap = 8) {
    Label("First item")
    Label("Second item")
    for item in items() ->
        Label(item.Name)
    if showExtra() then
        Label("Extra item")
}
```

The CE builder supports `Yield`, `YieldFrom`, `For`, `Combine`, `Zero`, and `Delay` - enabling natural F# control flow inside widget trees.

### Component Builder

Components with local state use `let!` for signal bindings:

```fsharp
[<Component>]
let TodoList () =
    let todos, setTodos = createSignal []
    let filter, setFilter = createSignal "all"

    let filtered = createMemo (fun () ->
        match filter() with
        | "active" -> todos() |> List.filter (fun t -> not t.Done)
        | "done" -> todos() |> List.filter (fun t -> t.Done)
        | _ -> todos()
    )

    Stack() {
        FilterBar(filter(), setFilter)
        For(filtered) (fun todo _ ->
            TodoItem(todo, setTodos)
        )
    }
```

### Control Flow Components

Reactive control flow that updates surgically:

```fsharp
// Show: conditional rendering
Show(when' = isLoggedIn) {
    UserProfile()
} |> withFallback (LoginForm())

// For: keyed list rendering (each item tracked by identity)
For(items, key = fun item -> item.Id) (fun item index ->
    ItemRow(item, index)
)

// Switch/Match: multi-branch conditional
Switch() {
    Match(when' = isLoading()) { Spinner() }
    Match(when' = isError()) { ErrorMessage() }
    Match(when' = true) { Content() }
}
```

These control flow components are reactive boundaries. When `isLoggedIn` changes, only the `Show` region updates - the rest of the tree is untouched.

## Props and Events

### Props Flow Down

```fsharp
[<Component>]
let UserCard (name: Accessor<string>) (avatar: Accessor<string>) =
    Card() {
        Image(src = avatar())
        Label(name())
    }

// Usage: props are signals, enabling fine-grained updates
UserCard (userName) (userAvatar)
```

Props are signal accessors (`unit -> 'T`). Reading a prop inside the component body establishes a subscription - when the parent's signal changes, the child's subscribed regions update.

### Events Flow Up

```fsharp
[<Component>]
let Counter (onCountChanged: int -> unit) =
    let count, setCount = createSignal 0

    createEffect (fun () -> onCountChanged (count()))

    Button("Increment", fun () -> setCount (fun c -> c + 1))
```

Event callbacks are plain functions. No message types, no discriminated union dispatch. The parent passes a callback; the child calls it.

### Two-Way Binding

For controlled inputs, signals can be shared:

```fsharp
[<Component>]
let SearchBar (query: Signal<string>) =
    let value, setValue = query
    Input(value = value(), onInput = fun e -> setValue e.Value)
```

## Modifiers

Visual properties are applied via fluent modifier methods:

```fsharp
Label("Hello")
    .fontSize(16)
    .color(theme.primary)
    .padding(8, 12)
    .cornerRadius(4)
```

Modifiers return a new widget description (struct copy, no allocation). They are constrained by phantom type markers to prevent invalid combinations at compile time.

### Layout Modifiers

```fsharp
widget
    .width(200)
    .height(Fill)        // Fill available space
    .minWidth(100)
    .maxWidth(400)
    .padding(8)
    .margin(top = 16)
    .align(Center)
```

### Style Modifiers

```fsharp
widget
    .background(theme.surface)
    .border(1, theme.outline)
    .cornerRadius(8)
    .shadow(elevation = 2)
    .opacity(0.9)
```

### Interaction Modifiers

```fsharp
widget
    .onClick(handler)
    .onHover(handler)
    .onFocus(handler)
    .cursor(Pointer)
    .focusable(true)
    .accessibilityLabel("Close dialog")
```

## Navigation

- Previous: [01_signal_system.md](./01_signal_system.md)
- Next: [03_rendering_backends.md](./03_rendering_backends.md): Platform rendering
