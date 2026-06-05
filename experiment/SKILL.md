---
name: experiment
description: "Lambda skill — experiment running adapter: data-gathering runs, .db verification, concat/analyze workflow. Loaded sequentially with other lambda skills."
category: lambda-skills
version: 0.5.0
---

# Experiment

## Role

Run HAGA experiments via `run.py`, collect per-run .db files in `data/runs/`,
verify data integrity, and feed analysis tooling.

**Research execution entry point:** `docs/roadmap.md` (in the experiments repo)
organizes current priorities with an "Immediate" section for what to do next.
Actionable S-items live in `docs/issues-open.md`. When the user says "start the
roadmap" or "begin carrying out the research plan," load docs/roadmap.md and
execute "Immediate" items in order, referencing issues-open.md for details.

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

## Gates

Gate 1 (permission to implement):
- Present scope table: TSP instance, config variant, run count, db mode.
- Confirm schema matches current logging.py (if new columns, flag).
- State which existing .db files will be kept vs retired.
- Note any code changes required (variant toggle, run_id patch).

Gate 2 (permission to commit):
- List all .db files produced with row counts and status.
- Verify: non-zero best_fitness, status="completed", expected generation count.
- Present `git diff --stat` if code was changed.
- Propose commit message with variant name and replicate count.

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
  When populating from an external source (LLM chat, design doc, article,
  meeting notes), the `Source` field is a URL or quoted excerpt, not a repo
  path. Bias to "simple, single-knob variants of existing code" — when the
  user says "some simple experiments," 5-8 items, not an exhaustive dump.
  Each item should be independently runnable. Tag the highest-leverage item
  as `planned`, others as `idea`. If the source arrives in chunks ("I'll send
  one exchange per message, stand by"), hold and acknowledge each chunk
  briefly; do not process until the user signals done.
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

4. For each variant:
   a. Commit the variant code change FIRST as a standalone commit:
      `git add <changed-files> && git commit -m "fix-mi: short description"`.
      This captures the variant implementation as a distinct state before
      any runs.
   b. Run all replicates.
   c. Commit code + all .db files together as `test-{variant-name}-r{N}[-{condition}]`:
      `git add -A && git commit -m "test-elite-on-r3-mi-fix"`.
      `r{N}` is the total replicate count in this commit (`r3` = 3 reps
      in the results snapshot).
      This captures both the source change AND the run data in one
      self-contained results snapshot. Push immediately.
   d. **Incremental expansion:** when adding more replicates later
      (e.g. 3 → 5), commit with `r{new_total}`:
      `git add -A && git commit -m "test-elite-on-r5-mi-fix"`.
      The old results commit remains in history. Don't re-commit the
      code change — it's already captured.
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

### Upstream Sync (Dashboard Commits)

`upstream/dashboard` (the haga_master fork point) occasionally gets GUI fixes,
`.gitignore` changes, and other non-experiment commits. These need to propagate
to all experiment branches to keep the fork's dashboard usable.

**When:** Listed as a recurring task in docs/roadmap.md §Immediate. Also
trigger when the user says "sync the dashboard" or after upstream/dashboard
acquires new commits.

**Workflow:**

1. **Identify missing commits.** `git log --oneline upstream/dashboard --not origin/dashboard`
   on the `dashboard` branch. Shows what's on upstream but not on your origin.

2. **Classify.** Only cherry-pick GUI fixes and gitignore changes. Skip commits
   containing test scripts, .db data files, screenshots, or local checkpoint
   junk — these are session artifacts that don't belong in the experiments repo.

3. **Cherry-pick to dashboard.** On the `dashboard` branch:
   `git cherry-pick <sha1> <sha2> ...`
   Handle empty cherry-picks (already-applied changes like `.codegraph/` gitignore
   entries) with `git cherry-pick --skip`.

4. **Propagate to experiment branches.** Checkout each branch and cherry-pick
   the same commits:
   ```
   git checkout mi-sweep && git cherry-pick <sha1> <sha2>
   git checkout select-press && git cherry-pick <sha1> <sha2>
   git checkout elite-delete && git cherry-pick <sha1> <sha2>
   ```
   These are pure GUI/gitignore changes — no experiment-code impact, so
   conflicts are unlikely.

5. **Push all branches:**
   ```
   git checkout dashboard
   git push origin dashboard mi-sweep select-press elite-delete
   ```

6. **Verify.** Import the dashboard modules to confirm they resolve:
   `PYTHONPATH=. .venv/bin/python -c "from haga_tools.dashboard.main import main; print('OK')"`
   The dashboard needs `PYTHONPATH=.` (repo root) because `haga_tools/` lives
   at repo root, not under `src/`.

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
1. Switch to the experiment branch in `~/haga_experiments/`
2. Implement the variant code change (write_file or patch)
3. Set a distinguishable `run_id`: patch `run.py` line 129 from
   `f"{instance_name}-run"` to `f"{instance_name}-<variant>-repN"`
4. Commit the variant code change FIRST as a standalone commit:
   `git add <changed-files> && git commit -m "fix-mi: short description"`
   This captures the variant implementation before any runs.
5. Run each replicate independently from the experiments repo using
   haga_master's venv:
   `cd ~/haga_experiments && PYTHONPATH=src ~/haga_master/.venv/bin/python run.py data/tsplib/<instance>.tsp --db "data/runs/<instance>-<variant>-repN_$(date +%Y-%m-%d-%H%M%S).db" --note "<description>"`
6. After ALL replicates for a variant are done, commit the code state +
   all .db files together as `test-{variant-name}-r{N}[-{condition}]`:
   `git add -A && git commit -m "test-elite-on-r3-mi-fix"`
   (`r3` = 3 replicates in this snapshot).
   This captures both the source change AND the run data in one atomic
   results snapshot. Push immediately.
7. Revert source file to original if the change was a variant toggle
   (e.g. one-line return in a method). Keep the change if it's a
   permanent improvement (e.g. a bug fix shared by all variants).
8. Implement next variant or finalize.

**Key principle:** commit in two phases per variant. First commit the code change
alone (variant implementation). Then after all replicates finish, commit code + .db
files together as the results snapshot. This gives each commit a clear role: one
for "what I changed" and one for "what happened."

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

## Replicated Runs (Multiple Replicates Per Variant)

For statistical confidence, run each variant N times (3 is a common minimum).
All post-hoc analysis (mean, variance, confidence intervals) is done across
replicates at the analysis stage — *not* by merging data at the DB level.

**Selection-fraction experiments** follow the same pattern with abbreviated
naming: run_id = `f{percent}-rep{N}`, filename = `f{percent}-rep{N}_{timestamp}.db`.
The variant IS the fraction (e.g. `f10`, `f25`, `f75`). No instance prefix in
the filename — the instance is constant across the sweep.

For statistical confidence, run each variant N times (3 is a common minimum).
All post-hoc analysis (mean, variance, confidence intervals) is done across
replicates at the analysis stage — *not* by merging data at the DB level.

**Workflow:**

0. **Verify RDRAND independence.** Before running replicates, confirm the RNG
   strategy is stateless hardware entropy per call. `RDRandom` uses the
   `rdrand64` CPU instruction directly — no seed, no shared PRNG state, each
   call is fresh entropy. Replicates don't need explicit seeding. If the config
   uses a different RNG, confirm independence before proceeding.

1. **Run each replicate independently.** Use run_id suffix:
   `{instance}-{variant}-rep1`, `{-rep2}`, etc. Patch run.py line 129 before
   each replicate.

2. **Analysis cross-replicate.** The DB is per-replicate. To compute aggregate
   curves (mean best_fitness per generation across 3 replicates):

   ```python
   # Pseudocode — concat into one DB, then group by generation, avg
   sqlite3 series.db "
   SELECT generation, AVG(best_fitness) AS mean_best,
          AVG(mean_fitness) AS mean_mean,
          VAR_SAMP(best_fitness) AS var_best
   FROM generations
   GROUP BY generation
   ORDER BY generation"
   ```

   For full cross-replicate aggregation across multiple .db files:
   ```bash
   PYTHONPATH=src .venv/bin/python tools/concat_runs.py  # produces series.db
   ```
   Then query series.db grouping by generation.
   **Note:** concat deduplicates on `runs.run_id` (PRIMARY KEY) — all replicates
   MUST have unique run_ids or only the first per duplicate is merged.

3. **Naming.** run_ids follow `{instance}-{variant}-rep{N}` pattern (or
   `{variant}-rep{N}` for selection-fraction sweeps where the fraction IS the
   variant). DB filenames follow the same: `{variant}-rep{N}_{timestamp}.db`.
   The run_id (not filename) is what the dashboard labels the run.

   For selection-pressure experiments with Top{F}Selection strategies,
   variant = `f{percent}` (e.g. `f10`, `f25`, `f75`), giving filenames
   like `f10-rep1_2026-05-31-120000.db`.

4. **No DB-level merging of replicates.** Each replicate is its own row in
   `runs` table. The `generations` table has separate process_id chains per
   replicate. Queries filter by run_id and aggregate.

## Experiment Planning (Design Phase)

Before running any code, produce a plain-text planning document at
`docs/plans/<name>-plan.md` (in the experiment repo) covering:

- **Hypothesis** — what question are you answering? What outcome would
  confirm/reject?
- **Parameter sweep** — which strategy parameter(s), what values, what
  resolution. Justify the range.
- **Replication** — how many runs per value. 3 is baseline.
- **TSP instance** — choose one (lin318, berlin52, etc.). Keep constant
  across all variants in one sweep.
- **Schema requirements** — if the experiment logs a parameter that isn't
  in the current `generations` table schema, list the new column and its
  type. See `references/parameterizing-strategies.md`.
- **Aggregation plan** — how will you combine similar runs? Mean/median per
  generation? Final tour length distribution? Convergence time?

  **Visualization preference:** For group comparison (overlaying N parameter
  values against each other), use arithmetic mean — one clean solid line per
  group. No envelopes, no shaded bands, no error bars. Four lines per plot is
  the target; more than that becomes illegible. If the lines diverge, the
  groups are separable. If they merge, they aren't. Within-group spread
  (envelope) is secondary to between-group separation — add it only as a
  diagnostic view if a threshold result is ambiguous.

  Apply this to ALL logged metrics, not just best_fitness. The sigma
  trajectories (best_mut_intensity, mean_mut_intensity, variance) often carry
  more signal about parameter effects than convergence curves — they reveal
  how the GA's adaptive dynamics change under different regimes, not just
  whether it converges faster or slower.

## Cross-Branch Experiment Audit

Review and synthesize findings across all experiment branches in the experiments
repo. Used when the user asks for the state of all experiments, or to identify
cross-branch gaps and dependencies before starting a new experiment.

### Workflow

1. **Inventory branches.** `git branch -a` lists both local and remote branches.
   Include `upstream/` when present (upstream = haga_master fork point).

2. **Full commit logs per branch.** `git log --oneline <branch> --` for each
   experiment branch. Identify the fork point (the last commit shared with the
   pristine branch — everything above it belongs to the experiment).

3. **For each experiment branch, checkout and read:**
   - `experiment.json` — machine-readable metadata, variant list, results.
     Check BOTH root AND `<experiment-name>/` subdirectory — document location
     drifts between experiments.
   - `lab-notebook.md` — prose analysis, conclusions, open questions.
     Try root, `docs/`, and `<experiment-name>/`.
   - `docs/ideas-open.md` / `docs/ideas-closed.md` — idea trackers (often empty
     template shells on early branches).
   - `docs/plans/` — execution plans and design decisions (on branches that
     include them).
   - `experiments-logbook.md` or `logbook.md` — chronological master journal at
     root level, if present.

4. **Cross-reference findings across all branches:**
   - **Cross-branch fix gaps** — does a known fix on one branch need
     cherry-picking to another (e.g., the mi-fix on `elite-delete` commit
     `021b03a` that `select-press` lacked)?
   - **Document location drift** — note per-branch where artifacts live.
     Three patterns: root-level (`elite-delete`), subdirectory (`mi-sweep`
     uses `2026-05-30-mi-sweep/`), and `docs/` (`select-press` uses
     `docs/lab-notebook.md`).
   - **Naming inconsistencies** — branch names vs directory names vs documented
     conventions. Logbook.md may reference `exp/<date>-<slug>` conventions that
     were never actually used.
   - **Common conclusions** — do findings converge across experiments? (e.g.,
     "selection pressure dominates convergence endpoint regardless of mutation
     regime" appears in both mi-sweep and elite-delete).
   - **Incomplete experiments** — identify branches with negative controls
     awaiting re-run, or documented re-run plans that were never executed.
   - **Ideas trackers** — check if real content exists vs template-only.

5. **Compile structured summary.** Per branch: fork point, commit count,
   research question, key findings, status, document locations, open questions,
   cross-branch dependencies.

See `references/cross-branch-audit-checklist.md` for an output template.

### Pitfalls

- **Branch naming ambiguity with `git log`.** `git log --oneline <branch>` may
  fail when branch name collides with a file/directory name. Use
  `git log --oneline <branch> --` (trailing `--`) to disambiguate.
- **Stale experiment.json.** The root-level file on some branches is a scaffold
  written before running, while a subdirectory copy has the actual post-run data.
  Always check ALL locations.
- **Logbook name drift.** Three patterns: `experiments-logbook.md`, `logbook.md`,
  `docs/lab-notebook.md`. Grep broadly for any of these. Also check for
  `docs/lab-notebook.md` vs root-level ones.
- **Ideas trackers may be empty.** `docs/ideas-open.md` and `docs/ideas-closed.md`
  are often template-only on early experiment branches. Check actual content
  length before reporting findings as significant.
- **Upstream branches are NOT experiments.** `remotes/upstream/master` and
  `remotes/upstream/dashboard` are pristine haga_master history — the fork
  point. Skip when auditing experiments.
- **Commit message conventions vary.** Branches use `test-`, `exp()`, `fix-`,
  `run-config:`, or plain prefixes. Don't filter by convention — read all
  unique commits above the fork point.
- **Cross-branch schema drift.** .db files on branches with different
  `generations` table schemas can't be concatenated. Note if schema
  differences matter for the analysis.
- **Fork point identification.** The experiment's commits start at the first
  commit NOT shared with the pristine branch (usually `dashboard` or `master`).
  Everything below is shared haga_master history — ignore when auditing
  experiment-specific work.

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

### Population Sizing Edge Cases (N % 4, mid=0, tau Dead Zone)

See `references/population-sizing-edge-cases.md` for three known bugs that
crash experiment runs or make results indistinguishable:

1. **SelectionAlignedScaling** may produce N not divisible by 4, causing
   IndexError in StriatedReproduction (fix: post-round to multiple of 4).
2. **StriatedReproduction** crashes when n_parents < 2 (mid=0) on small
   leaf populations (fix: guard mid, clamp right index).
3. **TripleIndependentSigma dead zone** — mut_intensity frozen at 1.0 for
   M>=200, making selection-fraction effects invisible. Fix exists on
   `elite-delete` commit `021b03a` (decouple tau_mi, float intensity).

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

### Don't Override --db for Single Runs Without Explicit Request

Auto-naming produces `{instance}_{timestamp}.db` — this IS the convention.
The filename is NOT the run's title (the title is `runs.run_id`, default
`{instance_name}-run`). Forcing `--db` to a custom name on auto-named singles
breaks the established pattern. Only use `--db` when the user explicitly asks
for a specific output path (e.g., series mode, or variant experiments with
custom `run_id`). Default behavior is correct.

### Concat Deduplicates on run_id Primary Key

`concat_runs.py` uses `INSERT OR IGNORE INTO runs SELECT ...` — `runs.run_id`
is the PRIMARY KEY. If all replicates share the same run_id (e.g.
`lin318-f10` for all 3 f10 reps), only the FIRST is merged; the rest are
silently dropped. **Fix:** patch `run.py` line 129 to include the replicate
suffix before each replicate: `{instance}-{variant}-rep{N}`. The filename
(`--db` flag) is separate from the run_id label that concat sees.

**Exception:** Variant experiments where the user wants the option name in
both the run_id AND the filename. In that case, use `--db` with
`{instance}-{variant}_{timestamp}.db` AND patch `run.py` run_id to
`{instance_name}-{variant}`. This is an explicit user request, not a silent
override.

### Two-Phase Commit Pattern

For each variant group, make TWO commits:
1. **Code change commit** — `git add <changed-files> && git commit -m "fix-mi: ..."`
   Captures the variant implementation before any runs. No .db files exist yet.
2. **Results commit** — `git add -A && git commit -m "test-elite-on-r3-mi-fix"`
   Code state + all .db files from the completed replicates together.

`r{N}` in the results commit name indicates the total replicate count
(`r3` = 3 replicates in this snapshot). When expanding incrementally
(e.g. 3 → 5), use `r{new_total}` (`test-elite-on-r5-mi-fix`). The commit
message body should note it's an append. The old results commit remains
in history — each snapshot is independently verifiable.

Don't skip the code-change commit — without it, the variant's code state is
only visible as a diff between results commits, not as a named point in
history. Don't run replicates before committing the change either — if
runs produce data under an uncommitted code change, the .db files have no
code-pedigree anchor. If the user corrects this, their preference is the
two-phase pattern: first commit the code change, then run, then commit
code + .db files together.

### Permanent Fix vs Variant Toggle

Not every code change is a variant — distinguish between:
- **Permanent fix** — a bug fix or improvement that applies to all variants
  (e.g. decoupling tau_mi, removing int(round())). Commit it once, keep it
  across all variant runs. Don't revert between groups.
- **Variant toggle** — a one-line behavioral switch (e.g. return-early to
  disable elitism). Revert after running the variant group. Only this gets
  toggled per variant.

### Selection Fraction × FifthBreedingReplacement Clean Integer Constraint

When sweeping the selection fraction `f` (parent pool = `N * f`), verify the
interaction with `FifthBreedingReplacement`'s hardcoded `ceil(P * 0.20)`.
If `N * f * 0.20` is not integer, the ceil introduces fractional-difference
noise between variants. For lin318 (N=200), the constraint is `40f ∈ ℤ` —
satisfied by f ∈ {0.10, 0.20, 0.25, 0.30, 0.40, 0.50, 0.60, 0.75, 0.80}.
Always verify for the specific TSP instance's computed N before proposing
sweep values.
