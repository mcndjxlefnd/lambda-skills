# Strategy Parameterization

How to make a hardcoded constant in a strategy into a configurable
constructor parameter, threaded through the session config.

## Pattern

Most strategies have hardcoded constants (e.g. `Top50Selection` picks
`N // 2`, `FifthBreedingReplacement` uses `* 0.20`). To sweep them:

### 1. Add constructor parameter

```python
class TopPSelection(ISelection):
    """Select top fraction p of population as parents."""

    def __init__(self, selection_fraction: float = 0.5):
        self._fraction = selection_fraction

    def select(self, genotype: "Genotype") -> np.ndarray:
        N = len(genotype.fitnesses)
        k = max(1, math.ceil(N * self._fraction))
        sorted_indices = np.argsort(genotype.fitnesses)[::-1]
        return sorted_indices[:k].astype(np.int32)
```

### 2. Register (or replace existing) in strategy_registry.py

```python
STRATEGY_REGISTRY = {
    # Replace hardcoded entry with one that takes a kwarg:
    "top_p_selection": (
        "haga.strategies.selection.top_p_selection",
        "TopPSelection",
    ),
    # OR keep the original and add the new one alongside
}
```

### 3. Thread config through SessionConfig

Two approaches, increasing in generality:

**A. Single-field (preferred for one-parameter sweeps):** Add a typed field
directly to the `SessionConfig` Pydantic model in `config.py`:

```python
class SessionConfig(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid")
    strategies: dict[str, str]
    selection_fraction: float = 0.5
    base_case_strategy: str
    subordinate_config_path: str
```

Then in `entrypoint.py`, access via `ipc.session_config.selection_fraction`
(after `ipc.initialize()` populates it) and pass as kwargs:

```python
selection = load_strategy(
    strat_ids["selection"],
    fraction=ipc.session_config.selection_fraction,
)
```

This is simpler and avoids nested dict parsing. The config JSON written by
`build_root_config()` includes the field automatically. Subordinate processes
read and validate it (harmless — they don't use it).

**B. Generic strategy_kwargs (for multi-parameter sweeps):** The strategy
config dict passes `**kwargs` to `load_strategy()`:

```json
{
  "strategies": {
    "selection": "top_p_selection",
    "selection_kwargs": {"selection_fraction": 0.25}
  }
}
```

`load_strategy` does `cls(**kwargs)`, so any kwargs from the config dict
arrive at the constructor.

## Schema Changes for Swept Parameters

If you want the swept parameter value logged in the `generations` table
for every generation (e.g., `selection_fraction REAL`):

1. **Add column to CREATE TABLE** in `SQLiteGenerationLogger._create_tables()`
   (logging.py lines 65-84). Use `ALTER TABLE IF NOT EXISTS` or add to the
   CREATE TABLE directly (the CREATE uses IF NOT EXISTS, so it won't re-create).

2. **Expose via `**extra`.** The `GenerationLogger.log()` method accepts
   `**extra`. Pass the parameter value from `AGA`:

   ```python
   self.logger.log(
       ...,
       selection_fraction=self._selection_fraction,
   )
   ```

3. **Insert the extra value.** `SQLiteGenerationLogger.log()` builds its
   INSERT with positional args plus `extra.get("selection_fraction")`. If
   the column is new, add it to both the column list and the positional
   tuple.

4. **Schema drift risk.** If you add a column on the experiment branch,
   .db files from other branches won't have it. The concat tool inserts
   into fixed columns — missing columns cause errors. Either:
   - Accept schema drift (analysis is per-branch, .db files stay on branch)
   - Add the column as nullable (DEFAULT NULL) everywhere
   - Conditionally concat only matching-schema .db files
