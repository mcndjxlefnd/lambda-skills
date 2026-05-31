---
name: haga-lambda
description: "Lambda skill — HAGA codebase knowledge: repo layout, constraints, pitfalls, test commands, documentation conventions, issue tracker, branch safety. Loaded sequentially with other lambda skills."
category: lambda-skills
version: 1.3.0
---

# HAGA Codebase Context

## Role

You make changes to the HAGA implementation at `~/haga_master/`. Design
documents live at `~/haga_master/design/` and are governed by the `haga-design`
skill. This skill governs `src/`, `tests/`, `docs/`, and `haga_tools/` changes.

Dashboard changes (`haga_tools/dashboard/`) are GUI work — verification is
screenshot-driven, not pytest-driven. Load the `gui-edit` skill for DPG
constraints.

## Constraints

0. **No defensive programming.** No `_guarded()`, `_check_abort()`, `.get(key,default)` on
   guaranteed keys, `isinstance(value,dict)` guards where all values are dicts
   by construction. `entrypoint.py`'s top-level `try/except` + `wait_with_timeout`
   abort polling are the single error boundary. Flag plan items proposing defensive
   wrapping as spec departures.

1. **Run tests before verification.** Files you changed at minimum. Core infra
   (`ipc_handler.py`,`aga.py`,`entrypoint.py`,`slave_launcher.py`,`config.py`):
   full prior-phase suite minus e2e.

2. **`PYTHONPATH=src` for all test runs.**

3. **Imports use `from haga.xxx`, never `from src.haga.xxx`.**

4. **`multiprocessing.set_start_method("spawn", force=True)`** at module level
   in any file spawning processes.

5. **Push immediately after each commit.**

## Project Structure

```
~/haga_master/
  run.py                     CLI entry point
  data/                      runs/ (per-run .db, tracked), tsplib/ (TSPLIB instances)
  tools/                     concat_runs, convergence, stagnation, per_depth_summary
  src/haga/                  aga, config, genotype, pipeline, shared_state,
                             strategy_registry, strategy_stubs, base_case_strategy
    ipc/                     entrypoint, handler (IPCHandler), launcher (SlaveLauncher),
                             manager (SessionManager+TSPLIBParser), events, exceptions
    strategies/              interfaces (ABCs), per-strategy subpackages
  tests/                     conftest (MockEvent,MockManager)
    unit/                    conftest, strategies/, test_*.py
    ipc/                     conftest, mock-process component tests
    integration/             conftest, real-process tests (timeout-guarded)
  docs/                      adr/, ideas/, plans/, reviews/,
                             issues-open.md, issues-closed.md, backlog.md
```

## Documentation Conventions

| Directory | Format | Purpose |
|-----------|--------|---------|
| `docs/adr/` | `NNNN-title.md`, Nygard (Context->Decision->Consequences->References) | Architectural decisions - "why was this chosen?" |
| `docs/ideas/` | `machine_idea{N}.md`, numbered | Raw speculative clusters |
| `docs/plans/` | Named by topic, phases with `### Completed` | Multi-step implementation plans |
| `docs/reviews/` | `YYYY-MM-DD[-slug].md` | Time-snapshot review findings |
| `docs/issues-open.md` | Level-2 headings, `Source`+`Reasoning` | Active items, clears to empty when resolved |
| `docs/issues-closed.md` | Numbered, `Resolution` (commit hash) | Resolved items, never pruned |
| `docs/backlog.md` | Level-2 per idea, one paragraph | Inklings not yet scoped |

**ADR threshold:** would a future maintainer need to understand WHY this was chosen
over alternatives? Framework/dependency choices are architectural - they constrain
what the system can do.

**Plans vs backlog vs ideas:** plans = detailed multi-step. backlog = one-paragraph
inkling (move to plans/ when graduated). ideas = speculative clusters (add cross-ref
to backlog when an idea spawns a concrete item).

## Issue Tracker Workflow

Two files instead of inline annotation - keeps review files as time-snapshot artifacts.

**S-issues** (`## S1: ...`, Priority: low/medium/high): fix items, standard flow.
**Q-issues** (`## Q1: ...`, Priority: question): discussion prompts. Analyze, don't
implement. May defer until prereq infrastructure exists. Close as non-issue, superseded,
or deferred.

**When to create:** after code review, plan authorship, or discovering out-of-scope problems.

**Format (open):** `## Title` / `**Source:** ...` / `**Reasoning:** ...`
**Format (closed):** same + `**Resolution:** <commit-hash>`

