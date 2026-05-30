## DPG "clear" functions reset too much state

A `_clear_chart()` that resets user selections (`checked_runs`, `selected_processes`)
or UI toggles (`show_comparison`) breaks multi-selection flows. The callback chain
is:

1. User checks second run → `_on_run_check` adds to `checked_runs` (now 2)
2. `_rebuild_process_tree` sees 2 runs → `new_active = None` → calls `_clear_chart()`
3. `_clear_chart` resets `checked_runs = set()` — wipes selections
4. `_update_chart` sees empty `checked_runs` → comparison toggle stays hidden

**Rule:** `_clear_chart` must only clear **display series** (line series values,
visibility, axis labels). It must NOT touch:
- User selections (`checked_runs`, `selected_processes`)
- UI toggles (`show_comparison`, `normalize_x` state)
- Checkbox widget values (`dpg.set_value("comparison_toggle", ...)`)

Full state reset belongs in `_load_file(replace=True)` only — that's the
"new file" clean-slate path. When you see a cleanup function that resets
`checked_runs` or selection state, that's the bug.

**Also applies to:** any DPG app with multi-select + derived views. The pattern
is "clear visual" vs "reset interaction state" — these are different operations
that happen to live in the same function by accident.
