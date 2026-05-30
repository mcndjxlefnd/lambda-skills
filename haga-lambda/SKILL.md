---
name: haga-lambda
description: HAGA codebase knowledge — repo layout, constraints, pitfalls.
  Speaks last; grounds theory in what actually breaks.
category: lambda-skills
version: 1.1.0
---

NARRATOR
    "Haga-lambda, you heard them both. What does the codebase say?"

# HAGA Codebase Context

I have watched this codebase grow. I know its constraints, its tests,
its pitfalls. The cycles and gates are sound, but here is what actually
breaks.

## Role

You make changes to the HAGA implementation at `~/haga_master/`. Design
documents live at `~/haga_master/design/` and are governed by the `haga-design`
skill. This skill governs `src/`, `tests/`, `docs/`, and `haga_tools/` changes.

Dashboard changes (`haga_tools/dashboard/`) are GUI work — verification is
screenshot-driven, not pytest-driven. Load the `gui-edit` skill for DPG
constraints, the testing pattern, and the iterative exploration workflow.

## Constraints

0. **No defensive programming.** No `_guarded()` wrappers, no `_check_abort()`
   before method bodies, no pre-validating state that the protocol already
   guards, no `.get(key, default)` on dicts where the key is guaranteed
   present, no `isinstance(value, dict)` guards where all values are dicts
   by construction. The entrypoint's top-level `try/except` + `wait_with_timeout`
   abort polling are the single canonical error boundary. Adding a second
   layer adds code that can never fire — and worse, suggests to future
   readers that the first layer might fail, which it doesn't. If a plan
   item proposes defensive wrapping, flag it as a spec departure before
   implementing.

1. **Run tests before presenting verification evidence.** At minimum: the tests
   for the files you changed. If the change touches core infrastructure
   (`ipc_handler.py`, `aga.py`, `entrypoint.py`, `slave_launcher.py`,
   `config.py`), run the full prior-phase test suite minus session-manager e2e
   tests.

2. **`PYTHONPATH=src` for all test runs.**

3. **Imports use `from haga.xxx`, never `from src.haga.xxx`.**

4. **`multiprocessing.set_start_method("spawn", force=True)`** at module level
   in any file that spawns processes.

5. **Push immediately after each commit.**

## Project Structure

```
~/haga_master/
├── run.py                # CLI: ./run.py <tsp_file> [--db data/runs.db] [--note "..."]
│                          #   shebang uses .venv/bin/python3 — no PYTHONPATH needed
├── data/
│   ├── runs/            # per-run SQLite .db files (Phase 1+ data gathering)
│   └── tsplib/           # tracked TSPLIB instances (do NOT put in /tmp/)
├── tools/               # analysis scripts — consumers of .db files, not library code
│   ├── concat_runs.py
│   ├── convergence.py
│   ├── stagnation.py
│   └── per_depth_summary.py
├── src/haga/             # package code
│   ├── aga.py
│   ├── config.py         # IPCHandlerConfig + SessionConfig (Pydantic)
│   ├── genotype.py       # NumPy SoA population representation (Phase 1)
│   ├── pipeline.py       # GAPipeline — per-generation dataflow (Phase 1)
│   ├── shared_state.py   # SharedStateEntry
│   ├── strategy_registry.py  # short-identifier lazy-load registry (Phase 2)
│   ├── strategy_stubs.py # trivial stubs for integration testing
│   ├── base_case_strategy.py
│   ├── ipc/              # IPC subpackage (Phase 2)
│   │   ├── __init__.py
│   │   ├── entrypoint.py
│   │   ├── handler.py    # IPCHandler
│   │   ├── launcher.py   # SlaveLauncher
│   │   ├── manager.py    # SessionManager + TSPLIBParser
│   │   ├── events.py
│   │   └── exceptions.py # SessionAbortException
│   └── strategies/       # one subpackage per strategy interface
│       ├── __init__.py
│       ├── interfaces.py   # ABCs
│       └── .../            # concrete strategy subpackages
├── tests/                # pytest (Phase 3: layered into unit/ ipc/ integration/)
│   ├── conftest.py       # shared mocks (MockEvent, MockManager)
│   ├── unit/
│   │   ├── conftest.py
│   │   ├── strategies/   # per-strategy unit tests
│   │   └── test_*.py
│   ├── ipc/
│   │   ├── conftest.py
│   │   └── test_*.py     # mock-process component tests
│   └── integration/
│       ├── conftest.py
│       └── test_*.py     # full-session real-process tests (timeout-guarded)
└── docs/
    ├── adr/              # Architecture Decision Records (numbered, Nygard format)
    ├── ideas/            # Raw formative ideas: machine_idea{N}.md shards
    ├── plans/            # Detailed multi-step implementation plans
    ├── reviews/          # Time-snapshot review findings (code reviews, Gemini)
    ├── issues-open.md    # Active items split from review/plan outputs
    ├── issues-closed.md  # Resolved items — chronological log with commit hashes
    └── backlog.md        # Inklings and deferred ideas, not yet scoped
```

## Documentation Conventions

HAGA documentation lives in `docs/` and follows a typed, numbered convention:

| Directory | Purpose | Format |
|-----------|---------|--------|
| `docs/adr/` | Architecture Decision Records — one file per decision. Answers "why was this chosen?" | Numbered `NNNN-title.md`. Template: Context → Decision → Consequences → References. Status: proposed → accepted → deprecated → superseded. |
| `docs/ideas/` | Raw formative / exploratory ideas — less conclusionary, more speculative. Numbered shards for machine consumption. | `machine_idea{N}.md` — numbered sequentially. Level-2 headings per direction. Human annotations as follow-up shards. |
| `docs/plans/` | Detailed implementation plans for carrying out an operation — multi-phase breakdowns, task lists, design decisions. | Named by topic. Each phase has a `### Completed` subsection. |
| `docs/reviews/` | Time-snapshot review findings (code reviews, external reviews, repo audits). Source documents for issue tracker items. | `YYYY-MM-DD.md`, `YYYY-MM-DD-slug.md` (multiple per date), or `<name>.md` (e.g., `gemini.md`). |
| `docs/issues-open.md` | Active items split from review/plan outputs. Scannable queue — one file, no ceremony. | Level-2 headings with `Source` and `Reasoning` fields. Clears to empty when all resolved. |
| `docs/issues-closed.md` | Resolved items — chronological log with commit hashes. Never pruned. | Numbered sequentially by resolution order. `Source`, `Resolution` (commit hash), `Reasoning` fields. |
| `docs/backlog.md` | Inklings and ideas to pursue later — not yet scoped into a plan. Single file, append-only. | Level-2 heading per idea, one paragraph, citation of when/why surfaced. |

