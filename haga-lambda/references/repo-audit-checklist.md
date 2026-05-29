# Repo Audit Checklist

Systematic repo state assessment. Use when returning to the repo after time
away, switching branches, or assessing what needs doing next.

## 1. Branch State

- `git branch --show-current` — which branch?
- `git status --short` — clean tree? Any untracked crud?
- `git stash list` — any stashed WIP?
- `git log --oneline <branch> -20` — what's been done recently?
- `git log --oneline <branch>..<other> --left-right` — divergence from other branch

## 2. Test Health

Run full suite minus e2e on current branch:

```
cd ~/haga_master && PYTHONPATH=src .venv/bin/python -m pytest tests/ -k "not test_e2e_12_city_tsp and not test_npy_cleanup_on_failure"
```

If any fail, flag them before touching code.

## 3. Issues Tracker

- `docs/issues-open.md` — what's pending?
- `docs/issues-closed.md` — what was recently resolved?
- Check for Q-items that may be blocked by missing infrastructure
- Cross-reference with plan docs for completion state

## 4. Plan Completion State

- `docs/plans/` — grep for "Completed" subsections vs latest commits
- Does the plan say Phase N is complete but no commits reflect it?
- Are there phases marked complete that have no completion log commit?

## 5. Schema Consistency

When multiple files define the same table schema, diff their `CREATE TABLE`
statements:

```
grep -r "CREATE TABLE" src/ tools/ tests/ --include="*.py"
```

Key tables to check:
- `generations` — `src/haga/logging.py`, `tools/concat_runs.py`, test helpers
- `runs` — `src/haga/logging.py` (two sites), `tools/concat_runs.py`

Column count mismatch → `INSERT INTO ... SELECT *` will fail.

Also check the `log()` method's positional INSERT vs the schema — every
column in the schema must have a corresponding parameter in the INSERT.

## 6. Cross-Branch Schema Drift

When two branches have diverged, diff their `CREATE TABLE` statements:

```
git diff <branch-a>..<branch-b> -- src/haga/logging.py
git diff <branch-a>..<branch-b> -- tools/concat_runs.py
```

Column differences mean .db files from one branch can't be ingested by
tools on the other branch. See pitfall: Cross-Branch Schema Drift Breaks
Tool Ingestion.

## 7. Dead Code

Grep for patterns specified in "Defensive Programming Is Rejected" pitfall:

- `isinstance(` in `src/haga/` — guards on types guaranteed by construction
- `.get(` in `src/haga/` — defaults on keys guaranteed by protocol
- `| None` type annotations on always-set attributes
- Methods defined but never called anywhere (`close()`, etc.)

## 8. Data Artifacts

- `data/runs/` — .db files present? How many? Which instances?
- `data/tsplib/` — TSPLIB files and config JSONs tracked?
- Any `.npy` files that leaked from failed runs?
- Untracked data files in other directories?

## 9. .gitignore

- Are per-run .db files in `data/runs/` NOT gitignored? (They should be tracked.)
- Is `data/runs.db` (the old shared file) gitignored?
- Any new patterns added without approval?

## 10. Tools Consistency

- `tools/concat_runs.py` — schema matches `logging.py`?
- `tools/convergence.py` — reads columns that exist in current schema?
- `tools/stagnation.py` — same
- `tools/per_depth_summary.py` — same
