# DearPyGui State Management Pitfalls

## Case Study: Comparison Table Never Appeared (2026-05-29)

**Symptom**: Checking 2 runs should show a "Show Comparison Table" toggle. It never appeared.

**Root cause trace**:
```
_on_run_check(run_B)          # adds run_B to checked_runs → now {A, B}
  → _rebuild_process_tree()   # len(checked)==2 → new_active=None
    → _clear_chart()          # reset checked_runs=set(), show_comparison=False
  → _update_chart()           # checked is now empty → toggle stays hidden
```

The `_clear_chart()` function was a "reset everything" function that wiped user
selections. It was called from `_rebuild_process_tree` when 0 or 2+ runs were
checked (i.e. no single run owned the tree).

**Fix**: Split responsibilities:
- `_clear_chart()` → clears display series only (line series values, visibility).
  Does NOT touch `checked_runs`, `selected_processes`, or `show_comparison`.
- `_load_file(replace=True)` → the "new file" clean-slate. Resets all state
  including `checked_runs` and comparison toggles.

**General rule**: In DPG apps with multi-select + toggle features, audit every
"clear" or "reset" function for over-resetting. If a cleanup function resets
selection state, it will break any flow where selection is accumulated across
multiple callbacks before the final update runs.

## Cross-DB Comparison Table

When runs come from different .db files (via "Add .db..."), each run carries
its own `db_path`. The comparison query must group by db_path:

```python
runs_by_db: dict[str, list[str]] = {}
for rid in checked:
    db = runs_by_id[rid]["db_path"]
    runs_by_db.setdefault(db, []).append(rid)
rows = []
for db_path, rids in runs_by_db.items():
    rows.extend(comparison_data(db_path, rids, milestones))
```

Otherwise only the first run's db is queried and runs from other files are
silently dropped.