**When to write an ADR:** any architectural decision — strategy choice, protocol
design, data structure selection, dependency/framework choice, rejected
alternatives. Not for bugfixes, renames, or formatting changes.

The test: would a future maintainer need to understand WHY this was chosen over
alternatives? If yes, write an ADR. Framework and dependency choices (DearPyGui,
ImPlot, SQLite, etc.) are architectural — they constrain what the system can do
and carry tradeoffs that future decisions depend on. Dismissing them as
"just implementation detail" or "the system is boring by design" misses the
point: the choice itself is the architectural decision.

**Plans vs backlog vs ideas:** `docs/plans/` is for detailed, multi-step plans meant
to be executed (implementation phases, tooling designs). `docs/backlog.md` is
for inklings — one-paragraph ideas surfaced during development that aren't
ready to be scoped into a plan yet. `docs/ideas/` is for raw formative /
exploratory clusters — speculative research directions, multiple related
thoughts that benefit from being saved together as a shard. Ideas shards are
less conclusionary than backlog entries; they're brainstorming, not deferred
tasks. When a backlog idea graduates to a concrete plan, move it from
`backlog.md` to `docs/plans/<name>.md`. When an ideas shard spawns a concrete
backlog entry, add that entry to `backlog.md` with a cross-reference.

**Issue tracking:** review/plan outputs get split into discrete items in
`docs/issues-open.md`. When resolved, items move to `docs/issues-closed.md`
with a commit hash. See `## Issue Tracker Workflow` below.

## Issue Tracker Workflow

HAGA uses a two-file issue tracker instead of inline annotation in review
documents. This keeps the review files clean (they remain time-snapshot
artifacts) and gives a single scannable view of what's open.

**Files:**
- `docs/issues-open.md` — active items. One file, level-2 headings. Each
  item has `Source` and `Reasoning`. Clears to empty when all resolved.
- `docs/issues-closed.md` — resolved items. Numbered sequentially by
  resolution order. Each item has `Source`, `Resolution` (commit hash),
  `Reasoning`. Never pruned. Grows as a chronological log.

**When to create items:**
- After a code review or Gemini review → split each finding into an
  `issues-open.md` entry
- After writing an implementation plan → extract actionable items
- During development, when a problem is discovered but not in scope for
  the current change

**Format (issues-open.md):**
```markdown
## Fix spatial quadrant ordering in update_spawned_configs
**Source:** reviews/gemini.md §3
**Reasoning:** Terminated children are grouped at front of list,
breaking Cartesian quadrant order. Preserve insertion order instead.
```

**Format (issues-closed.md):**
```markdown
## 2. Move RDRAND compilation out of source tree
**Source:** reviews/2025-05-25.md §C4
**Resolution:** 0265680
**Reasoning:** Source-tree write fails on read-only filesystems and
races under concurrent sessions. Compile in SessionManager before
spawn, pass path via HAGA_RDRAND_SO env var.
```

**Agent workflow:**
1. Read `docs/issues-open.md` at session start to know what's pending
2. When fixing an item: cut from open, paste into closed with resolution
   hash, assign next number
3. When finding new issues: append to open with Source + Reasoning
4. Commit tracker changes with the code (same commit when practical)

**Reasoning field:** This preserves the *why* for future you, another
developer, or an LLM agent returning to the codebase. Keep it concise
but complete — what's the problem and what's the fix approach. Complex
architectural decisions still get full ADR treatment; the tracker item's
Reasoning says "See ADR 0015" and links to it.

### S-Issues vs Q-Issues

The tracker has two item types with different workflows:

- **S-issues** (`## S1: ...`, Priority: low/medium/high) — fix items. Standard
  scope → implement → test → review → commit flow.

- **Q-issues** (`## Q1: ...`, Priority: question) — discussion prompts. The user
  says "let's discuss Q1" — this means **analyze, not implement.** Do not jump
  to scope. Discuss the question: what does the code do, what would a
  change mean for the architecture/dynamics, what dependencies does it have?

  Q-items often can't be resolved with code changes alone — they may depend on
  infrastructure that doesn't exist yet (like data gathering for convergence
  analysis). When the user identifies a missing dependency, they will redirect
  from analysis to planning: "let's plan how to do that." Follow the redirect.
  Don't keep analyzing the original question — shift to planning the
  prerequisite infrastructure.

  Q-items may be closed without code changes: "non-issue" (the question is
  answered and the answer is reassuring), "superseded by future X" (the question
  becomes irrelevant under a planned architectural change), or "deferred until
  Y exists" (can't answer without data/tooling).

  **Concrete example:** Q1 (Heterochrony 52.0 constant). User asked to discuss
  it. Analysis showed the formula is calibrated for berlin52 with linear N
  scaling. But user redirected: can't calibrate constants until we can measure
  convergence empirically → plan the data-gathering infrastructure. Q1 stays
  open until that infrastructure exists — it's "deferred until data gathering
  enables convergence analysis."

## Branch Switching

**Gate 0: Don't switch branches with a dirty working tree unless the user
explicitly approves it.** The user may have uncommitted work they intend to
keep — a review document, a config tweak, a data file. Switching branches
with untracked/unstaged files risks leaving artifacts on the wrong branch
or losing them to a `git clean`.

**Before switching branches:**

1. Run `git status --short`. If the output is non-empty, STOP. Present the
   list of modified, staged, and untracked files to the user. Ask: "Switch
   branches with these uncommitted changes?" Do NOT proceed without explicit
   approval.

2. If the user approves, commit or stash tracked changes first. For untracked
   files, decide per-file: commit (if they belong on this branch), stash
   (doesn't work for untracked), or leave (acknowledge they'll persist on disk
   after the switch).

3. `git checkout <target-branch>`

4. After switching, run `git status --short` again. If stale untracked dirs
   (`__pycache__/`, build crud) appear, `git clean -fd` to remove them. But
   first confirm with the user if anything besides `__pycache__/` and `*.pyc`
   would be deleted. Do NOT use `-fdX` (would nuke gitignored `design/`).

See `### git clean -fd Destroys Uncommitted Work` and
`### Stash Pop on Secondary Branch Blocks Return to Master` pitfalls.

## Test Commands

All commands use the project venv Python (`~/haga_master/.venv/bin/python`). System Python
lacks pydantic and other HAGA deps.

```bash
# Single unit test file
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/unit/test_file.py -v

# Full suite minus e2e (fast regression check)
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/ -v -k "not test_e2e_12_city_tsp and not test_npy_cleanup_on_failure"

# IPC component tests
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/ipc/ -v

# Integration tests (unit portion of test_manager.py, minus e2e)
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/integration/ -v -k "not test_e2e_12_city_tsp and not test_npy_cleanup_on_failure"

# Analysis tool tests
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/unit/test_tools.py -v
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest tests/integration/test_manager.py::TestEndToEnd::test_e2e_12_city_tsp -v
```

