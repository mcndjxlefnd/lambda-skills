# Process Tree Multi-Selection Pattern

## Problem
DPG selectables auto-toggle on click. If the callback also toggles a Python-side
set, the two toggles cancel out. Rebuilding the tree loses visual highlight state
unless `default_value` is set.

## Solution: Three-Layer Pattern

### 1. Tree builder accepts `selected` set
```python
def build_process_tree(processes, callback, selected: set[str] | None = None):
    selected = selected or set()
    for p in processes:
        dpg.add_selectable(
            label=p["process_id"],
            default_value=p["process_id"] in selected,  # highlight matches state
            callback=lambda s, a, u: callback(u, ..., s),  # forward sender
            user_data=p["process_id"],
        )
```

### 2. Callback reads DPG state, doesn't toggle independently
```python
def _on_process_click(process_id, sender):
    # DPG already toggled the selectable — read its new state
    is_selected = dpg.get_value(sender)
    if is_selected:
        _STATE["selected_processes"].add(process_id)
    else:
        _STATE["selected_processes"].discard(process_id)
    _update_chart()
```

### 3. Lambda forwards sender as third positional arg
DPG callback signature is `(sender, app_data, user_data)`. The lambda must
capture sender (`s`) and pass it through:
```python
lambda s, a, u: callback(u, extra_arg, s)
#                  ^user_data       ^sender forwarded
```

## Common Mistakes
- Toggling `selected.discard/add` without checking `dpg.get_value(sender)` → double-toggle
- Not passing `default_value` on rebuild → visual state lost after tree refresh
- Lambda `lambda s, a, u: callback(u)` drops sender → callback can't read DPG state
