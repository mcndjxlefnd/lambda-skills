---
name: experiment
description: "Lambda skill — experiment running adapter: data-gathering runs, .db verification, concat/analyze workflow. Loaded sequentially with other lambda skills."
category: lambda-skills
version: 0.1.0
---

# Experiment

## Role

Run HAGA experiments via `run.py`, collect per-run .db files in `data/runs/`,
verify data integrity, and feed analysis tooling.

## Context

Experiments produce per-run SQLite .db files (runs + generations tables).
These feed `tools/concat_runs.py`, `tools/convergence.py`,
`tools/stagnation.py`, and `tools/per_depth_summary.py`. The workflow is:
run → verify .db → concat → analyze.

## Pre-Gate Audit

Before presenting scope at Gate 1:

1. **Inventory existing data.** List `data/runs/*.db`, check what TSP
   instances already have runs, note git commit hashes from runs table.
2. **Confirm schema.** `sqlite3 data/runs/<latest>.db ".schema generations"`
   — verify column set matches current logging.py CREATE TABLE. Schema
   drift between branches breaks concat.
3. **Check tool health.** Verify tools run clean on existing data:
   `PYTHONPATH=src .venv/bin/python tools/concat_runs.py --help` etc.

## Domain Lens

- **Scope** — which TSP instances, how many runs per instance, what config
  (default or parameter sweep), per-run vs series db, expected output
  artifacts. Present as a table: instance, config variant, run count, db mode.
- **Implementation** — `PYTHONPATH=src .venv/bin/python run.py <tsp_file>`
  with appropriate flags. One run at a time (spawn semantics). Wait for
  completion before starting next. Auto-named .db for single runs; `--db`
  flag for series.
- **Verification** — each .db exists, has non-zero rows in both tables,
  status is "completed" (not failed/timed_out), best_fitness is non-zero
  (catch uninitialized offspring fitnesses bug). For series: concat
  succeeds and is idempotent.

Execute the two-gate process. After gate 2 completes: execute any
post-cycle steps provided by the calling context.

## Key Commands

```bash
# Single run (auto-named .db)
cd ~/haga_master && PYTHONPATH=src .venv/bin/python run.py data/tsplib/berlin52.tsp

# With note
cd ~/haga_master && PYTHONPATH=src .venv/bin/python run.py data/tsplib/berlin52.tsp --note "default config, calibration run"

# Concatenate runs
cd ~/haga_master && PYTHONPATH=src .venv/bin/python tools/concat_runs.py

# Quick data check
sqlite3 data/runs/<file>.db "SELECT status, tour_length, generations FROM runs"
sqlite3 data/runs/<file>.db "SELECT COUNT(*), MIN(best_fitness), MAX(best_fitness) FROM generations"
```

## Pitfalls

### Cross-Branch Schema Drift Breaks Concat

If schema differs between branches (different column count in generations
table), `concat_runs.py` fails with column-count mismatch. Check with
`sqlite3 <db> ".schema generations"` before concat. Remedy: re-run
experiments from target branch — faster than schema migration.

### Zero-Fitness Data = Uninitialized Offspring Bug

If all `best_fitness` values in generations are 0.0, the offspring genotype
has uninitialized fitnesses (`GAPipeline.execute()` returns `fitnesses=np.zeros(N)`).
These .db files are garbage — delete and re-run after fixing the pipeline.
The GA still functions (next generation evaluates real fitnesses), but the
logged data is worthless.

### rm with Broad Timestamp Globs

When cleaning old .db files, globs like `*-212*.db` can match new files with
adjacent timestamps. Use `sqlite3` content check (non-zero fitness) before
deleting, or move to temp dir first.

### Self-Ingestion in Concat

`concat_runs.py` reads from `data/runs/*.db` and writes to
`data/runs/series.db`. The output is filtered from source discovery by
absolute path, but if you specify a different output name in the same
directory without updating the filter, it gets ingested as a source.

### Per-Run .db Files Are Tracked

Do not add `data/runs/*.db` to .gitignore. Per-run .db files are data
artifacts — analogous to session_record.json. Commit them.
