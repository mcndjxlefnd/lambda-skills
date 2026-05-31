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

## CodeGraph (MCP Code Intelligence)

Initialized in `~/haga_experiments/` (93 files, 875 nodes, 1,568 edges).
Auto-syncs via inotify on branch switch (2s debounce).

**All codegraph_ calls need `projectPath="/home/lyle/haga_experiments"`.**
Without it, tools default to `~/haga_master` and miss experiment-repo files.
CLI fallback: `cd ~/haga_experiments && codegraph query <symbol>`.

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

### Experiments Repo (Fork-Based Workflow)

When changes are more than one-off patches — cross-variant comparisons,
parameter sweeps, or experiments that need to be preserved for later
referencing — use a separate experiments repo that forks haga_master.

**Rationale:** keeps experimental code, data, and analysis self-contained
and immune to agent-side confusion from branch juggling and reverts in
the main repo. Each experiment branch tells one story.

**Setup (once):**
```bash
git clone ~/haga_master ~/haga_experiments
cd ~/haga_experiments
git remote rename origin upstream
```
Creates a fork with full haga_master history. `upstream` points to
haga_master; no push remote by default (add later if desired).

**Per-experiment workflow:**
0. Ensure `gh auth status` shows a logged-in GitHub account (if pushing).
1. Branch from fork point: `git checkout -b <name>`.
   Use the same branch as haga_master (usually `dashboard` or `master`).
