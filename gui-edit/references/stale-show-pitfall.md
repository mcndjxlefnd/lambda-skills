# Stale-Show Pitfall: Set-Iterated Render Loop

## Case Study: Deselected Process Lines Don't Disappear (2026-05-31)

**Symptom**: User selects 3 processes in the tree — their line plots
appear in the correct colors. User deselects one — the highlight flips
off in the tree, but the line plot stays drawn. Only when the user
deselects the LAST remaining process do all the previously-deselected
lines vanish at once.

**Root cause**: `_update_chart()` in `haga_tools/dashboard/main.py`
renders process-level lines via:

```python
if selected:
    for run_id in checked:
        for pid in sorted_procs:        # iterates the subset
            tag = f"line_run_{run_id}_proc_{pid}"
            if dpg.does_item_exist(tag):
                dpg.set_value(tag, [xs, ys])
                dpg.configure_item(tag, show=bool(xs))
            else:
                dpg.add_line_series(...)
```

When a proc is deselected, the callback removes it from
`selected_processes`. The next `_update_chart` call iterates the new
(shrunk) `selected` set — the deselected proc's tag is never visited,
so its `show=True` (set during the previous update) is unchanged.

The line only disappears when `selected` becomes empty and the
function enters the `else` (run-level) branch, which runs a "Hide all
process-level series" sweep over the full domain. That's why the
final deselection looks like it cleans up everything at once.

**Trace of the callback chain** (confirming the symmetric hide is the
missing piece, not the toggle):

```
process_tree.py  →  add_selectable(default_value=pid in selected, ...)
  DPG auto-toggles on click  →  callback fires
  →  _on_process_click(pid, sender)
    →  dpg.get_value(sender)  # DPG's new state — sync, not toggle
    →  selected_processes.discard(pid)  # removed correctly
    →  _update_chart()
      →  render loop iterates remaining selected
      →  deselected pid's line_series tag: NOT VISITED, show still True
```

## Pre-existing partial fix

The render function already had a loop:

```python
# Hide stale process-level series from unchecked runs
for r in _STATE["runs"]:
    rid = r["run_id"]
    if rid in checked:           # skips checked runs entirely
        continue
    for proc in _STATE.get("tree_processes", []):
        ...
        dpg.configure_item(tag, show=False)
```

This covers (unchecked run, any proc) but NOT (checked run, deselected
proc). The bug was that the early `continue` skipped the entire
checked-run branch instead of falling through to a proc-level check.

## Fix: Unified Hide Sweep

Replace each of the three "Hide stale" loops (convergence, variance,
sigma) with one that hides when EITHER the run is unchecked OR the
proc is deselected. Equivalent formulation: hide unless `run in
checked AND proc in selected`.

```python
# Hide (run, proc) series that aren't part of (checked × selected)
for r in _STATE["runs"]:
    rid = r["run_id"]
    for proc in _STATE.get("tree_processes", []):
        pid = proc["process_id"]
        if rid in checked and pid in selected:
            continue
        tag = f"line_run_{rid}_proc_{pid}"   # prefix varies per chart
        if dpg.does_item_exist(tag):
            dpg.configure_item(tag, show=False)
```

This is one loop covering both stale cases. The prefix changes
(`line_`, `var_`, `sigma_<metric>_<suffix>_`) but the shape is
identical across the three charts. The "Hide all process-level series"
in the `else` (no selection) branch is the degenerate case of this
same sweep with `selected == ∅` — that one is correct as-is.

## General Pattern (Beyond DPG)

Any time a render loop is shaped like "for x in active_subset:
update_or_create", you need a symmetric "for x in full_domain where x
NOT IN active_subset: hide" — OR a unified sweep with the visibility
condition inverted. Skipping the complement is the bug.

Diagnostic one-liner: grep for `for .* in <set>` in a render function
and check whether anything in the *complement* gets touched.
