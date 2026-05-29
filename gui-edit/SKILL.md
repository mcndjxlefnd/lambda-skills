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

## Lens

**Scope** = visual problems mapped to dashboard code. Find widget creation
sites with `search_files(pattern="<visible text>", path="haga_tools/dashboard")`.
Read around the match for layout context (parent group, spacers, siblings).

**Implementation** = edit the GUI code under `haga_tools/dashboard/`. DPG
constraints:
- Line series coloring: `dpg.bind_item_theme(tag, theme)` — NOT `configure_item(color=)`
- Subplots: parameter is `columns` not `cols`
- Timers: frame callback, no `set_timeout`
- Clipped text: `dpg.add_spacer(height=N)` before the element
- Visibility: `dpg.configure_item(tag, show=True/False)`
- Stale values: `dpg.set_value(tag, new_text)`
- Axes auto-scale: `dpg.fit_axis(axis_tag)`

**Verification** = re-launch: `PYTHONPATH=src .venv/bin/python -m
haga_tools.dashboard.main`. Run `pytest tests/ -v -k "not test_e2e"`.

Execute the viz-verify process. Apply this lens at every gate.

---

Gui-edit falls silent. The conversation turns.
