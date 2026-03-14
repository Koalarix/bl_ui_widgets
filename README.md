# bl_ui_widgets

A Blender addon framework for drawing interactive UI widgets directly in the **3D Viewport** using Blender's GPU module. Provides buttons, sliders, checkboxes, text inputs, and a draggable panel container — all rendered as pixel-perfect overlay elements.

**Author:** Jayanam | **License:** GNU GPLv3

---

## Blender Version Compatibility

> **This addon targets Blender 2.80–3.x and is NOT compatible with Blender 4.0 or later (including 5.0.1+).**

Several APIs this addon relies on were removed or renamed in Blender 4.0:

| API used | Status in Blender 4.0+ |
|---|---|
| `import bgl` | **Removed** — crashes on addon load |
| `bgl.glEnable / glDisable / glBindTexture` | **Gone** with the `bgl` module |
| `gpu.shader.from_builtin('2D_UNIFORM_COLOR')` | **Renamed** to `'UNIFORM_COLOR'` |
| `gpu.shader.from_builtin('2D_IMAGE')` | **Renamed** to `'IMAGE'`; texture binding API changed |
| `blf.size(font_id, size, dpi)` | **Changed** — DPI argument removed in Blender 4.0 |
| `image.gl_load()` | **Removed** — replaced by `gpu.texture.from_image()` |

To port this addon to Blender 4.x/5.x all `bgl` calls need replacing with `gpu` module equivalents, shader names need updating, `blf.size()` calls need the DPI argument dropped, and image loading needs rewriting around `gpu.texture.from_image()`.

---

## Installation (Blender 2.80–3.x)

1. Download or clone this repository.
2. In Blender: **Edit → Preferences → Add-ons → Install…**
3. Select the `bl_ui_widgets` folder.
4. Enable **"BL UI Widgets"** in the add-on list.

---

## Quick Start

Press **Shift+Ctrl+F** in the 3D Viewport to open the built-in demo panel. It demonstrates all widget types working together.

---

## Usage

### 1. Create a draw operator

Subclass `BL_UI_OT_draw_operator` from `bl_ui_draw_op.py`. Instantiate your widgets in `__init__`, then register them in `on_invoke`:

```python
from . bl_ui_button      import BL_UI_Button
from . bl_ui_label       import BL_UI_Label
from . bl_ui_drag_panel  import BL_UI_Drag_Panel
from . bl_ui_draw_op     import BL_UI_OT_draw_operator

class MY_OT_custom_panel(BL_UI_OT_draw_operator):
    bl_idname = "object.my_custom_panel"
    bl_label  = "My Custom Panel"
    bl_options = {'REGISTER'}

    def __init__(self):
        super().__init__()

        self.panel = BL_UI_Drag_Panel(100, 200, 280, 180)
        self.panel.bg_color = (0.15, 0.15, 0.15, 0.9)

        self.label = BL_UI_Label(20, 10, 200, 20)
        self.label.text = "Hello World"

        self.btn = BL_UI_Button(20, 40, 120, 30)
        self.btn.text = "Click Me"
        self.btn.set_mouse_down(self.on_click)

    def on_invoke(self, context, event):
        widgets_panel = [self.label, self.btn]
        self.init_widgets(context, [self.panel] + widgets_panel)
        self.panel.add_widgets(widgets_panel)
        self.panel.set_location(event.mouse_x,
                                context.area.height - event.mouse_y + 20)

    def on_click(self, widget):
        print("Button clicked!")
```

### 2. Register the operator and add a keybinding

```python
import bpy

classes = (MY_OT_custom_panel,)

def register():
    for cls in classes:
        bpy.utils.register_class(cls)

    wm = bpy.context.window_manager
    kc = wm.keyconfigs.addon
    if kc:
        km = kc.keymaps.new(name="3D View", space_type="VIEW_3D")
        km.keymap_items.new("object.my_custom_panel", "F", "PRESS",
                            shift=True, ctrl=True)

def unregister():
    for cls in classes:
        bpy.utils.unregister_class(cls)
```

### 3. Close the panel

Press **Esc** in the 3D Viewport to dismiss the active widget panel.

---

## Widget Reference

### Common properties (all widgets)

| Property | Type | Description |
|---|---|---|
| `x`, `y` | int | Position (top-left corner, relative to area) |
| `width`, `height` | int | Size in pixels |
| `bg_color` | (R,G,B,A) | Background color |
| `visible` | bool | Show/hide the widget (default `True`) |
| `tag` | any | Arbitrary identifier |

### BL_UI_Button

```python
btn = BL_UI_Button(x, y, width, height)
btn.text            = "Label"
btn.text_color      = (1, 1, 1, 1)
btn.text_size       = 16
btn.hover_bg_color  = (0.5, 0.5, 0.5, 1)
btn.select_bg_color = (0.7, 0.7, 0.7, 1)
btn.set_image("//img/icon.png")        # optional icon
btn.set_image_size((24, 24))
btn.set_image_position((4, 2))         # offset inside button
btn.set_mouse_down(callback)           # callback(widget)
```

### BL_UI_Checkbox

```python
chb = BL_UI_Checkbox(x, y, width, height)
chb.text       = "Enable feature"
chb.text_size  = 14
chb.text_color = (1, 1, 1, 1)
chb.is_checked = False
chb.set_state_changed(callback)        # callback(checkbox, state: bool)
```

### BL_UI_Slider

```python
sld = BL_UI_Slider(x, y, width, height)
sld.min          = 0.0
sld.max          = 1.0
sld.decimals     = 2
sld.show_min_max = True
sld.color        = (0.3, 0.6, 0.8, 1)
sld.hover_color  = (0.4, 0.7, 0.9, 1)
sld.set_value(0.5)
sld.set_value_change(callback)         # callback(slider, value: float)
```

### BL_UI_Up_Down (spinner)

```python
ud = BL_UI_Up_Down(x, y)              # width/height auto-sized
ud.min      = 1.0
ud.max      = 10.0
ud.decimals = 0
ud.color        = (0.3, 0.6, 0.8, 1)
ud.hover_color  = (0.4, 0.7, 0.9, 1)
ud.set_value(5.0)
ud.set_value_change(callback)          # callback(up_down, value: float)
```

### BL_UI_Textbox

```python
tb = BL_UI_Textbox(x, y, width, height)
tb.text       = "default text"
tb.text_size  = 14
tb.is_numeric = False                  # set True to restrict to numbers
tb.set_text_changed(callback)          # callback(textbox, text: str)
```

### BL_UI_Label

```python
lbl = BL_UI_Label(x, y, width, height)
lbl.text       = "My Label"
lbl.text_size  = 14
lbl.text_color = (1, 1, 1, 1)
# Labels are non-interactive — they never capture mouse events
```

### BL_UI_Drag_Panel

```python
panel = BL_UI_Drag_Panel(x, y, width, height)
panel.bg_color = (0.2, 0.2, 0.2, 0.9)
panel.add_widgets([widget1, widget2, ...])  # child widgets use panel-relative coords
panel.set_location(x, y)                   # reposition after init
```

Child widget coordinates are relative to the panel's top-left corner. When the user drags the panel, all children move with it automatically.

---

## Demo

The included `drag_panel_op.py` is a fully working example showing all widget types wired to Blender object operations (scale, rotate, hide). It is registered under **Shift+Ctrl+F** and is the entry point for the addon.
