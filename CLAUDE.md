# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**bl_ui_widgets** is a Blender addon (v0.6.4.2) that provides a framework for drawing interactive UI widgets (buttons, sliders, checkboxes, text inputs, draggable panels) directly in the 3D viewport using Blender's GPU module. Author: Jayanam. License: GNU GPLv3.

There is no CLI, build step, or test runner — this is a pure Python Blender addon. Testing requires loading the addon in Blender itself.

**Install/reload:** Copy the `bl_ui_widgets/` folder to Blender's addons directory, enable "BL UI Widgets" in Preferences → Add-ons, then trigger via **Shift+Ctrl+F** in the 3D View.

## Architecture

### Widget Hierarchy

All widgets inherit from `BL_UI_Widget` (base class in [bl_ui_widget.py](bl_ui_widget.py)):

```
BL_UI_Widget
├── BL_UI_Button       — clickable button (text + optional image/icon)
├── BL_UI_Checkbox     — boolean toggle with label
├── BL_UI_Label        — read-only text display (always returns False from is_in_rect)
├── BL_UI_Slider       — horizontal range slider with value callback
├── BL_UI_Textbox      — text/numeric input with caret, keyboard handling
├── BL_UI_Up_Down      — increment/decrement spinner
└── BL_UI_Drag_Panel   — container that holds child widgets and is draggable
```

### Operator Hierarchy

- [bl_ui_draw_op.py](bl_ui_draw_op.py) — `BL_UI_OT_draw_operator`: Abstract base operator. Manages draw handler registration, modal event loop, and event routing to all widgets. Subclass this to build a custom viewport UI.
- [drag_panel_op.py](drag_panel_op.py) — `DP_OT_draw_operator`: Concrete demo operator (registered as `object.dp_ot_draw_operator`, bound to Shift+Ctrl+F). Shows all widget types in action.

### Rendering Pipeline

1. `invoke()` registers a `POST_PIXEL` draw handler and a timer.
2. Each frame, `draw_callback_px()` calls `widget.draw()` for every widget.
3. Widgets use `gpu.shader.from_builtin('2D_UNIFORM_COLOR')` for solid shapes and `'2D_IMAGE'` for textures; text via `blf`.
4. Vertex geometry is pre-calculated in `update(x, y)` and cached as GPU batches — only recalculated when position/size changes.
5. Coordinate system: `(0,0)` = bottom-left of area; widget Y is flipped (`area_height - y_screen`).

### Event Flow

```
User input → Blender modal event → handle_widget_events(event)
  → for each widget: widget.handle_event(event)
    → dispatches to mouse_down / mouse_up / mouse_move / mouse_enter / mouse_exit / text_input
    → widget updates state → registered callback fires
```

The modal returns `{'PASS_THROUGH'}` for most events so Blender's normal shortcuts still work; ESC finishes the operator and unregisters handlers.

### Callback Pattern

Widgets accept function references rather than a framework approach:

```python
button.set_mouse_down(my_callback)
slider.set_value_change(my_callback)
checkbox.set_state_changed(my_callback)
textbox.set_text_changed(my_callback)
```

### Adding a New Widget

1. Subclass `BL_UI_Widget`.
2. Override `draw()` to render using GPU shaders.
3. Override `update(x, y)` to recalculate vertex geometry.
4. Override `handle_event(event)` (or the individual `mouse_*` / `text_input` methods) for interactivity.
5. Add an instance to the widgets list in your operator subclass.

### Key Properties (all widgets)

- `x`, `y`, `width`, `height` — position and size
- `bg_color` — RGBA background color tuple
- `visible` — bool (added v0.6.4.0); widgets with `visible=False` skip draw and event handling
- `tag` — arbitrary identifier string

## Blender API Notes

- Uses `gpu`, `gpu_extras.batch`, `bgl`, and `blf` — no external dependencies.
- `bgl` (OpenGL bindings) may be deprecated in future Blender versions; prefer `gpu` module calls where possible.
- Blender 2.80+ is required; tested through Blender 5.x.
- `__init__.py` `bl_info` must use literal values only — Blender parses it via AST without executing the module.
