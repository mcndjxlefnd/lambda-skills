---
name: gui-edit
description: HAGA dashboard domain lens for viz-verify. DearPyGui constraints,
  code locations, widget search patterns.
category: lambda-skills
version: 0.1.0
---

NARRATOR
    "Gui-edit, the Director wants changes. What does the codebase say?"

# Gui-Edit

I know the HAGA dashboard and how to edit DearPyGui. I tell viz-verify
_where_ to look and _what_ constraints apply. I do not build — I hand the
domain lens to the gatekeeper.

Dashboard code: `haga_tools/dashboard/` (main.py, data.py, process_tree.py).
State management pitfalls: `references/state-management-pitfalls.md`.
Stale-show pitfall (set-iterated render loop): `references/stale-show-pitfall.md`.

## Lens

**Scope** = visual problems mapped to dashboard code. **Prefer
`mcp_codegraph_codegraph_context`** for codebase exploration — one call
returns entry points, related symbols, and key bodies for the
relevant area. Use `codegraph_trace from→to` for flow questions
(e.g. click handler → state mutation → chart update). Fall back to
`search_files(pattern="<visible text>", path="haga_tools/dashboard")`
only for finding widget creation sites by literal text, and
`read_file` for layout context (parent group, spacers, siblings).

**Implementation** = edit the GUI code under `haga_tools/dashboard/`. DPG
constraints:
- Line series coloring: `dpg.bind_item_theme(tag, theme)` — NOT `configure_item(color=)`
- Line thickness: `dpg.add_theme_style(dpg.mvPlotStyleVar_LineWeight, 3.0, category=dpg.mvThemeCat_Plots)` — the `category` param is required or the style is silently ignored (renders at 1px). Bind to the series, not the plot.
- Subplots: parameter is `columns` not `cols`
- Timers: frame callback, no `set_timeout`
- Clipped text: `dpg.add_spacer(height=N)` before the element
- Visibility: `dpg.configure_item(tag, show=True/False)`
- Stale values: `dpg.set_value(tag, new_text)`
- Axes auto-scale: `dpg.fit_axis_data(axis_tag)` — NOT `fit_axis`
- `fit_axis_data` includes 0 in the range. For data in [0.2, 0.8] it stays at
  [0, 1]. Use `_fit_axis_tight(axis_tag, pad_frac=0.08)` when you need tight
  bounds that don't force 0. Pattern: iterate visible children via
  `get_item_children(axis_tag, slot=1)`, read `get_value(item)` for ys, compute
  min/max, call `set_axis_limits(axis_tag, lo, hi)`. Use for sigma rate charts;
  keep `fit_axis_data` for convergence (fitness can be negative).
- **Selectable toggle pitfall**: DPG selectables (`add_selectable`) auto-toggle their `value` on click. If your callback also toggles an external state set, they cancel out — the selectable toggles itself, then your code toggles the set, net zero. FIX: read `dpg.get_value(sender)` in the callback to sync with DPG's state instead of maintaining a parallel toggle. Lambda must forward `sender`: `lambda s, a, u: callback(u, ..., s)`. The callback signature is `(sender, app_data, user_data)`.
- **Tree rebuild + selection state**: when rebuilding a tree of selectables (e.g. process tree), pass the current selection set and set `default_value=p["id"] in selected` on each selectable. Otherwise the visual highlight state is lost on rebuild, even though the internal set still holds the selection.
- Visibility check: `dpg.is_item_visible()` checks item STATE which isn't
  populated until viewport renders. Fails with KeyError in headless/before-show
  mode. Use `dpg.get_item_configuration(item).get("show", True)` instead.
- Subplot axes: each axis inside `dpg.subplots()` needs its own fit call.
  They do NOT auto-fit from the parent or from sibling axes. Pattern: iterate
  axis tags via a dict + string replacement (e.g. `_SIGMA_Y_TAGS` + `_y_` → `_x_`).
- Watch/poll callbacks: after new data arrives, call fit on all relevant axes,
  not just the convergence chart axes.
