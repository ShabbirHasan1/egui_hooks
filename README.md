# egui_hooks

[![GitHub](https://img.shields.io/badge/GitHub-ryo33/egui__hooks-222222)](https://github.com/ryo33/egui_hooks)
![MIT/Apache 2.0](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)
[![Crates.io](https://img.shields.io/crates/v/egui_hooks)](https://crates.io/crates/egui_hooks)
[![docs.rs](https://img.shields.io/docsrs/egui_hooks)](https://docs.rs/egui_hooks)

React Hooks like API for enhancing the ergonomics of `egui::Memory`

| egui version | egui_hooks version |
| ------------ | ------------------ |
| egui@0.24.0  | 0.1.2              |
| egui@0.25.0  | 0.2.0              |
| egui@0.26.0  | 0.3.0              |
| egui@0.27.0  | 0.4.0              |
| egui@0.28.0  | 0.5.0              |
| egui@0.29.0  | 0.6.0              |
| egui@0.30.0  | 0.7.0              |
| egui@0.31.0  | 0.8.0              |
| egui@0.32.0  | 0.9.0              |

## Overview

This crate provids React Hooks like API for egui.

Though this started as a toy project, eventually I found that it's definitely
useful and that could be a core building block for widget development, and also
for application development.

Look at the [examples](https://github.com/ryo33/egui_hooks/tree/main/examples), run
`cargo run --example <example-name>` to see the demo.

## Features

- No resource leak: Opposite to using `egui::Memory` directly, the states are
  automatically freed from the HashMap when the widget will be no longer
  displayed. This is based on `TwoFrameMap` (2f kv) defined in this crate.
- No locking nor callback: You can manage states without `ui.data(|| { ... })`.
  This is because hooks encapsulate the underlying RwLock operation.
- Dependency tracking: Hooks has dependencies like
  `use_state(|| user_id.clone(), user_id)` or
  `use_effect(|| log(input), input)`, so you can precisely track the
  dependencies without manually writing `if` statements on state change.
- Composable: Hooks are composable, you can call existing hooks in your custom
  hooks.
- Familiar API: Hooks are designed to be similar to React Hooks API, so you can
  easily learn how to use them. Managing UI states in UI side is the key in
  recent UI development scene, but built-in `egui::Memory` is relatively
  low-level API and not friendly for applcation development, and egui_hooks
  provides a higher level API but with more precise control.

## How it works

If you use `use_state(|| 0usize, dep).into_var()` in a widget, the following
things happen:

1. On the first call of `use_state`, it creates a `Arc<ArcSwap<usize>>` in the
   `egui::Memory` with the default value.
2. If the `dep` is changed since the last frame, it stores the default value to
   the existing `ArcSwap`.
3. Returns a `Var<usize>` to the caller.
4. Caller can `Deref` or `DerefMut` the `Var` in their widget code.
5. When the `Var` is dropped, it stores the updated value to the `ArcSwap`.
6. Wenn the widget is no longer displayed, the `ArcSwap` is removed from the
   `egui::Memory`.

This is the typical lifecycle of a hook in egui_hooks.

Also, there is a persistent version of `use_state` called `use_persisted_state`.
It does the similar thing, but it stores the copy of the state to the
`egui::Memory` with _persisted_ methods. The persisted state is freed when the
widget is no longer displayed as like the not-persisted one. **You need `persistence` feature to use persisted hooks.**

## Intended use cases

- `use_state` for states in a specific widget (e.g. animation state, scroll
  position)
- `use_state` with `into_var()` to feed a variable in-place to `Window::open` or
  `TextEdit::singleline`
- `use_memo`, `use_cache` for caching expensive calculation
- `use_effect`, `use_future` for side effects (e.g. logging, network request)
- `use_global` for global settings (e.g. theme, locale)
- `use_kv` for sharing states between widgets (e.g. getting a position of a
  specific widget)
- `use_ephemeral_kv` for storing events in the current frame (e.g. providing
  custom response on a custom widget)
- `use_previous_measurement` for using the previous frame result for layouting
- `use_measurement` for calculating and memoizing the size of a widget for
  layouting

## Status

- [x] `use_memo`
- [x] `use_effect`
- [ ] `use_effect_with_cleanup`
- [x] `use_state`, `use_persisted_state`
- [x] `state.into_var()` to use state as a variable
- [x] `use_kv`, `use_persisted_kv`
- [x] `use_2f_kv`, `use_persisted_2f_kv`
- [x] `use_ephemeral_kv`
- [x] `use_global`, `use_persisted_global`, and `use_ephemeral_global`
- [ ] `use_cache` (a thin wrapper of caches in `egui::Memory`)
- [ ] `use_previous_measurement`
- [ ] `use_measurement` (calculate the size of the widget without fear of the
      [2^N problem](https://github.com/emilk/egui/issues/606#issuecomment-899065242).
- [ ] `use_future` (needs `tokio` feature)
- [ ] `use_throttle` and `use_debounce`
- [ ] [`use_drag_origin`](https://github.com/ryo33/egui_hooks/issues/9)
- [ ] `use_two_path` (it's joke, but really want to implement this)

## Usage

### use_state

```rust
// You can reset the initial state by changing the dependency part.
let count = ui.use_state(|| 0usize, ());
ui.label(format!("Count: {}", count));
if ui.button("Increment").clicked() {
    count.set_next(*count + 1);
}
```

### use_persisted_state

```rust
let count = ui.use_persisted_state(|| 0usize, ());
ui.label(format!("Count: {}", count));
if ui.button("Increment").clicked() {
    count.set_next(*count + 1);
}
```

### use_memo

```rust
let count = ui.use_state(|| 0usize, ());
let memo = ui.use_memo(
    || {
        println!("Calculating memoized value");
        count.pow(2)
    },
    count.clone(),
);
ui.label(format!("Memo: {}", memo));
if ui.button("Increment").clicked() {
    count.set_next(*count + 1);
}
```

### use_hook_as

In the following example, the `use_hook_as` is almost equivalent to call `ui.use_state(|| true, ())` in the show closure but allows you to pass the `open` state to the `Window::open` method.

```rust
for window in ["window1", "window2", "window3"] {
    let mut open = ui
        .use_hook_as(egui::Id::new(window), StateHook::new(|| true), ())
        .into_var();
    egui::Window::new(window)
        .open(&mut open)
        .show(ui.ctx(), |ui| {
            // ...
        });
}
```

### use_effect

```rust
let count = ui.use_state(|| 0usize, ());
ui.use_effect(|| println!("Count changed to {}", *count), count.clone());
```

### use_cleanup

```rust
ui.use_cleanup(|| println!("This widget is no longer displayed"), ());
```

## Custom Hooks

You can create your own hooks by the two ways.

1. Creating a function for a hook

This is the simplest and recommended way to create a custom hook.

```rust
fn use_search(ui: &mut Ui, client: &Client) -> Option<SearchResults> {
    let text = ui.use_state(|| String::default(), ()).into_var();
    ui.text_edit_singleline(&mut *text);
    ui.use_effect(|| client.search(&*text), text.clone())
}
```

Or, you can make it to an extension to `egui::Ui` to make it callable as `ui.use_search(client)`.

```rust
trait UseSearchExt {
    fn use_search(&mut self, client: &Client) -> Option<SearchResults>;
}

impl UseSearchExt for egui::Ui {
    fn use_search(&mut self, client: &Client) -> Option<SearchResults> {
        // same as the previous example
    }
}
```

2. Implement `Hook` trait

All built-in hooks are implemented in this way. This allow you to create a hook
with full control, but it is a bit verbose.

```rust
impl<D> Hook<D> for MyHook {
    type Backend = ()
    type Output = usize;

    fn init(
        &mut self,
        _index: usize,
        _deps: &D,
        _backend: Option<Self::Backend>,
        _ui: &mut egui::Ui,
    ) -> Self::Backend {
    }

    fn hook(self, backend: &mut Self::Backend, ui: &mut egui::Ui) -> Self::Output {
        let count = ui.use_state(0usize, ());
        ui.label(format!("Count: {}", count));
        if ui.button("Increment").clicked() {
            count.set_next(*count + 1);
        }
        count
    }
}
```
