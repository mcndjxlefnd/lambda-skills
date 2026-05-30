# DearPyGui 2.3.1 Pitfalls for the HAGA Dashboard

Pitfalls discovered while building and testing `haga_tools/dashboard/`.
Load when working on dashboard code.

## get_dearpygui_version() segfaults without create_context()

`dpg.get_dearpygui_version()` calls into the C extension and segfaults
(SIGSEGV) if no DearPyGui context exists. No Python traceback — just a
hard crash. `dpg.create_context()` must be called first.

```python
import dearpygui.dearpygui as dpg
dpg.create_context()
print(dpg.get_dearpygui_version())  # 2.3.1 — safe now
```

This also applies to `get_major_version()`, `get_minor_version()`, and
any other C-extension function that assumes an active context. In
practice, the dashboard's main.py calls `create_context()` early, so
this only bites during ad-hoc testing or one-liner imports.

## Dashboard launch smoke test

The dashboard is a native GUI app — pytest can't drive it. To verify
it initializes without crashing:

```bash
cd ~/haga_master && PYTHONPATH=src timeout 4 .venv/bin/python -m haga_tools.dashboard.main
```

Exit 124 = timeout killed it (it stayed alive — no crash).
Exit 1 + traceback = something broke during init.
Any segfault = C-extension crash (likely missing context, see above).

This is a quick sanity check, not a functional test. For functional
verification, launch it interactively on a machine with a display.

## Line series coloring requires themes

`dpg.add_line_series()` does not expose a `color` parameter directly.
Colors must be applied via a theme with `dpg.add_theme_color()` targeting
`mvLineSeries_LineColor`. Create one theme per color, then bind with
`dpg.bind_item_theme(series_tag, theme)`.

Dimmed colors (for distinguishing best-vs-mean) require a second theme
per color with reduced intensity.

Pattern from `main.py`:
```python
_COLOR_THEMES = []  # one theme per _COLORS entry
_DIM_THEMES = []    # dimmed variant

for color in _COLORS:
    with dpg.theme() as t:
        dpg.add_theme_color(dpg.mvLineSeries_LineColor, color, ..., tag=t)
    _COLOR_THEMES.append(t)
```

## dpg.subplots() parameter is `columns`, not `cols`

```python
# WRONG: dpg.subplots(rows=3, cols=1)
# RIGHT:
dpg.subplots(rows=3, columns=1)
```

Pyright may flag the return type as opaque — `# type: ignore` is
acceptable here.

## No built-in timer — use frame callback

DearPyGui has no `set_timer()` equivalent. Use `dpg.set_frame_callback()`
with a `get_delta_time()` accumulator to achieve periodic polling inside
the render loop.

Pattern from `main.py`:
```python
def _poll_callback():
    dt = dpg.get_delta_time()
    _STATE["watch_accumulator"] += dt
    if _STATE["watch_accumulator"] >= 1.0:
        _STATE["watch_accumulator"] = 0.0
        _refresh_live_data()

dpg.set_frame_callback(1, _poll_callback)  # frame 1 = after primary window
```

## Table column headers lack click callbacks

`dpg.add_table_column()` has no `callback` parameter. To make table
columns sortable, build a custom header row with `dpg.add_selectable()`
items above the table, wired to sort-and-rebuild callbacks.

Pattern from `main.py` Phase 7: comparison table uses a selectable
header row that triggers `_rebuild_comparison_table()` with the
chosen sort key.

## Plot style vars require `category=dpg.mvThemeCat_Plots`

`dpg.add_theme_style(dpg.mvPlotStyleVar_LineWeight, 3.0)` silently does
nothing when applied to a `mvLineSeries` theme component without the
`category` parameter. The style enum value is 0 (same as default), so
DPG applies it without error but the lines render at 1px.

**Fix:** Always pass `category=dpg.mvThemeCat_Plots` for plot styles:

```python
with dpg.theme_component(dpg.mvLineSeries):
    dpg.add_theme_color(dpg.mvPlotCol_Line, color, category=dpg.mvThemeCat_Plots)
    dpg.add_theme_style(dpg.mvPlotStyleVar_LineWeight, 3.0, category=dpg.mvThemeCat_Plots)
```

`add_theme_color` with `mvPlotCol_Line` works without `category` because
DPG auto-detects plot colors from the constant name, but style vars don't
get the same treatment. Always include `category` for both to be safe.

Binding the theme to the PLOT (not the series) does NOT work for
LineWeight — it must be on the series-level theme component. The
`category` parameter is the missing piece, not the binding target.

## Screenshot verification requires window focus

`import -window root` captures whichever window has focus, which may be
the terminal rather than the DPG window. To reliably screenshot the
dashboard, minimize or defocus the terminal first, or use
`xdotool search --name "HAGA Dashboard"` to find the window ID
(note: xdotool may not be installed — check with `which xdotool`).

## Selectable toggle: read sender state, don't double-toggle

DPG selectables (`add_selectable`) auto-toggle their `value` on click.
If the callback also toggles an external state set, they cancel out.

**Wrong:** callback does `selected.discard(id)` / `selected.add(id)` independently.
**Right:** read `dpg.get_value(sender)` to sync with DPG's own toggle.

Lambda must forward `sender` as third positional arg:
```python
lambda s, a, u: callback(u, extra_arg, s)
#                  ^user_data       ^sender
```

Callback reads DPG state:
```python
def _on_click(item_id, sender):
    if dpg.get_value(sender):
        _STATE["selected"].add(item_id)
    else:
        _STATE["selected"].discard(item_id)
```

When rebuilding selectables (e.g. process tree), pass the current selection
set and set `default_value=item["id"] in selected` to preserve visual highlight
state across rebuilds.