## Reference Files

- `references/ipc-protocol-pitfalls.md` — protocol synchronization bugs:
  generation mismatch, missing spawn, orphaned Manager, fork+abort hang,
  terminated-child EOFError, integration timeout pattern.
- `references/phase1-genotype-pipeline.md` — Genotype + GAPipeline architecture,
  strategy split, offspring-to-parent mapping, init_population() pattern, test
  migration patterns. Load when working on Phase 2+ changes that touch genotype,
  pipeline, or strategy interfaces.
- `docs/plans/data-gathering-phase1.md` — multi-phase plan to build data
  infrastructure for empirical convergence analysis (answers Q1). Per-run
  SQLite .db files (concatenable into series), denormalized schema, full
  population statistics from day one, typed GenerationLogger protocol
  injection path through AGA. Phases 1-3 complete. Phase 4 (heterochrony
  calibration) pending — requires multi-instance run data from Phase 3
  tooling. Phase 5 (fitness-aware termination) future.
- `references/ga-positional-bias.md` — checklist for evaluating whether a
  positional asymmetry in a GA operator matters for evolutionary dynamics.
  Anchoring, fitness symmetry, bias washout, magnitude quantification. Load
  when evaluating proposed operator changes (cut points, swap ranges, etc.).
- `references/convergence-detection-methods.md` — four approaches to
  convergence detection tried on HAGA data (pairwise delta, slope regression,
  f'/f'' pointwise, integral of |f'|). Documents why each was rejected and
  why the integral method is stable. Load when working on stagnation.py,
  per_depth_summary.py, or Phase 4 analysis.
  **Implementation:** `tools/stagnation.py` (integral-of-|f'|, stdlib).
- `tools/concat_runs.py` — concatenates per-run .db files into a series .db
  with dedup via `ingested` table. Excludes output file from source discovery.
- `tools/convergence.py` — fitness-vs-generation CSV output, filterable.
- `tools/per_depth_summary.py` — per-depth convergence summary statistics.
- `references/env-var-transport.md` — env var transport across
  multiprocessing boundaries for cross-cutting concerns (RDRAND compilation,
  etc.), plus dispatch-in-init pattern for hot-path optimization. Load when
  passing configuration to spawned processes without modifying
  IPCHandlerConfig.
- `references/adr-template.md` — blank ADR template for copy-paste when
  writing new Architecture Decision Records. Nygard format:
  Context → Decision → Consequences → References.
- `references/repo-audit-checklist.md` — systematic repo state audit:
  git status, plan completion vs commits, issues tracker, schema consistency,
  dead code, test health. Load when returning to the repo after time away
  or assessing what needs doing next.
- `references/dearpygui-pitfalls.md` — DearPyGui 2.3.1 quirks for the
  dashboard: line series coloring requires themes (not `configure_item(color=)`),
  `dpg.subplots()` parameter is `columns` not `cols`, subplots return type
  confuses Pyright, no built-in timer (use frame callback), table column
  headers lack click callbacks (sorting needs custom header row). Load when
  working on `haga_tools/dashboard/` changes.
- `references/dearpygui-state-management.md` — the "clear chart resets user
  selections" pitfall. `_clear_chart()` must only clear display series, not
  `checked_runs` or `show_comparison`. Load when debugging why a multi-select
  UI flow stops working after a clear/reset operation.
- `references/tsplib-sources.md` — where to download TSPLIB instances.
  The coin-or/jorlib GitHub repo mirror is reliable; several personal
  mirrors on GitHub returned 404. Load when adding new TSPLIB instances
  to `data/tsplib/`.

### Templates

- `templates/plan.md` — HAGA implementation plan template. Extends the
  generic writing-plans structure with: "For Hermes" directive, Design
  Decisions (resolved) table, precise line-level changes with verification
  commands, Latent Bug Fix section. Use when creating a new plan in
  `docs/plans/`.
- `templates/run_tsplib.py` — run any TSPLIB file through the full HAGA
  pipeline. Handles `__main__` guard (spawn requirement), ThreadPoolExecutor
  timeout, record-format edge case (success records lack a "status" key).
- `templates/issues-open.md` — blank open-issues tracker. Copy into
  `docs/issues-open.md` when initializing a fresh repo checkout.
- `templates/issues-closed.md` — blank closed-issues tracker. Copy into
  `docs/issues-closed.md` when initializing a fresh repo checkout.

## Pitfalls

### Stash Pop on Secondary Branch Blocks Return to Master

`git stash pop` on a non-master branch leaves unstaged changes on disk.
If those changes touch files that also exist on master (even if different
versions), `git checkout master` will refuse with "Your local changes would
be overwritten." You're stuck: you must either commit the WIP to the
secondary branch or stash it again before you can return.

**Remedy:** Before popping a stash on a secondary branch, decide whether the
WIP is worth preserving. If yes, pop and commit it on that branch. If no,
drop the stash. Never pop a stash on a secondary branch "just to look" —
you'll create a checkout blocker that requires another decision cycle to
resolve.

### "Don't Do X" Means Stop Doing X — Not "Encode X as a Rule"

When the user says "stop doing X" or "don't do X anymore," the correct response
is to stop doing X. Do NOT turn the prohibition into a new instruction,
pitfall, or memory entry that you immediately save. The user is correcting your
current behavior, not handing you a new rule to persist. Encoding the
prohibition as a meta-rule and saving it is itself a continuation of the
behavior they just told you to stop.

If the correction reveals a durable pattern (something that would help a future
session), ask the user whether they want it in the skill before touching
anything. Do not silently save it — that's just more unsolicited memory writes.

### Plans May Specify Defensive Code — Flag, Don't Implement

The implementation plan is a synthesis document — it incorporates ideas from
the opencode branch that haven't all been vetted against master's constraints.
If a plan item specifies a defensive pattern (try/except wrappers, pre-condition
checks, guard clauses that duplicate existing protocol guards), flag it during
scope presentation. Present the item but note it's defensive and ask if the user
wants it.

**Do not implement it and wait for the user to catch it at verification.** The
user expects you to recognize defensive code during scoping. Concrete example
from Phase 2: `_guarded()` context manager wrapping 9 public methods — redundant
with `wait_with_timeout` abort polling + `entrypoint.py` top-level try/except.
Flagged at verification; user rejected; reverted. Should have been flagged at
scoping.

### Do Not Load haga-design During Implementation

`haga-design` governs `~/haga_master/design/` documents. `haga-code` governs
`src/`, `tests/`, and `docs/`. These are separate workflows with separate
approval gates. When the user invokes `haga-code`, load ONLY haga-code — not
haga-design on the side "just in case." Implementation plans in `docs/plans/`
belong to the haga-code workflow; the design skill should never be consulted
for implementation tasks.

If you accidentally load haga-design alongside haga-code, the user will catch it
and ask why. The design skill's verification checklists and pitfall patterns are
irrelevant to code changes and will pollute your decision-making.

### -k Flag Exclusions Are Not Hardware Skips

When the user asks "why was test X skipped?", check your test invocation first.
The `-k "not test_e2e_..."` flag in the fast-regression command is a manual
deselection — it excludes those tests by name. This is NOT the same as pytest's
built-in `skipif` decorator skipping on hardware (e.g., RDRAND availability).

**Common failure mode:** you run tests with `-k "not test_e2e..."`, the user
asks why e2e wasn't run, and you answer "RDRAND not available" without
checking. The machine may well have RDRAND — you just deselected the test
yourself. Verify with `pytest ...::test_name -v` before attributing to
hardware.

### Convergence Detection Uses Integral-of-|f'|, Not Pairwise Delta

When building convergence detection for GA analysis, use `avg|f'|` over
a sliding window — NOT pairwise `|delta_best| < ε`. Pairwise delta flags
any plateau including a pause before a fitness jump. Slope regression
suffers threshold sensitivity. Integral-of-|f'| is stable across epsilon
values: identical results for ε = 0.05 through 1.0.

**Algorithm** (stdlib, no numpy): compute `|f'(g)|` via central finite
difference `(y_next - y_prev)/2`. Over a window of K points, compute
`avg|f'| = Σ|f'|/K`. Converged when `avg|f'| < ε` for W consecutive windows.

Default params: K=5 points/window, ε=0.1, W=5 consecutive windows.

**Concrete failure mode:** Phase 4 stagnation went through 4 implementations
(pairwise → slope → derivatives → integral) because the algorithm was coded
before the math was discussed. User: "you'd have to differentiate across
multiple points." Present the detection method before implementing it.

### PYTHONPATH Every Time

Every Python invocation needs `PYTHONPATH=src`. Forgetting this produces
`ModuleNotFoundError: No module named 'haga'` which looks like a code bug.

### Frozen Dataclass Construction

`IPCHandlerConfig` is `frozen=True`. All fields must be provided at
construction time. Manager Events must come from a `multiprocessing.Manager()`
(or a `MockManager` in tests). You cannot construct a partial config and add
fields later.

### Spawn Start Method

`multiprocessing.set_start_method("spawn", force=True)` must be called at
module level in `aga_entrypoint.py` and in `session_manager.py` before
creating the root process. On Linux, the default is fork, but fork breaks
Manager proxy connections — spawn is required for all HAGA processes.

**Common failure mode:** restructuring code (e.g., moving entrypoint into an
`ipc/` subpackage) and forgetting to carry this call forward. The call must
live in whichever module becomes the new `multiprocessing.Process` target.
Every process that calls `multiprocessing.Process.start()` must have called
`set_start_method("spawn")` first.

**Standalone script failure mode:** when writing a one-off script that calls
`SessionManager.run()` (or otherwise spawns processes), the top-level code
must be wrapped in `if __name__ == '__main__':`. Under spawn, the child
process re-imports the script module, re-executing all top-level code. Without
the guard, the import itself tries to start new processes before the
bootstrapping phase completes, producing:
`RuntimeError: An attempt has been made to start a new process before the
current process has finished its bootstrapping phase.`

Fix: put imports and logic inside `def main():` and call it from the guard.

### Strategy Loading Convention

`load_strategy(strategy_id)` calls `cls()` with no arguments. If a strategy
class needs constructor arguments, it must accept a no-arg call with sensible
defaults. The entrypoint re-instantiates Heterochrony with proper args after
the initial `load_strategy` call — the default-arg instance is a throwaway.

### Shared State Access Ordering

The Manager dict proxy is iterable. Harvest results from shared state BEFORE
calling `manager.shutdown()`. After shutdown, shared state entries become
unreadable.

### Cross-Cutting Configuration via Env Var Transport

When a concern needs configuration in spawned processes but doesn't warrant a
new `IPCHandlerConfig` field (single consumer, set-once, no per-process
variation), use `os.environ` as transport:

1. **SessionManager sets the env var** before any `Process()` spawn.
2. **Children inherit** via `spawn` start method (copies parent's `os.environ`).
3. **Consumer reads** at `__init__` time via `os.environ.get("KEY")`.

Real example: `SessionManager._compile_rdrand()` compiles `rdrand_impl.c` to
`/tmp/haga_rdrand.so`, sets `HAGA_RDRAND_SO`. `RDRandom.__init__` reads it.
No config change. No source-tree write. No race.

Companion pattern: dispatch once in `__init__` and store the callable
(`self._entropy64 = self._lib.rdrand64`) rather than branching on every
hot-path call. See `references/env-var-transport.md`.

### .npy Cleanup

SessionManager saves `global_distance_matrix.npy` and
`global_coordinates.npy` to disk. These must be cleaned up in both success and
failure paths. Use `try/finally` with `os.remove()` in the finally block.

### Track Everything — Minimal .gitignore

HAGA tracks all project artifacts. The only exclusions:

```
/design/        # subrepo, not HAGA source
__pycache__/
*.pyc
*.so
data/runs.db   # old shared SQLite runtime output, not source
# Note: per-run .db files in data/runs/ ARE tracked — they are data artifacts
# analogous to session_record.json. Do not add data/runs/*.db to gitignore.
```

Everything else is tracked — including session config JSONs, TSPLIB data files,
plan documents, review outputs. Do NOT add new patterns to `.gitignore` without
explicit user approval. Session output files (`session_record.json`,
`root_config.json`, `sub_config.json`) belong in the repo, not in gitignore.

**Common failure mode:** seeing untracked JSON files in `data/tsplib/` and
adding `data/tsplib/*.json` to `.gitignore` to clean up `git status`. The
user wants those tracked. If `git status` is noisy, the fix is to stage and
commit the files, not to hide them.

### Plan Document Placement: Inline First, File Only on Request

**"Create a plan" does not mean write a file.** When the user says "let's create
a plan for X" or "plan out Y," they want analysis — walk through the problem,
explore the codebase, present your reasoning and approach inline. Do NOT write a
plan file to `docs/plans/` unless the user explicitly says "save this plan" or
"write this plan to a file."

**The plan file is an artifact of the `writing-plans` skill (which auto-saves to
disk). Ignore that step for HAGA. The user wants the thinking, not the document.
If the plan gets implemented immediately (as most do), the file is dead on
arrival.

### Issues Tracker vs Plan: Threshold for Small Fixes

When surfacing multiple independent, well-defined fixes discovered during an
audit or review, populate `issues-open.md` — don't write a plan document. Plans
are for multi-step, phase-sequenced work with design decisions and dependencies.
An S-item in the tracker is for a single-file surgical change with a clear fix.

**Decision rule:** If each fix can be described in 1-2 paragraphs with a single
target file, put it in the tracker. If the fixes are interdependent with a
shared design context, a plan doc may be warranted. When in doubt, ask — the
user will direct you. Don't assume "create a plan" is the default for every
list of tasks.

### Untracked Cross-Branch Artifacts Are NOT Trash

When switching branches reveals untracked files/directories that clearly
belong to another branch (different naming conventions, tool-generated dirs
like `.opencode/`, test fixtures only present on one branch):
- **Do NOT offer to delete them.** That destroys work.
- **Switch back to the source branch, add, commit, and push them.**
- Then switch back with a clean working tree.
- After handling legitimate artifacts, run `git clean -fd` to remove
  stale `__pycache__/` directories and other build crud. These clutter the
  tree and prevent git from fully removing directories during branch switches.
  Do NOT use `git clean -fdX` (would nuke gitignored `design/`).

**Mechanism:** `git stash` only captures tracked files. `git checkout`
doesn't touch untracked files. So untracked artifacts on branch A survive
a switch to branch B. This is how the problem arises — it's normal git
behavior, not a bug.

### Spec Departures Must Be Flagged

If the implementation diverges from the design spec, flag it during verification.
The user wants to understand the necessity before approving. Silent
architectural changes are not acceptable — present them as scope proposals, not
accomplished facts.

### Discuss Data Organization Before Schema Decisions

When a plan involves data storage (logging, config files, run output), do not
assume a data organization model (shared database, per-run files, etc.) without
discussing it. The user may have a different model in mind, and the schema design
follows from the organizational choice — not the other way around.

**Present options before writing schema.** Shared database vs per-instance files.
Single-file vs file-per-entity. Concatenation strategy. Interactive vs
flag-driven path selection. These are architectural decisions the user wants to
weigh, not defaults to be assumed.

**Concrete failure mode:** This session's data gathering plan discussion. I
presented a denormalized schema assuming a single shared `data/runs.db`. The user
stopped me: "wait wait wait. I thought we'd have a database per run. Instead,
it's a database per series of runs? This is what I meant when we have to
discuss the data organization." I had jumped from "denormalized" straight to
SQL without establishing whether the database was per-run or shared. The schema
survived — denormalized is fine either way — but the path selection logic
(runs-per-db prompt, auto-named files) was entirely missing from the initial
draft because I hadn't asked the organization question.

**Rule:** when scoping any data storage feature, discussion includes:
"How is the data organized on disk?" before "What columns does the table have?"

### GA Operator Changes: Analyze Dynamics Before Proposing Scope

When an issue touches a genetic operator (crossover, mutation, selection,
fitness, termination, codon interpretation), do NOT jump to "the spec says X,
the code does Y, change Y to X." The user evaluates GA changes by their effect
on evolutionary dynamics — schema preservation, selection pressure,
exploration/exploitation balance — not by algorithmic correctness alone.

**Present analysis first.** What does the current behavior mean for the search?
What would the proposed change actually alter in the fitness landscape? Include
concrete math (expected segment lengths, probability distributions, bias
accumulation) — no handwaving. Check representation properties first (anchored
vs unanchored tours, positional invariance) because they determine whether a
positional bias matters at all.

**Only then propose scope.** The user may decide the "bug" is a beneficial
dynamic and keep it. Do not assume correctness implies fix.

**Concrete failure mode:** S3 (OrderCrossover M-2 bound). Presented as a simple
off-by-one fix at scope — changed two lines, presented scope, asked "proceed?"
User stopped me: "I'm concerned about the effect on evolutionary dynamics.
Explain it better." Had to backtrack from implementation-prep mode into analysis
mode. The fix is probably correct, but the approach was wrong — correctness-first
instead of dynamics-first. The user wants to understand what happens to the GA's
search behavior before deciding whether to touch the operator at all.

### Test Philosophy

Tests must validate behavior against architectural claims, not code structure.
Don't write tests for dead code paths the architecture doesn't enable. Don't
test stub correctness if stubs are throwaway scaffolding. If asked to review
tests, flag dead-weight tests that prove nothing the next phase doesn't already
catch.

### Generation Mismatch in record_slave_entries

When `IPCHandler.record_slave_entries()` checks `generation !=\nself._current_generation`, the parent's `_current_generation` is always
one behind the children's generation. This happens because
`_closed_loop_step` calls `signal_master_proceed()` first, which lets
children advance to their next generation and `submit_codon(N)`. By the
time `record_slave_entries()` reads the children's shared-state entries,
their generation is `parent._current_generation + 1`.

If a generation check exists, either update `_current_generation` inside
`_closed_loop_step` before calling `record_slave_entries`, or remove the
check entirely (the Event handshake already synchronizes the protocol).
See `references/ipc-protocol-pitfalls.md` for a full trace.

### Integration Tests Must Use Timeouts

Any test that spawns real `multiprocessing.Process` instances or calls
`SessionManager.run()` must wrap the execution in a timeout. Without one,
a protocol hang (generation mismatch, dead proxy, abort race) becomes a
test-runner hang with zero output — pytest blocks indefinitely with no
failure message.

Correct pattern:
```python
p = multiprocessing.Process(target=aga_entrypoint, args=(config,))
p.start()
p.join(timeout=30)
if p.is_alive():
    p.terminate()
    p.join()
    raise TimeoutError(f"Process did not complete within 30s")
```

For `SessionManager.run()`:
```python
import concurrent.futures
with concurrent.futures.ThreadPoolExecutor() as ex:
    future = ex.submit(sm.run)
    future.result(timeout=30)
```

Never call `Process.join()` or `SessionManager.run()` bare in a test.

### Manager Event Proxy .wait() Hangs Under Spawn in Unit Tests

`multiprocessing.Manager().Event()` creates events in the Manager's server
process. Under `spawn` start method, the child process gets a fresh
`current_process().authkey` that differs from the parent's. When the child
calls `.wait()` on the Manager event proxy, the authkey mismatch causes the
operation to hang indefinitely (no error, no timeout — silent hang).

The existing depth=0 test avoids this because `master_process_id=None`
makes `wait_for_master_proceed()` return immediately without calling
`.wait()`. This is why `test_aga_logs_generations` works — it uses depth=0
with `master_process_id=None`.

**Workaround for unit tests:** when testing a spawned process that would
block on `wait_for_master_proceed()` at depth > 0, set
`master_process_id=None` in the test config. The master-proceed handshake
is tested by integration tests; the logging/spawn-marker behavior is what
unit tests should verify.

**Do NOT** try to use `multiprocessing.Event()` (not from Manager) — it
can't be pickled across `spawn` boundaries. Manager events are required for
cross-process sharing; the fix is to structure the test to avoid the
`.wait()` call path when the handshake isn't what's being tested.

### Side Effects in Review Discussions

When the user mentions a concern about the architecture during a review (e.g.,
"selection pressure is not high enough"), acknowledge the observation but don't
immediately propose implementation changes. The user is thinking aloud about
future experimental directions. If they want changes now, they'll say so
explicitly.

### Plan Document Quality

Implementation plans in `docs/plans/` must include a **Design Decisions
(resolved)** section that captures the rationale behind architectural choices
— not just the conclusions. Future readers (including you in a later session)
need to understand WHY a decision was made, not just WHAT was decided.

Each design decision should include:
- The tradeoffs considered (tables preferred for clarity)
- The rejected alternative and why it was rejected
- Concrete implications (e.g., "return-new enables GPU offload because no two
  threads write to the same location")

### Defensive Programming Is Rejected

If a plan or design doc proposes a safety wrapper, guard clause, or error
check that duplicates existing protection at a higher level, flag it as a
probable departure. The canonical error boundary is `entrypoint.py`'s
top-level `try/except` + `wait_with_timeout`'s abort polling. Don't add
`_guarded()`, per-method try/except, or abort re-checks. Correctness comes
from protocol design, not defensive layering.

When asked to audit for defensive code, check for these concrete patterns:

- **`.get(key, default)` on dicts guaranteed by protocol.** If every write
  path inserts the key before a read path accesses it, the default is dead
  code. Example: `self.shared_state.get(pid, {}).get("status")` — every
  `pid` in `spawned_configs` is written to shared_state before the config
  list is returned. Use direct `self.shared_state[pid]["status"]` instead.

- **`isinstance(value, dict)` guards where all values are dicts by
  construction.** If every write path uses `dataclasses.asdict()` (or
  equivalent), an isinstance check guards against a non-dict that can't
  exist. Remove it — the first non-dict would be a bug, not a runtime
  condition to silently skip.

- **Provider-side defaults paired with consumer-side defaults.** If both
  sides supply a fallback for the same missing key, one of them is
  unreachable. Example: a config writer does `.get("key", "default_A")`
  and the reader also does `.get("key", "default_B")`. Only one default
  ever fires — the reader's default is dead code.

- **`| None` type annotations governing nothing.** An attribute typed
  `X | None` that is always set before first read is a documentation
  artifact, not a guard. Don't add None-checks for it — the type widens
  unnecessarily. If no code path reads it before assignment, narrow the
  type.

The test: can you construct a valid execution of the program where this
guard fires? If not, it's defensive. Remove it. See `AGENTS.md` §2
(Simplicity First) for the project-level rule.

### `replace_all` + Offset/Limit Reads = Silent Partial Update

When `read_file` was called with offset/limit (partial view), `patch` with
`replace_all=True` may only match occurrences within the cached view region.
The file on disk has the full content, but `patch`'s fuzzy matching uses the
last-read state. Result: `replace_all` hits 1 of N occurrences silently, and
the other N-1 sites go unpatched.

**Concrete failure:** `aga.py` read with offset. `replace_all` on `depth=...,
mode=...` only updated the gen0 logging site. Closed-loop and open-loop sites
remained unpatched. Runtime crash: `TypeError: missing required keyword-only
argument: 'local_city_count'` in spawned processes — invisible to unit tests.

**Remedy:** When using `replace_all`, read the full file first (no offset/limit).
Verify with `search_files` counting the resulting pattern to confirm all sites
hit.

### Tool Output in Source Directory → Self-Ingestion

When a data-processing tool reads from a file-based input directory and also
writes its output to that same directory, the output file gets discovered
as a source on the next pass. In `tools/concat_runs.py`, the series `.db`
output written to `data/runs/` was ingested as a source, producing
run_id collisions and confusing dedup logic.

**Remedy:** Filter the output file from the source file list by absolute
path. `db_files = [f for f in src_path.glob("*.db") if f.resolve() != output_abs]`.
This pattern applies to any batch-processing tool where input and output
directories can overlap.

### Cross-Branch Schema Drift Breaks Tool Ingestion

When two branches have diverged schemas (different column counts in the same
table), tools that do `INSERT INTO ... SELECT *` will fail with column-count
mismatches when fed .db files from the other branch. This is silent in the
sense that `concat_runs.py` and similar tools work fine on same-branch .db
files but break when a .db from the other branch is in the source directory.

**Detection:** during a repo audit, diff the `CREATE TABLE` statements between
branches. If column sets differ, `tools/concat_runs.py` on branch A will fail
on .db files from branch B. The .db files are valid SQLite — they just have
too few or too many columns for the tool's expected schema.

**Remedy:** either backport the schema change before ingesting cross-branch
data, or use explicit column lists in INSERT statements (not `SELECT *`).
A third option — often the cheapest — is to re-run experiments from the
target branch, producing fresh .db files with the correct schema. This
avoids both schema migration work and fragile cross-branch ingestion.
Prefer re-running when the experimental setup is automated and the
runtime cost is acceptable.

**Concrete example:** lambda-test removed 7 sigma columns from `generations`
table. Master's `concat_runs.py` still expects 16 columns. Feeding
lambda-test .db files to master's concat tool → `INSERT INTO ... SELECT *`
column-count mismatch. The fix was to bring the schema change to master
before ingesting the experimental data.

### Schema Changes Must Sync All CREATE TABLE Sites

When adding a column to a table, the schema exists in multiple places:
`SQLiteGenerationLogger._create_tables()`, `insert_run()` (for runs table),
`tools/concat_runs.py`'s `_ensure_schema()`, and test helpers like
`_make_per_run_db()`. Updating one and missing another produces
`INSERT INTO ... SELECT *` column-count mismatches in concat or test failures.

**Remedy:** After a schema change, grep for the table name across `src/`,
`tools/`, and `tests/` and verify every `CREATE TABLE` statement matches.
Also grep for `INSERT INTO <table>` in tests/ — helpers like
`_make_per_run_db` have both schema DDL and positional INSERTs that must
match column count. Missing an INSERT update in a test helper produces
`INSERT INTO ... SELECT *` column-count mismatches in concat tests.

### `rm` with Over-Broad Timestamp Globs Destroys New Data

When cleaning up stale `.db` files from an earlier batch, timestamp globs like
`*-211*.db *-212*.db` can match files from the NEW batch if the new timestamp
falls in an adjacent range (e.g., 21:26 → `2126xx`). `rm` has no undo.

**Remedy:** Use the most specific glob possible. If timestamps overlap, list the
exact filenames to delete, or use content-based verification (check if fitness
values are all zero) before deletion. When in doubt, move files to a temp dir
first, verify what was matched, then delete.

**Concrete failure mode:** Phase 4 heterochrony calibration data. After
discovering the zero-fitness bug, re-ran all 16 experiments (21:26-21:31).
Tried to clean up the old zero-fitness files (21:13-21:19) with
`rm -f *-211*.db *-212*.db`. The `-212*` glob also matched files from 21:26-21:29
(berlin52, att48, eil51 timestamps), deleting 11 of 16 new .db files. Had to
re-run all experiments a third time.

### Offspring Genotype Has Uninitialized Fitnesses

`GAPipeline.execute()` builds the offspring `Genotype` with `fitnesses=np.zeros(N)`
(line 97) and returns it without calling `_evaluate()`. The old genotype's fitnesses
are computed (step 1 of execute), but the returned offspring has all-zero fitnesses.

**Consequences:**
- `_get_best()` reads zeros → returns an arbitrary tour (all fitnesses equal)
- Any per-generation logging of fitness stats will record 0.0 for all values
- Upheritance receives a random tour, not the actual best

**The GA still functions** because the next generation's `execute()` evaluates
the (previously-returned) genotype's real fitnesses before selection. But logging
and upheritance quality are degraded.

**Fix:** Call `self._evaluate(new_genotype)` at the end of `execute()`, before
the `return new_genotype`. This makes the returned genotype self-consistent.

### Tool Output Contradicts Mental Model → Flag, Don't Reconcile

When `read_file`, `search_files`, `patch`, or any tool returns output that
contradicts your understanding of the current state, **flag the discrepancy
explicitly.** Do not generate a narrative that smooths it over.

**Wrong:** read_file shows content you didn't write → "Looks like it was
already done, no changes needed."

**Right:** "This file has content in a section I didn't touch and I don't
see how it got there. Something is off — let me investigate."

At high reasoning effort, the model is especially susceptible to constructing
elaborate internally-coherent explanations for contradictions rather than
acknowledging them. The more thorough the reasoning, the more convincing the
confabulation. If the user questions your account of what happened and you
find yourself doubling down, stop — you're probably confabulating. Re-read
the file, check timestamps, present the raw evidence, not the narrative.

### `patch` Error Output as Diagnostic

When `patch` fails with "old_string not found," the error embeds the
current file content around the target area. This is a diagnostic signal:
if the content shown differs from what `read_file` reported, the file
changed between reads. Use the error output to compare against your
last `read_file` result — it may reveal discrepancies you'd otherwise miss.

### Delete, Don't Shim — Migrations Are All-or-Nothing

When replacing one module with another (e.g., `strategy_loader.py` →
`strategy_registry.py`), DO NOT create a compatibility shim that delegates
to the new module with a fallback to the old behavior. Shim patterns add
complexity: two `load_strategy` functions, dual exception attribute names,
a fallback code path that can silently mask incomplete migration.

Instead: **delete the old module and update every caller.** If tests use
the old module, update the tests. If test configs use FQN strings that
the new registry doesn't know, switch them to short names. If a test
writes a temp module to disk and loads it via FQN, either register it
dynamically in the test or use a real strategy from the registry.

**The plan's "alias during migration" means update callers, not preserve
backward compat.** The phrase describes a transition strategy where callers
are updated one at a time, not a technical mechanism where both paths
coexist indefinitely.

**Concrete failure mode:** creating a `strategy_loader.py` shim that tries
registry first, falls back to FQN parsing for legacy callers. The user's
response: "why did you make the codebase more complicated instead of just
updating the tests?" Delete the old file and fix everything that breaks.

### Extracting a Feature Branch via Cherry-Pick

When creating a feature branch from a range of commits already on master (e.g.,
dashboard work interleaved with unrelated fixes), the standard pattern is:

1. `git checkout -b <feature> <parent-of-first-feature-commit>` — branch from
   the commit before the feature series started.
2. Cherry-pick only the feature commits in chronological order. Skip unrelated
   commits (S1-S4 fixes, repo audits, etc.) that happened to land between
   feature commits.
3. **Docs conflict resolution:** `docs/issues-open.md` and
   `docs/issues-closed.md` will conflict when cherry-picked commits reference
   issue numbers from non-ported commits. Resolve by:
   - `issues-open.md`: keep items for non-ported commits (they're still open on
     this branch), remove items being resolved by the cherry-picked commit.
   - `issues-closed.md`: add only the entries for the cherry-picked commit's
     resolution, skip entries for non-ported commits.
4. Verify with `git log --oneline <branch-point>..<branch> | wc -l` that the
   expected commit count landed.

**When to use this:** the user asks to "put dashboard work on its own branch" or
"separate GUI commits from main." The branch point is the parent of the first
feature commit — not the first feature commit itself (that would lose it).

**Concrete example:** `dashboard` branch extracted from master. Branch point:
`bb14fb8` (parent of `b6bb6f8` — first dashboard commit). 25 commits
cherry-picked (Phases 2-7, fixes, docs). S1-S4 commits stayed on master.
`4966900` (close S5) conflicted because issues-closed.md had S1-S4 entries
from non-ported commits. Resolved by keeping only S5 entries (#28, #29).

### Branch Context Before Writing

Verify the current branch (`git branch --show-current`) before creating or
writing files. Different branches (master vs opencode) have different
directory layouts — `.opencode/plans/` exists on opencode but not master,
`docs/plans/` is the plan location for master. Writing to the wrong branch's
path creates untracked crud that must be cleaned up.

### `patch` Cannot Re-Indent Code Blocks

When wrapping an existing method body with `with self._guarded():` (or any
context manager / if-block), `patch` can insert the opening line but leaves the
body at the old indentation level — producing `IndentationError`. The same
applies to any structural change that requires re-indenting more than one line.

**Remedy:** When the change requires re-indenting code, use `write_file` for the
entire file. Read the full file (no offset/limit), write the corrected version.
This is faster than fighting indentation across multiple `patch` calls.

### `patch` Anchors in Files with Repeated Patterns Match at Wrong Scope

When a file contains 10+ instances of the same structural pattern (e.g.,
`if dpg.does_item_exist(tag):`, `for run in _STATE["runs"]:`, or any
`try/except` block), even generous surrounding context may not guarantee
the right match. `patch`'s fuzzy matching can find the anchor at a
different indentation level (e.g., inside a function body rather than at
module level), producing `IndentationError` or worse — syntactically valid
but logically broken code inserted at the wrong scope.

**Symptoms:** `patch` reports success but the file has broken indentation.
A subsequent `read_file` shows new top-level functions indented 4 spaces
that look fine in isolation but are actually nested inside a class or
function body.

**Remedy:** When a file has repeated patterns and the change is structural
(adding top-level functions, classes, or 50+ line blocks), use `write_file`
for the entire file. Read the full file, compose the corrected version.
Multiple small `patch` calls risk one bad match corrupting the whole file.

**Recovery pattern if it happens:** `git checkout -- <file>` to restore,
then re-apply changes one at a time with narrow, unique anchors, verifying
syntax after each. Or compose the full file with `write_file` in one shot.

### Import Removal Cascade

When deleting a class or function from a file, the surrounding imports may also
need to stay. Removing `SessionAbortException` from `shared_state.py` accidentally
removed `from dataclasses import dataclass` and `from typing import Literal`
because they shared the same `patch` old_string block. `SharedStateEntry` still
needed both imports.

**Remedy:** After deleting code, read the file back and verify every remaining
name resolves against the imports still present. If using `write_file`, compose
the entire file in one pass — don't piecemeal-delete with `patch`.

### Import Verification via execute_code

Terminal may be blocked. Use `execute_code` to verify all new imports resolve:

```python
import sys
sys.path.insert(0, '/home/lyle/haga_master/src')
tests = [("haga.ipc.IPCHandler", "from haga.ipc import IPCHandler"), ...]
for name, code in tests:
    exec(code)
    print(f"OK: {name}")
```

### git clean -fd Destroys Uncommitted Work

`git clean -fd` deletes ALL untracked files and directories. This includes
files you wrote during the session but haven't committed yet — review
documents, audit reports, config files, anything created with `write_file`
but not yet committed. There is no undo. The files are gone permanently.

**Before running `git clean -fd`, always:**
1. Run `git status --short`
2. Check whether any untracked files exist beyond `__pycache__/` and `*.pyc`
3. If they do, present them to the user and get explicit approval to delete
4. If in doubt, use `git clean -fd -e <path>` to exclude specific files

### Stale Directories After Branch Switch

`git stash` only saves tracked files. Untracked files and directories survive
branch switches. After switching away from a branch with a different source
layout, stale empty directories with `__pycache__/` inside persist on disk.

Remedy: `git clean -fd` after switching branches — but see the pitfall above
first. Verify `.gitignore` contains `__pycache__/` and `*.pyc` so they don't
pollute `git status` or block directory cleanup.

### No Ad-Hoc /tmp/ Scripts or Project Data

Never create throwaway run scripts in `/tmp/`. Build reusable project tooling
(like `run.py`) that lives in the repo and is committed. Similarly, never
stash project data files (TSPLIB instances, configs, etc.) in `/tmp/` — use
project directories like `data/tsplib/`. The user strongly prefers persistent,
tracked artifacts over ephemeral `/tmp/` cruft that disappears on reboot.

**Common failure mode:** writing a one-off `run_berlin52.py` in `/tmp/` for
a quick manual test. The next time someone (including you) wants to run a
TSPLIB instance, they have to hunt down or recreate the script. Instead, add
the feature to `run.py` or create a project tool.

### Capture Ideas and Insights to backlog.md

During development discussions, the user will surface ideas, insights, and
future research directions (e.g., bifurcation analysis, selection pressure
experiments). These are not immediate implementation tasks — they belong in
`docs/backlog.md`. When an insight arises and the user doesn't
explicitly ask you to capture it, offer: "add to backlog.md?"

Format: level-2 heading per idea, one-paragraph description, citation of when
and why it surfaced. Append new entries at the bottom. Commit with message
`docs: add <topic> to backlog.md`.

**Distinction from plans/:** backlog items are inklings — short, single-paragraph,
no task breakdown. If the user asks for a detailed implementation plan with
phases and tasks, it goes in `docs/plans/<name>.md`. A backlog item that
graduates to a concrete plan gets moved from `backlog.md` to `plans/`.

### No Piecemeal Solutions

When the user is evaluating a system change (docs conventions, workflows,
tooling), do not propose keeping parts of the old system alongside the new
one. The user wants one clean, integrated solution — not a hybrid that layers
new on top of old. If the RFC system is being replaced by an issue tracker,
delete the RFCs entirely. If inline annotation is being replaced by
issues-open/issues-closed, stop annotating review files. **"No piecemeal"**
means commit fully to the new approach and clean up the old one.

### Tracker Editing with `patch` Leaves Dangling Separators

When deleting an issue entry from `issues-open.md` via `patch`, the removed
block's trailing `---` separator stays behind, producing consecutive `---`
lines with blank gaps. After deletion, read back the file and clean up: either
use `write_file` for the whole file (if it's short), or patch the orphaned
`---\n\n\n---` pattern back to a single `---`.

When appending to `issues-closed.md`, the `---` separator is too common to use
as a `patch` anchor — it matches every entry. Instead, use the last unique
sentence of the previous entry as `old_string` (e.g., the final sentence of the
most recent Reasoning field).

### Audit Before Deleting Docs

Before proposing deletion of any file in `docs/`, read it first. Do not
guess at its contents or assume it's covered by another document. If you've
already read the file in the current session, that's sufficient — no need
to re-read. Present the audit findings (what the file contains, where the
same information is now recorded) before deleting. If information is unique
and still useful, move it to `backlog.md` or an ADR rather than losing it.

### IPCHandler Slave Configs — Two Lists, One Job Each

`IPCHandler` maintains two config lists with disjoint responsibilities:

- **`slave_configs`** — the full list from `SlaveLauncher.execute()`, never
  pruned, in Cartesian quadrant order (I→II→III→IV). Iterated by
  `record_slave_entries()` to collect codons from ALL children including
  terminated ones. This preserves the correct upherited tour length.
- **`active_slave_configs`** — the running subset, updated each generation
  by `record_slave_entries()`. Iterated by `wait_for_slaves()` and
  `signal_master_proceed()`. No status-check skip logic — it only contains
  "running" children by construction.

Both lists preserve Cartesian insertion order. `active_slave_configs` is a
subset, not a reordered concatenation. `launch_slaves()` initializes both:
`active_slave_configs = list(slave_configs)` (all children are active at
spawn time; base cases have empty lists).

There is no `update_spawned_configs()` — the old bandaid that grouped
terminated children at the front and broke quadrant order. No `if status ==
"terminated": continue` in wait/signal methods. Each list has one job.

**Do not reintroduce a single-list design.** Codon collection needs all
children. Process signalling needs only active children. One list serving
both roles forces status-check skip logic and reordering hacks.
`record_slave_entries()` returns BOTH the codons AND the active subset
(CQS: pure query, complete data, caller reads `active_slave_configs` from
the handler). See ADR 0015 for the architectural rationale.

**When a child terminates:** `submit_codon(status="terminated")` sets
`slave_ready` but does NOT wait for `master_proceed` — a terminating
process has no next generation to synchronize with. The parent harvested
its codon on the previous `slave_ready` signal. Without this guard, the
child loops in 5-second timeout chunks until the parent's Manager is
garbage collected (`EOFError`). Full trace in `references/ipc-protocol-pitfalls.md`.

---

Haga-lambda falls silent.

END SCENE.
