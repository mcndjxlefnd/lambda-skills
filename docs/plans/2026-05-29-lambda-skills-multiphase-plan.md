# lambda-skills Implementation Plan — Multi-Phase

> **For Hermes:** Use the two-gate primitive + inter-phase adapter for each phase. Gates apply per-phase.

**Goal:** Build a typed lambda calculus for Hermes skill composition, grounded in Liu (2026) λA, that provides type safety, static lint, bounded termination, effect tracking, and pipeline algebra for co-loaded skills.

**Architecture:** The existing lambda-skills stack (primitive → adapter → orchestrator, co-loaded outer-first) already encodes a lambda calculus. Each phase extends the frontmatter type system, adds formal operators as new primitives, derives lint rules from operational semantics, and builds tooling. The phases are ordered by dependency: types first (everything builds on them), then effects, then operators, then lint, then algebra.

**Tech Stack:** SKILL.md frontmatter (YAML) for type/effect declarations, Python for lint tooling, Hermes co-loading for composition. No external DSL — types live in the skill files themselves.

---

## Phase 0: Formal Specification — λ-skills Calculus

**Status:** Plan-only. Delivers a spec document, not code.

### Task 0.1: Define the λ-skills Type System

Map λA's type system to skill composition:

```
τ ::= Str                          # base type: all skill I/O is text
    | τ₁ → τ₂                     # function type: a skill maps input to output
    | τ₁ × τ₂                     # product type: parallel skill output
    | ⟨lᵢ : τᵢ⟩                   # variant type: case/routing dispatch
    | {x:τ | P(x)}                # refinement type: guard predicate
```

**Str is the base type.** All skill inputs and outputs are text (markdown, JSON, code, natural language). This matches λA's design: "A simply typed calculus suffices because agent configurations operate at the string level."

**Key difference from λA:** In λA, types are checked by a compiler. In λ-skills, types are *declared* in frontmatter and *checked* at composition time. The checker validates that skill A's `output_type` matches skill B's `input_type` when they're composed.

### Task 0.2: Define the λ-skills Term Formers

Which of λA's 11 term formers map to skill composition:

| λA Term | λ-skills Equivalent | Primitive or Derivable |
|---|---|---|
| lam p θ | Skill LLM call — the body of a skill | Primitive (the skill *is* the term) |
| tool[f] | Skill tool invocation (terminal, web, file) | Primitive |
| fixₙ e | Bounded loop — retry/iterate with max steps | Primitive |
| mem e σ | Persistent state across skill invocations | Primitive |
| e₁ » e₂ | Sequential skill composition | Derivable (load order) |
| e₁ \| e₂ | Parallel skill execution | Derivable |
| case e of {lᵢ ⇒ eᵢ} | Routing skill — dispatch by label | Derivable |
| guard e P | Output validation with predicate | Derivable |
| e₁ ⊕ₚ e₂ | Probabilistic choice | Deferred (temp=0 only) |
| ⟨e₁, e₂⟩, πᵢ e | Pairing (for parallel results) | Derivable |
| if e₁ then e₂ else e₃ | Conditional | Derivable |

### Task 0.3: Define the Composition Semantics

Co-loading order `orchestrator, adapter, primitive` is equivalent to function application:
```
orchestrator(adapter(primitive(work)))
```