- **State management pitfall**: DPG "clear" functions often reset too much state.
  A `_clear_chart()` that resets `checked_runs` or `show_comparison` breaks
  multi-selection flows — the callback chain is: user checks run →
  `_on_run_check` adds to `checked_runs` → `_rebuild_process_tree` calls
  `_clear_chart` → state wiped → `_update_chart` sees empty selection.
  RULE: `_clear_chart` must only clear **display series** (line series values,
  visibility). It must NOT touch user selections (`checked_runs`, processes)
  or UI toggles (`show_comparison`). Full state reset belongs in
  `_load_file(replace=True)` only — that's the "new file" clean-slate path.
  When you see a cleanup function that resets `checked_runs` or selection
  state, that's the bug.
- **Stale-show pitfall (set-iterated render loop)**: A render loop that
  iterates a *subset* of items to update/create line series (e.g. `for pid
  in selected_processes`) will NEVER hide line series for items that just
  left that subset. The loop simply doesn't visit them. Classic symptom:
  visual state of the selectable flips off correctly (DPG handles that
  side), but the corresponding chart line stays drawn — until the subset
  becomes empty, at which point a different branch (e.g. the run-level
  `else` in `_update_chart`) does a full hide sweep, and all the stale
  lines disappear at once. So "deselect one" looks broken, "deselect the
  last" looks like a sudden fix. DIAGNOSIS: grep the render function for
  `for x in <subset>` and check whether anything hides items in the
  *complement*. The existing "Hide stale from unchecked runs" loop is
  often present but only covers one half (unchecked run, any proc) — the
  other half (checked run, deselected proc) is the missing case.
  FIX: unify the hide into one sweep with the condition for *not*
  hiding = `(item in active_set) AND (item in selected_subset)`. Anything
  failing that test is stale and gets `configure_item(tag, show=False)`.
  In `_update_chart`, the three "Hide stale" loops (convergence, variance,
  sigma) become a single shape: hide `(run, proc)` unless `run in checked
  AND proc in selected`. The `else` branch's "Hide all process-level
  series" loop is the equivalent full-domain sweep when selected is
  empty — that one is correct as-is.
- **Color-index consistency pitfall**: when a widget's color is bound by
  *index* into a palette list, every other widget that should share that
  color must derive its index from the **same** ordering. The classic
  symptom is "tree highlight color doesn't match the chart line color for
  the same item." Cause: tree themes indexed by `sorted(all_processes)`,
  chart themes indexed by `sorted(selected)`. These only agree when the
  selected items are the first N in the full sort. FIX: build a
  stable `item_id → color_index` map at source-of-truth time (e.g. when
  the tree is built), and look up that map from every other place that
  needs a color. Don't re-derive indices inline — different lists,
  different sorts, different `enumerate` counts, different bugs.

**DPG legend pitfall (2.3.1):** `dpg.add_plot_legend(parent=..., show=False)` sets the
item's internal `show` config to `False`, but ImPlot still renders the legend
at the C++ level. The `show` flag on legends is silently ignored. To hide a
legend, **remove the `add_plot_legend()` call entirely**. The plot's right-click
context menu still offers "Show Legend" if a user needs it temporarily. If you
need a programmatic toggle, conditionally add/delete the legend item at runtime
via `dpg.delete_item()`/`dpg.add_plot_legend()`.

**DPG font management:** The default DPG font (ProggyClean) is a bitmap font.
Scaled via `dpg.set_global_font_scale(1.3)`, it interpolates pixels and
produces soft/blurry text. The fix is to load a proper TrueType font:
```python
dpg.create_context()
with dpg.font_registry():
    default_font = dpg.add_font(
        "/usr/share/fonts/TTF/DejaVuSans.ttf", 17)
dpg.bind_font(default_font)
```
Available on Manjaro: DejaVu Sans, Liberation Sans, Noto Sans. DejaVu Sans
at 17px roughly matches 1.3x bitmap scale with crisp rendering. Getting
font size right may interact with fixed-width panels (left panel at 480px
may clip process tree text) and axis tick labels (small floats with many
decimals overlap at larger sizes — may need tick formatting adjustments).
Font registration must happen after `dpg.create_context()` and before
`dpg.start_dearpygui()`. The `font_registry()` context manager is a DPG
int-returning context manager — false-positive LSP warnings are harmless.