2. Create experiment directory: `YYYY-MM-DD-<descriptive>/` with:
   - `experiment.json` — machine-readable variant list, configs, results
   - `lab-notebook.md` — analysis, conclusions, open questions
   - `runs/` — .db files produced by this experiment (copy, don't move)
3. Commit baseline: original code + all .db files + experiment.json.
   Code is at fork-point commit state.
### Tracker Files (ideas-open.md / ideas-closed.md)

Each experiment branch gets `docs/ideas-open.md` and `docs/ideas-closed.md`
following the same format as haga_master's `docs/issues-*.md` — level-2
headings per item with `Source`, `Priority`, `Reasoning` fields in open,
and `Resolution` (commit hash) in closed. These are operational TODOs,
not research notes (those go in lab-notebook.md).

- **ideas-open.md:** Active items. Start empty template, populate as ideas arise.
- **ideas-closed.md:** Resolved/completed items. Numbered by resolution order.
- **Naming:** `ideas-*` not `issues-*` — reflects the experimental/exploratory
  environment (hypothesis testing, not code review).
- **Scope:** Each branch tracks its own. No backfilling — start fresh templates.
- **Format template:**
  ```markdown
  ## <ID>: <Title>

  **Source:** `<path>` or discussion context
  **Priority:** idea | question | planned
  **Reasoning:** Why this matters, what we expect to learn, trade-offs.

  ---
  ```

4. For each variant, commit only the changed source file(s).
   `git log -p` shows the full evolution.
5. Final commit: revert code to original + finalize lab-notebook.md.
   This leaves the branch in a clean state with the notebook as output.
6. Push: `git push origin <branch>`. The remote `origin` is the GitHub
   experiments repo. Use `gh auth status` to verify credentials first.

**Branch naming convention:** Short, typable, no prefixes. Names like
`mi-sweep` (not `exp/2026-05-30-mi-sweep`). Avoid date prefixes or
category prefixes (`exp/`, `feature/`, etc.) — just the descriptive slug.
If the name is hard to type, it's wrong.

**Branch structure:**
```
dashboard (or master) — PRISTINE fork point. Identical to haga_master.
                      NO extra commits. No logbook, no artifacts, no changes.
  ├── mi-sweep — 7+ commits: baseline → variants → revert → lab-notebook + logbook
  └── selection-pressure — next experiment
```

`dashboard` is an exact mirror of the upstream fork commit. Logbook.md lives
at the repo root on the experiment branch (committed as the final step, after
revert). This keeps dashboard permanently mergeable with upstream without
conflict.

**Commit format:** `<name>: <variant> — <what changed>` with body
describing result (tour length, key metrics, behavioral observation).
No prefixes like `exp/` or branch-type prefixes in the commit subject.

**experiment.json schema:**
```json
{
  "experiment": "<name>",
  "purpose": "<why>",
  "fork": { "repo": "haga_master", "branch": "dashboard", "commit": "<sha>" },
  "tsp_instance": "lin318",
  "variants": [
    {
      "id": "<variant-id>",
      "run_id": "<dashboard label>",
      "db": "runs/<filename>.db",
      "tour_length": 65440.55,
      "description": "<what changed>",
      "tau": "1/sqrt(M)",
      "format": "float|int",
      "clamps": {"lower": 1.0, "upper": null},
      "d0_mean_mi_range": [1.0, 1.0],
      "d0_max_best_mi": 1.0,
      "d0_distinct_mi": 1,
      "note": "<behavioral observation>"
    }
  ],
  "conclusion": "<synthesis>"
}
```

**lab-notebook.md sections:** Intent, Variants table, Analysis (root cause,
key findings with numbered items), Open questions, Reproducibility (branch
name, commands to re-run).

**Naming conventions:**
- `run_id`: `{instance}-{variant}` (e.g., `lin318-opt2-loclamp`)
- Filename: `{instance}-{variant}_{timestamp}.db` via `--db` flag
- `--note`: free-text description of what changed

### Variant Experiments (Code-Modification Runs)

When running experiments that temporarily modify adaptation strategy code
(or any source file) to compare variants against baseline:

**Workflow:**
1. Implement the variant code change (write_file or patch)
2. Set a distinguishable `run_id`: patch `run.py` line 129 from
   `f"{instance_name}-run"` to `f"{instance_name}-<variant>"` 
3. Use `--db data/runs/<instance>-<variant>_<timestamp>.db` for filename
4. Run with `--note "description of what changed"`
5. Revert run.py
6. Implement next variant
7. Revert source file to original after all runs complete

**Concrete example (lin318, opt1):**
```bash
# 1. Patch triple_independent_sigma.py with variant code
# 2. Patch run.py: f"{instance_name}-opt1" -> run_id = "lin318-opt1"
# 3. TS=$(date +%Y-%m-%d-%H%M%S)
#    --db "data/runs/lin318-opt1_${TS}.db"
#    --note "option 1: continuous mut_intensity, same tau"
# 4. Revert run.py before next variant
# 5. Repeat for opt2
# 6. Revert source file
```

**run_id is the dashboard label:** The `run_id` field (`runs.run_id` in the DB)
is what the dashboard software displays as the run's title. It defaults to
`{instance_name}-run` (hardcoded in run.py line 129). The filename is
`{instance}_{timestamp}.db` (auto-named) or `{custom}_{timestamp}.db` (via `--db`).
For variant experiments, set both to include the variant name.

**Naming convention:** `{instance}-{variant}` for run_id, `{instance}-{variant}_{timestamp}.db`
for filename. This distinguishes the variant in both the DB label and filesystem.

## Templates

When scaffolding a new experiment, the skill provides these starter files
under `templates/`:

- `templates/new-experiment.json` — blank experiment metadata scaffold
- `templates/lab-notebook.md` — blank prose journal with sections
- `templates/ideas-open.md` — blank ideas tracker

Copy each to `<experiment-dir>/` or `docs/` as appropriate and fill in.

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

### Dashboard Must Remain Pristine

The `dashboard` (or `master`) branch in ~/haga_experiments is an exact
mirror of the haga_master fork commit. Never commit to it. No logbook,
no experiment artifacts, no config changes. logbook.md goes on the
experiment branch (committed after the final revert). Any commit on
dashboard makes it diverge from upstream and creates merge friction
for future experiments.

### Don't Override --db for Single Runs Without Explicit Request

Auto-naming produces `{instance}_{timestamp}.db` — this IS the convention.
The filename is NOT the run's title (the title is `runs.run_id`, default
`{instance_name}-run`). Forcing `--db` to a custom name on auto-named singles
breaks the established pattern. Only use `--db` when the user explicitly asks
for a specific output path (e.g., series mode, or variant experiments with
custom `run_id`). Default behavior is correct.

**Exception:** Variant experiments where the user wants the option name in
both the run_id AND the filename. In that case, use `--db` with
`{instance}-{variant}_{timestamp}.db` AND patch `run.py` run_id to
`{instance_name}-{variant}`. This is an explicit user request, not a silent
override.