The reduction rules from λA map directly:
- **E-Comp:** `(e₁ » e₂) v → e₂ (e₁ v)` — adapter's output flows into orchestrator's continuation
- **E-Fix-Step:** `fixₙ e v → e (λx. fixₙ₋₁ e x) v` — each ReAct iteration decrements the bound
- **E-Fix-Zero:** `fix₀ e v → v` — forced termination when bound exhausted
- **E-Guard-Ok:** `guard e P v → v'` when P(v') — output passes validation
- **E-Guard-Fail:** `guard e P v → stuck` when ¬P(v') — checked runtime error
- **E-Case:** Classifier reduces to label lⱼ, then branch eⱼ — routing dispatch

### Task 0.4: Define the Lint Rule Derivation

Each lint rule must be derivable from the operational semantics. The paper's key insight: rules derived from semantics have 100% precision on fault injection, vs. ad-hoc rules with ~54% real-world precision.

Template for deriving a lint rule:
1. Identify the semantic defect in the calculus (e.g., "case with no branches = stuck for all inputs")
2. Define what pattern in a SKILL.md corresponds to that defect
3. The lint rule flags the pattern → proven sound by Theorem 5.8

### Task 0.5: Document the Full Mapping

Write `docs/spec/lambda-skills-calculus.md` — a self-contained spec that:
- Defines all types, terms, and reduction rules
- Maps each to the existing meta-lambda architecture
- Identifies what exists already (two-gate primitive, co-loading) and what's missing
- Serves as the reference for all subsequent phases

---

## Phase 1: Type System — Frontmatter Declarations

**Dependency:** Phase 0 spec.

### Task 1.1: Define Frontmatter Type Schema

Extend SKILL.md frontmatter with type declarations:

```yaml
---
name: my-skill
description: Use when doing X.
types:
  input: Str                          # required
  output: Str                         # required
  input_refinement: "valid_json"      # optional: predicate name
  output_refinement: "valid_yaml"     # optional: predicate name
composition:
  role: atomic                        # atomic | chain | router | parallel
  expects: [phase-handling]           # skills this one expects to be co-loaded with
  provides: [scope, verification]     # what this skill contributes to the stack
---
```

**Design decisions:**
- `input`/`output` are mandatory for typed skills, absent for untyped (backward compatible)
- Refinements are predicate names, not inline code — they reference a shared predicate registry
- `composition.role` tells the checker how this skill can be composed
- `composition.expects` enables dependency checking (does the loaded stack satisfy this skill's requirements?)

### Task 1.2: Define the Predicate Registry

Predicates are named validators that check output strings:

```yaml
# In a shared predicates.yaml or inline in skill frontmatter
predicates:
  valid_json:
    description: "Output is parseable JSON"
    check: "json.loads(output)"
  valid_yaml:
    description: "Output is parseable YAML"
    check: "yaml.safe_load(output)"
  non_empty:
    description: "Output is not empty or whitespace-only"
    check: "output.strip() != ''"
  has_file_paths:
    description: "Output contains at least one absolute file path"
    check: "re.search(r'/[\w/.-]+', output)"
```

### Task 1.3: Write the Type Checker

A Python script `tools/typecheck.py` that:
1. Reads a co-loaded skill stack (list of skill names/ paths)
2. Parses frontmatter from each SKILL.md
3. Verifies: output of skill N matches input of skill N+1 (for sequential composition)
4. Verifies: composition roles are compatible (atomic can't be composed with router without an adapter)
5. Verifies: `expects` dependencies are satisfied
6. Reports type errors with file:line references

### Task 1.4: Write Type Safety Theorems (Prose)

Adapt λA's theorems to skill composition:

**Theorem 1 (Progress):** If a typed skill stack is well-formed and the top-level term is not a value, then either it reduces to another well-formed term, or it is a guard failure (stuck).

**Theorem 2 (Preservation):** If a typed skill stack is well-formed and reduces, the result is well-formed under the same type environment.

**Theorem 3 (Type Safety):** Evaluation of a well-typed skill stack either produces a value of the expected type, or encounters a guard failure — never undefined behavior.

These are stated as prose theorems, not Coq. Mechanization is Phase 7.

### Task 1.5: Retrofit Existing Skills

Add type declarations to the four existing lambda-skills:

| Skill | input | output | role |
|---|---|---|---|
| two-gate | Str (task description) | Str (completion status) | atomic |
| inter-phase | Str (phase artifacts) | Str (plan diffs) | adapter |
| phase-handling | Str (phase number) | Str (transition log) | orchestrator |
| haga-lambda | Str (task) | Str (code changes) | adapter |

---

## Phase 2: Effect System

**Dependency:** Phase 1 (types must exist before effects can reference them).

### Task 2.1: Define Effect Declarations

Extend frontmatter:

```yaml
effects:
  uses: [llm, terminal, file, web, memory]  # what this skill uses
  produces: [file_changes, git_commits]      # side effects it creates
  requires: []                                # effects it needs from composed skills
  pure: false                                 # true if no side effects at all
```

**Effect taxonomy (from λA's sketched system):**
- `llm` — calls the language model (lam)
- `terminal` — executes shell commands (tool)
- `file` — reads/writes files (tool)
- `web` — makes HTTP requests (tool)
- `memory` — persistent state across invocations (mem)
- `pure` — no side effects (pure λ-calculus terms)

### Task 2.2: Effect Composition Rules

From λA's TE rules:

```
TE-Lam:    Γ ⊢ lam p θ : Str → Str ! llm(θ)
TE-Tool:   Γ ⊢ tool[f] : τ₁ → τ₂ ! io
TE-Mem:    Γ ⊢ mem e σ : τ → τ' ! state(Σ)
TE-Comp:   Γ ⊢ e₁ » e₂ : τ₁ → τ₃ ! ε₁ · ε₂
TE-Fix:    Γ ⊢ fixₙ e : τ → τ ! εⁿ
```

Effect composition forms a monoid (ε, ·, pure). This means:
- Sequential composition: effects are concatenated (ε₁ · ε₂)
- Fixed iteration: effect is repeated n times (εⁿ)
- Pure terms contribute no effects

### Task 2.3: Effect Safety Checker

Extend the type checker from Phase 1:
- A skill declaring `effects: {uses: [file], pure: false}` can't compose with a sandbox that blocks file access
- `requires` effects must be provided by composed skills or the runtime
- Pure skills can be optimized (memoized, reordered)

### Task 2.4: Effect-Aware Sandboxing

This is the practical payoff:
- A skill stack can declare a maximum effect budget
- The checker verifies no skill in the stack exceeds it
- Enables "this workflow can read files but not execute commands" guarantees

---

## Phase 3: Composition Operators as Primitives

**Dependency:** Phase 1 (types tell us what each operator's input/output must be).

### Task 3.1: Define the Operator Skill Format

Each composition operator becomes a primitive skill. The operator defines *structure*, not *content* — it's the skeleton that adapters hang domain logic on.

A composition operator skill has a new `operator: true` frontmatter field and defines:
- **Input slots:** what sub-skills it takes
- **Reduction rule:** what happens when the operator executes
- **Termination bound:** if applicable (for fixₙ)

### Task 3.2: Implement fixₙ — Bounded Loop

The most important new primitive. Maps to the two-gate pattern but with a hard termination bound.

```yaml
---
name: fixn-loop
description: Bounded iteration — repeat a body skill up to n times with guaranteed termination.
operator: true
types:
  input: Str
  output: Str
bounds:
  max_iterations: 10          # default, overridable
slots:
  body:                        # the skill to repeat
    type: Str → Str
  terminate_predicate:         # when to stop (optional, defaults to user approval)
    type: Str → Bool
---
```

Behavior: `fixₙ body input` — apply body to input, check terminate predicate. If not terminated and iterations < n, repeat. If iterations = n, force-return current state. Guaranteed to terminate in ≤ n steps.

This *bounds* the await-approval loop within each gate. Two-gate is a composition
of two guard operators — `(fix_n ∘ guard) » (fix_m ∘ guard)` — where each guard's
internal "await approval" loop is bounded independently. Two-gate does not become
fix₁; rather, each gate gains a termination bound. The two-gate primitive remains
the composition structure; fix_n is applied to each gate's await-approval loop.

### Task 3.3: Implement case — Router

```yaml
---
name: case-router
description: Dispatch to sub-skills based on input classification.
operator: true
types:
  input: Str
  output: Str
slots:
  classifier:                  # skill that classifies input → label
    type: Str → ⟨lᵢ⟩
  branches:
    l₁: skill_a                # skill for label l₁
    l₂: skill_b                # skill for label l₂
  default: skill_fallback      # required for exhaustiveness
---
```

### Task 3.4: Implement guard — Output Validation

```yaml
---
name: guard-validate
description: Validate skill output against a predicate. Retry on failure.
operator: true
types:
  input: Str
  output: {x:Str | P(x)}       # refinement type
slots:
  body:                         # skill whose output to validate
    type: Str → Str
  predicate: valid_json         # from predicate registry
  retry: 3                      # max retries before stuck
---
```

### Task 3.5: Implement » — Sequential Composition

Already exists implicitly (co-loading order). Make it explicit:

```yaml
---
name: compose-seq
description: Sequential composition — run skill A, pipe output to skill B.
operator: true
types:
  input: τ₁
  output: τ₃                    # A: τ₁→τ₂, B: τ₂→τ₃
slots:
  first:
    type: τ₁ → τ₂
  second:
    type: τ₂ → τ₃
---
```

### Task 3.6: Implement | — Parallel Composition

```yaml
---
name: compose-parallel
description: Run skills A and B in parallel, combine results as a pair.
operator: true
types:
  input: τ
  output: τ₁ × τ₂
slots:
  left:
    type: τ → τ₁
  right:
    type: τ → τ₂
---
```

---

## Phase 4: Static Lint Tool

**Dependency:** Phases 1-3 (lint rules reference types, effects, and operator semantics).

### Task 4.1: Derive Lint Rules from Semantics

Map λA's 26 lint rules to skill-specific equivalents:

| λA Rule | Severity | λ-skills Equivalent |
|---|---|---|
| L001 | ERROR | Skill body is empty after frontmatter — no instructions |
| L002 | ERROR | `types.input` or `types.output` missing in a typed skill |
| L003 | ERROR | `bounds.max_iterations: 0` — vacuous loop |
| L004a | ERROR | Skill uses fixₙ but has no terminate branch and no alternative |
| L004b | WARN | Skill uses fixₙ with bounded fallback but no explicit base case |
| L004c | INFO | Framework/adapter provides termination (not in this skill's body) |
| L005 | ERROR | case-router has no branches |
| L006 | ERROR | Type mismatch: skill A's output ≠ skill B's expected input |
| L007 | WARN | Skill declares `effects.uses: [llm]` but no model configured |
| L008 | ERROR | Adapter has no delegation instruction ("Execute the X process") |
| L009 | WARN | Orchestrator references a phase/skill that doesn't exist |
| L010 | WARN | Composition expects co-loaded skill X but it's not in the load list |
| L011 | WARN | Adapter's audit steps contain implementation actions (scope creep) |
| L012 | ERROR | guard predicate not in predicate registry |
| L013 | WARN | case-router has no default branch (non-exhaustive) |
| L014 | INFO | Skill is pure but declares effects (unnecessary) |
| L015 | ERROR | Two skills in a stack have the same `name:` (collision) |
| L016 | WARN | Adapter too thin — domain lens adds no information beyond primitive |
| L017 | WARN | `bounds.max_iterations` missing on a skill that describes a loop |
| L018 | WARN | `composition.role` incompatible with composed skills |
| L019 | INFO | Skill has `expects` that are satisfied by defaults/runtime |
| L020 | WARN | effect `requires` not provided by any composed skill |
| L021 | ERROR | Unbounded loop with no termination mechanism at all |
| L022 | WARN | Non-exhaustive dispatch — labels in classifier output don't cover all branches |
| L023 | ERROR | Adapter delegation instruction names a skill not in the load stack |
| L024 | WARN | Orchestrator continuation references a phase/skill with incomplete type |
| L025 | WARN | Deprecated predicate referenced in guard |
| L026 | INFO | Skill has no `composition` section (untyped, legacy mode) |

### Task 4.2: Implement YAML+Body Joint Analysis

This is the paper's critical finding: YAML-only lint precision is ~52%, but joint YAML+Python analysis raises it to 96%. For skills, the equivalent is: lint must read the skill *body* (the markdown instructions), not just the frontmatter.

Examples of what body analysis catches that frontmatter-only misses:
- Adapter has delegation instruction in frontmatter `expects` but NOT in the body → L008
- Skill body describes a loop ("retry until...") but frontmatter has no `bounds.max_iterations` → L017
- Adapter body contains "write file", "commit" in audit steps → L011 (scope creep)
- Frontmatter says `composition.role: atomic` but body says "compose with..." → L018

Implementation: parse the markdown body for patterns (delegation instructions, loop descriptions, implementation verbs in audit sections) and cross-reference with frontmatter declarations.

### Task 4.3: Build the Lint CLI

```bash
# Lint a single skill
lambda-skills lint ~/.hermes/skills/lambda-skills/inter-phase/SKILL.md

# Lint a co-loaded stack (checks composition)
lambda-skills lint --stack phase-handling,inter-phase,two-gate

# Lint all skills in a directory
lambda-skills lint ~/.hermes/skills/

# Output format
λ ~/.hermes/skills/lambda-skills/inter-phase/SKILL.md
  ERROR  L008  Adapter has no delegation instruction in body
  WARN   L017  Body describes loop pattern but no bounds.max_iterations
  INFO   L004c Framework handles termination externally
```

### Task 4.4: Validation — Fault Injection

Reproduce the paper's Experiment A: inject 5 known fault types into known-good skills, verify 100% detection.

Fault types:
1. Remove delegation instruction from adapter
2. Empty the skill body after frontmatter
3. Remove `types.output` from frontmatter
4. Set `bounds.max_iterations: 0`
5. Remove all branches from a case-router

### Task 4.5: Validation — Production Corpus

Run lint on the Hermes skill corpus (~100+ skills in `~/.hermes/skills/`) and categorize findings — mirroring the paper's Table 2 and stratified analysis (Table 3).

---

## Phase 5: Pipeline Algebra

**Dependency:** Phase 3 (operators must exist to prove algebraic properties about them).

### Task 5.1: Define Identity Skill

```yaml
---
name: identity
description: Identity skill — returns input unchanged. Neutral element for ».
operator: true
types:
  input: τ
  output: τ                    # same type
effects:
  pure: true                   # no side effects
---
```

Behavior: `identity(x) = x`. Corresponds to λA's `id = λx:Str. x`.

### Task 5.2: Prove Monoid Laws (Prose)

Adapt Theorem 5.7 to skill composition:

1. **Associativity:** `(A » B) » C ≡ A » (B » C)` — co-loading order is associative
2. **Identity:** `A » identity ≡ identity » A ≡ A` — identity skill can be eliminated

These enable optimization: `identity » skill_A » identity » skill_B` reduces to `skill_A » skill_B`.

### Task 5.3: Implement Algebraic Optimizer

A tool that reads a skill composition stack and applies algebraic rewrites:
- Eliminate redundant identity skills
- Reorder pure skills (those with `effects.pure: true`) for optimization
- Fuse adjacent LLM calls when possible (lam » lam → single combined lam)

### Task 5.4: Prove More Rewrite Rules

Additional algebraic identities from λA's design:

- `guard(guard(e, P₁), P₂) ≡ guard(e, P₁ ∧ P₂)` — predicate fusion
- `case(case(e, branches₁), branches₂) ≡ case(e, flattened_branches)` — dispatch fusion
- `fixₙ(fixₘ e) ≡ fix_min(n,m) e` — nested bounded loop collapse
- `mem(mem(e, σ₁), σ₂) ≡ mem(e, σ₁ ∪ σ₂)` — store fusion

---

## Phase 6: Tooling & Integration

**Dependency:** Phases 1-5.

### Task 6.1: CLI Entry Point

`lambda-skills` CLI with subcommands:
- `lint` — static analysis (Phase 4)
- `check` — type + effect checking (Phases 1-2)
- `compose` — verify a co-loaded stack is well-formed
- `optimize` — algebraic rewriting (Phase 5)
- `spec` — output the formal specification (Phase 0)

### Task 6.2: Hermes Integration

- Pre-load hook: when Hermes loads a co-loaded skill stack, run `check` + `lint` first
- Configurable severity threshold: `skills.lint.min_severity: warn` (default: skip INFO)
- Gate mode: refuse to load a stack with ERROR-level lint findings

### Task 6.3: Test Suite

- Unit tests for type checker, effect checker, each lint rule
- Fault injection suite (Experiment A reproduction)
- Regression tests on known-good skill stacks
- Semantic faithfulness tests (like the paper's 125 tests): for each operator, construct a term, execute against a mock LLM, verify output matches semantics

### Task 6.4: Documentation

- `docs/spec/lambda-skills-calculus.md` — formal spec (Phase 0 deliverable)
- `docs/usage/lint-rules.md` — each lint rule with example and remediation
- `docs/usage/type-system.md` — type declarations, predicate registry, effect taxonomy
- `docs/usage/operators.md` — each composition operator with usage patterns
- `docs/usage/migration.md` — adding types to existing untyped skills

---

## Phase 7: Formal Verification (Long-Term)

**Dependency:** Phase 1-5 spec must be stable.

### Task 7.1: Mechanize Core Theorems

Port λA's Coq development (1,519 lines, 42 theorems) to λ-skills. Key theorems:
- Progress: well-typed term either reduces or is stuck (guard failure)
- Preservation: reduction preserves type
- Type Safety: from Progress + Preservation
- Substitution lemma
- Termination of bounded fixpoints
- Pipeline algebra (monoid laws)
- Lint soundness

### Task 7.2: Evaluate on Hermes Skill Corpus

Reproduce the paper's evaluation methodology:
- Run lint on all ~100+ Hermes skills
- Stratify by skill category (software-development, mlops, etc.)
- Manually verify a sample to measure real-world precision
- Compare YAML-only (frontmatter) vs. joint (frontmatter + body) lint precision

### Task 7.3: Publish

Write a companion paper or technical report: "λ-skills: A Typed Lambda Calculus for Hermes Skill Composition." Frame it as an application of λA to a real-world agent system, with the meta-lambda stack as a case study.

---

## Dependency Graph

```
Phase 0 (Spec)
  └─→ Phase 1 (Types)
        └─→ Phase 2 (Effects)
        └─→ Phase 3 (Operators)
              └─→ Phase 4 (Lint)
              └─→ Phase 5 (Algebra)
                    └─→ Phase 6 (Tooling)
                          └─→ Phase 7 (Verification)
```

Phases 1 and 3 can partially overlap (types needed for operators, but operator definitions can start once the type spec is stable). Phase 4 depends on 1-3 because lint rules reference types, effects, and operator semantics. Phases 5 and 6 are independent of each other. Phase 7 gates on everything.

---

## Risks & Open Questions

1. **Co-loading as composition.** Is concatenating skills into a single prompt truly equivalent to function composition? The LLM may not respect the semantics — it could "skip" an operator. This is the compilation adequacy problem (§5.3): we can't prove the LLM executes the calculus correctly, only that the terms are well-formed. Mitigation: semantic faithfulness tests (125 tests passing means 100% empirical agreement).

2. **Str as the only base type.** The paper argues this suffices ("agent configurations operate at the string level"), but richer types (JSON schema, file paths, structured outputs) would catch more errors. Risk: the type system is too weak to be useful. Mitigation: start with Str and add structured subtypes in a later phase if needed (this is exactly the paper's §9 future work).

3. **Body parsing for lint.** Parsing natural language skill instructions to detect loops, delegation, and scope creep is inherently heuristic. The paper's 54% YAML-only precision reflects the same problem: the semantics are in the body, not the metadata. Mitigation: start with high-precision rules (L008: explicit delegation string match), add heuristic rules as Warn-level, validate via fault injection.

4. **Adoption burden.** Requiring type/effect declarations on all skills is a high bar. Mitigation: everything is optional — untyped skills work as before, just without the guarantees. The tooling degrades gracefully.

5. **Operator skills vs. meta-lambda skills.** The existing two-gate/inter-phase/phase-handling stack is one specific composition pattern. New operators (case, guard, fixₙ) are different patterns. Do they coexist as separate primitives, or does two-gate become an instance of fixₙ? Risk: fragmentation. Mitigation: Phase 3 should fold two-gate into fix₁ with user-approval predicate — one general primitive, not many special ones.

---

## Phase 0 Deliverable (Next Step)

Before any implementation, write `docs/spec/lambda-skills-calculus.md` — the formal specification. This is the reference document all subsequent phases build against. It should be thorough enough that someone could implement the calculus from it without reading the λA paper.