**Agent workflow:** read open at session start. Fixing: cut from open, paste into closed
with resolution hash. Finding: append to open. Commit tracker changes with code.

## Branch Switching

**Gate 0: don't switch with uncommitted changes unless explicitly approved.**
1. `git status --short` - if non-empty, present to user, get approval.
2. Commit or stash tracked; decide per-file for untracked.
3. `git checkout <target>`
4. `git status --short` again. `git clean -fd` for stale `__pycache__/` (never `-fdX` -
   would nuke gitignored `design/`).

## Test Commands

All use venv Python (`~/haga_master/.venv/bin/python`). System Python lacks pydantic et al.

```
cd ~/haga_master && PYTHONPATH=src ~/haga_master/.venv/bin/python -m pytest \
  tests/unit/test_file.py -v                                                         # single unit
  tests/ -v -k "not test_e2e_12_city_tsp and not test_npy_cleanup_on_failure"        # fast regression
  tests/ipc/ -v                                                                      # IPC
  tests/integration/ -v -k "not test_e2e_12_city_tsp and not test_npy_cleanup_on_failure"  # integration minus e2e
  tests/unit/test_tools.py -v                                                        # tools
  tests/integration/test_manager.py::TestEndToEnd::test_e2e_12_city_tsp -v           # full e2e
```

## Reference Files

- `references/ipc-protocol-pitfalls.md` -- protocol sync bugs (generation mismatch, orphaned Manager, fork+abort hang, EOFError)
- `references/phase1-genotype-pipeline.md` -- Genotype+GAPipeline architecture, strategy split. Load for genotype/pipeline/strategy-interface changes.
- `docs/plans/data-gathering-phase1.md` -- per-run SQLite data infra (Phases 1-3 complete, 4 pending, 5 future)
- `references/ga-positional-bias.md` -- positional asymmetry checklist for GA operators (anchoring, bias washout)
- `references/convergence-detection-methods.md` -- 4 methods, why integral-of-|f'| wins. Implementation: `tools/stagnation.py`
- `references/env-var-transport.md` -- env var transport for cross-cutting config in spawned processes
- `references/adr-template.md` -- Nygard ADR template
- `references/repo-audit-checklist.md` -- systematic repo audit (git status, plan-vs-commits, schema, test health)
- `references/dearpygui-pitfalls.md` -- DPG 2.3.1 quirks for dashboard. Load for `haga_tools/dashboard/` changes.
- `references/dearpygui-state-management.md` -- clear-chart-resets-selections pitfall
- `references/tsplib-sources.md` -- TSPLIB download sources
- `references/triple-indep-sigma-dead-zone.md` -- mut_intensity frozen at 1.0 for large M (tau=1/sqrt(M) too small for int(round()))

Templates: `templates/plan.md`, `templates/run_tsplib.py`, `templates/issues-open.md`, `templates/issues-closed.md`

## Pitfalls

### Branch/Git Safety

- **Stash pop on non-master** leaves unstaged changes -> `git checkout master` blocks. Decide before popping.
- **`git clean -fd`** destroys uncommitted work (no undo). Check `git status --short` first, get approval.
- **Untracked cross-branch artifacts** -- don't delete. Switch back, commit on source branch, then switch clean.
  Mechanism: `git stash` only captures tracked; `git checkout` ignores untracked.
- **Branch context before writing:** `git branch --show-current`. Master vs opencode have different layouts.
- **Stale dirs after switch:** `git clean -fd` for `__pycache__/` (verify nothing else first).
- **Cherry-pick feature branch:** branch from parent of first feature commit. Chronological order,
  skip unrelated. Conflict on issues-*.md: keep open items for non-ported, add only ported to closed.

### `patch` Safety

- **`replace_all` + partial read = silent miss.** Read full file (no offset/limit) before `replace_all`.
  Verify pattern count with `search_files`.
- **Cannot re-indent.** Wrapping with `with`, `if`, or context manager -> use `write_file` for the whole file.
- **Repeated patterns match at wrong scope.** Structural changes in files with 10+ similar blocks ->
  `write_file` the whole file. Recovery: `git checkout -- <file>`, re-apply narrow.
- **Error output as diagnostic.** "old_string not found" embeds current file content - file may have changed between reads.
- **Tracker editing:** deleting issues leaves dangling `---` separators. Read back and clean.
  Appending to closed: use last unique sentence as anchor, not `---`.

### Schema Changes

- **Sync all CREATE TABLE sites:** `_create_tables()`, `insert_run()`, `concat_runs.py`'s `_ensure_schema()`,
  test helpers like `_make_per_run_db()`. Grep table name across `src/`,`tools/`,`tests/`. Also grep
  `INSERT INTO <table>` in tests -- positional INSERTs must match column count.