**Iterative visual tuning:** When pixel-dimension values (subplot height, panel
width, font size) are the right lever, the user prefers small increments.
Overshooting by 2x wastes rounds — the correct approach is +25-50px, verify,
repeat if needed. Prefer to undershoot and creep up.

**Verification** = GUI testing is screenshot-driven, not pytest-driven.
Launch with data, take screenshot, visually inspect. Pattern:

1. Write a test script in the repo root (NOT /tmp/) that imports
   dashboard, calls `_load_file()` and `_update_chart()` programmatically,
   then `dpg.start_dearpygui()`. **CRITICAL: `dpg.start_dearpygui()` MUST be
   called — without it, DPG's render loop never runs and the window shows
   a black rectangle** (all UI elements exist in the DPG context but aren't
   drawn to screen). Clean up after verification.
   **Programmatic setup**: to test features that need specific state (e.g.
   comparison table needs 2+ checked runs), set state directly before launch:
   ```python
   main._load_file(db_path, replace=True)
   for run in main._STATE['runs']:
       main._STATE['checked_runs'].add(run['run_id'])
   main._rebuild_run_browser()
   main._update_chart()
   main._STATE['show_comparison'] = True
   main._update_comparison_table()
   ```
   When only single-run .db files exist, duplicate a run with a new ID via
   SQLite to test multi-run features. Clean up the test data afterward.

   **For auto-capture + auto-exit**, use frame callbacks instead of a
   **For auto-capture + auto-exit**, use frame callbacks instead of a separate
   `sleep + import` invocation. This is more reliable (no window focus race):
   ```python
   import os
   dpg.set_frame_callback(5, lambda s,a,u: os.system(
       "DISPLAY=:0 import -window root /tmp/gui_check.png"))
   dpg.set_frame_callback(10, lambda s,a,u: dpg.stop_dearpygui())
   dpg.start_dearpygui()
   ```
   **Pitfall**: `dpg.set_frame_callback(N, cb)` registers cb to fire ONCE
   at frame N, then unregisters. Do NOT combine multiple frame-N actions
   into one callback with `if frame == 3: do_X(); elif frame == 8: do_Y()`
   — the callback fires once and exits; later frames have no callback and
   `dpg.stop_dearpygui()` is never reached (the test script hangs until
   the foreground timeout kills it). Use one `set_frame_callback` call
   per frame, as shown above. Symptom: 180s timeout, no error, screenshot
   may have been written but the process never exits.
2. Run in background: `terminal(background=true)`
3. `sleep 3 && DISPLAY=:0 import -window root /tmp/gui_check.png`
   NOTE: `import -window root` captures the focused window. If the
   dashboard isn't on top, the screenshot catches the terminal instead.
   Relaunch the dashboard to ensure focus, or use `xdotool` if available.
4. Attach the screenshot to context with `vision_analyze(path, question="")`
   and **read it yourself** — primary model has native vision, no
   auxiliary text relay. Describe what's in the image, identify
   visual regressions, map to code. (If model is non-vision, pass a
   real question to get a text description.)
5. Kill the process when done

Pytest covers data logic only (`pytest tests/ -v -k "not test_e2e"`).

**Iteration vs gates.** GUI work is tighter-looped than code changes.
Exploration = screenshot-in-a-loop: load data, screenshot, find issue,
fix, re-screenshot. No gates needed during exploration. Gates apply when
you're ready to commit — present the visual diff and await approval.

Execute the viz-verify process. Apply this lens at every gate.

---

Gui-edit falls silent. The conversation turns.