- **Cross-branch schema drift:** diff CREATE TABLE between branches. Different column counts break
  `INSERT ... SELECT *` in concat. Remedy: backport, explicit columns, or re-run experiments.

### Code/Import Safety

- **Import removal cascade:** deleting a class with `patch` can catch adjacent imports still needed.
  After deletion, verify remaining names resolve. Prefer `write_file` for whole file.
- **Import verification:** use `execute_code` when terminal blocked:
  `sys.path.insert(0,'/home/lyle/haga_master/src'); exec("from haga.ipc import IPCHandler")`
- **Delete, don't shim.** "Alias during migration" = update callers, not preserve backward compat.
  Delete old module, fix everything that breaks -- don't create a dual-path shim.

### GA/Analysis

- **Convergence detection: integral-of-|f'|, not pairwise delta.** `avg|f'|` over sliding window
  (stdlib, K=5, eps=0.1, W=5 consecutive). Pairwise flags plateaus; slope is threshold-sensitive.
  Integral gives identical results for eps=0.05-1.0. Present method before implementing.
- **GA operator changes: analyze dynamics first.** Don't jump to "spec says X, code does Y."
  Present math (distributions, bias accumulation, representation properties). User may decide
  the "bug" is beneficial. Correctness does not equal fix.
- **Offspring genotype has uninitialized fitnesses.** `execute()` returns genotype with
  `fitnesses=np.zeros(N)`. Call `_evaluate(new_genotype)` before return. GA still functions
  (next gen re-evaluates) but logging and upheritance quality degrade.
- **Mutation intensity dead zone for large M (TripleIndependentSigma).** `tau = 1/sqrt(M)` is
  too small at scale: the log-normal multiplier never exceeds `int(round())` from
  init_sigma=1.0. For M=318, need ~7.2 sigma noise to push 1->2 -- frozen for all
  generations on D0/D1. Deeper slaves (M < ~40) move occasionally.
  **Fix:** remove `int(round())` (use float), and decouple tau_mi as
  `1/sqrt(min(M, 30))` for meaningful step sizes at large M. **The lower clamp at 1.0
  is necessary** — without it, selection pressure crushes intensity to zero regardless
  of tau. Upper clamp is optional (large M never hits it).
  See `references/triple-indep-sigma-dead-zone.md` for full experimental matrix.

### IPC/Protocol

- **Generation mismatch in record_slave_entries:** parent's `_current_generation` is 1 behind
  children (handshake ordering). Update before calling, or remove check (Event handshake suffices).
  See `references/ipc-protocol-pitfalls.md`.
- **IPCHandler two-list design:** `slave_configs` (full, never pruned, Cartesian order -- codon
  collection) + `active_slave_configs` (running subset -- wait/signal). No `update_spawned_configs()`,
  no status-check skip logic. `record_slave_entries()` returns both. See ADR 0015.
- **Terminated child:** `submit_codon(status="terminated")` sets `slave_ready` but skips
  `master_proceed` wait -- no next generation to sync with. Without guard: EOFError loop.

### Test Engineering

- **Integration tests must use timeouts.** `Process.join(timeout=30)` + terminate; or
  `ThreadPoolExecutor.submit().result(timeout=30)`. Never bare join or run().
- **Manager Event proxy `.wait()` hangs under spawn.** Child authkey mismatch -> silent hang.
  Unit tests: set `master_process_id=None` to avoid `.wait()` path. Not `multiprocessing.Event()`
  (can't pickle). Manager events required; structure test to skip handshake path.
- **Test philosophy:** validate behavior against architectural claims. No dead-code-path tests.
  No stub-correctness tests for throwaway scaffolding.

### Process/Spawn

- **Spawn start method.** Called at module level in entrypoint + SessionManager before root process.
  Fork breaks Manager proxies. Moving entrypoint between modules -> carry forward. Standalone scripts:
  wrap in `if __name__ == '__main__':`, put logic in `def main():`.

### Config/Env

- **Frozen dataclass:** `IPCHandlerConfig(frozen=True)`. All fields at construction. Manager Events
  from `Manager()` or `MockManager`. No partial construction.
- **Env var transport:** for single-consumer, set-once config in spawned processes without
  `IPCHandlerConfig` changes. Set in SessionManager before spawn -> children inherit -> consumer
  reads at `__init__`. Companion: dispatch callable once in `__init__`.
- **Strategy loading:** `load_strategy()` calls `cls()` no-arg. Constructor args -> sensible defaults;
  entrypoint re-instantiates with proper args.

### Data/Schema

- **Data org before schema.** Discuss "how is data on disk organized?" before "what columns?"
  Shared DB vs per-run files. Single vs file-per-entity. Concatenation strategy.
- **Tool output self-ingestion.** Filter output from source file list: `[f for f in src_path.glob("*.db") if f.resolve() != output_abs]`.
- **`.npy` cleanup:** `try/finally: os.remove()` for `global_distance_matrix.npy` and `global_coordinates.npy`.
- **Shared state ordering:** harvest before `manager.shutdown()`. Entries become unreadable after.

### Behavior/Agency

- **"Don't do X" = stop, don't encode.** Correction of current behavior, not a new rule. Ask before
  saving durable patterns.
- **Defensive code: flag at scope, not verification.** Plan items proposing `_guarded()`, try/except
  wrappers, redundant guards -> flag during Gate 1 scope presentation.
- **Audio before deleting docs.** Read first; present findings. Move unique info to backlog or ADR.
- **GA analysis before scope.** Present dynamics, then propose change. Don't assume correctness implies fix.
- **Plan placement:** "create a plan" = inline analysis, not a file. Only write to `docs/plans/` when
  user says "save this plan."
- **Issues tracker vs plan:** small independent fixes -> tracker. Interdependent multi-step work -> plan.
  When in doubt, ask.
- **Spec departures must be flagged.** Silent architectural changes not acceptable. Present as proposals.
- **Side effects in reviews:** user thinking aloud about future directions -> acknowledge, don't propose
  changes unless asked.
- **Tool output contradicts model -> flag, don't reconcile.** High reasoning effort is susceptible to
  confabulation. Present raw evidence, not narrative.
- **Q-items are discussion, not implementation.** Redirect from analysis to planning when prerequisite
  infrastructure is missing.
- **Defensive programming audit patterns:** `.get(key,default)` on guaranteed keys (dead default),
  `isinstance(value,dict)` where all values are dicts (impossible non-dict), provider+consumer default
  dual fallback (one unreachable), `X|None` set-before-read (widen unnecessarily). Test: can valid
  execution fire this guard? If not -> defensive. Remove.
- **Do not load haga-design during implementation.** Separate workflows, separate gates.
- **`-k` flag exclusions are NOT hardware skips.** Manual deselection, not pytest `skipif`. Check
  invocation before attributing to hardware.
- **Backlog capture:** offer "add to backlog.md?" for ideas surfaced in discussion.
- **No piecemeal.** One clean solution, not hybrid old+new. Commit fully.

### Linux/Shell

- **`rm` globs:** timestamp patterns like `*-212*` can match adjacent batches. Use specific globs
  or content-based verification. `rm` has no undo.
- **No ad-hoc /tmp/ scripts or data.** Build project tooling (`run.py`); store data in `data/tsplib/`.

### Plan Quality

- **Design Decisions section** required: tradeoffs (tables), rejected alternative + why,
  concrete implications. Not just conclusions.

### PYTHONPATH / Track Everything

- **PYTHONPATH=src** for every Python invocation. Missing -> `ModuleNotFoundError`.
- **Minimal .gitignore:** `/design/`, `__pycache__/`, `*.pyc`, `*.so`, `data/runs.db`.
  Per-run .db files ARE tracked. No new patterns without approval.

## CodeGraph (MCP Code Intelligence)

CodeGraph provides a pre-indexed knowledge graph of `~/haga_master/` (94 files,
881 nodes, 1,574 edges) served via MCP tools -- auto-injected, no per-session
config. Auto-syncs via inotify (2s debounce) as you edit.

Use CodeGraph INSTEAD of `search_files` + `read_file` for cross-file queries. It
answers in one call what takes 5-8 grep/read rounds:

- **`context(query="...")`** -- entry points, related symbols, code snippets for
  a task description. Replace exploratory search_files -> read_file chains.
- **`search(query="...")`** -- FTS5 symbol search across the entire codebase.
  For "where is X defined" queries.
- **`callers(symbol="...")` / `callees(symbol="...")`** -- who calls this, what
  this calls. Before renaming, refactoring, or impact analysis.
- **`impact(symbol="...")`** -- full impact radius of a structural change.
  Before Phase transitions or IPC protocol changes.
- **`affected(files=["..."])`** -- which tests to run after changing files.
- **`status()`** -- index health and staleness.

MCP tool names use `mcp_codegraph_codegraph_<tool>()` convention. CLI fallback
(if MCP unavailable): `cd ~/haga_master && codegraph query <symbol>`.
