# lambda-skills Implementation Plan — The Conversational Model

> **For Hermes:** Use the two-gate primitive + inter-phase adapter for each
> phase. Gates apply per-phase.

**Goal:** Build a character-based discussion system for Hermes skill
composition — an OS layer that casts characters, each with a distinct
perspective, who wrestle with a shared idea while hearing the voices
that came before them. The system provides structural lint, conversation
guardrails, and empirical validation grounded in the insight that
structural defects account for the majority of output quality loss.

---

## Core Thesis

LLMs don't reduce terms. They inhabit perspectives. A skill that says
"you are the skeptic auditor, your job is to find gaps" produces a
different cognitive posture than one that says "you are the constructive
planner, your job is to build a path forward." The LLM's latent-space
trajectory through a prompt is shaped by the voices it inhabits, in the
order it inhabits them.

The λA paper proves that structural defects — missing termination, empty
branches, silent delegation collapse — account for the majority of
output failures in agent configurations. Its lint tool catches these
with 96-100% precision. We adopt the same thesis: **structure eliminates
a class of errors.** But our structure is conversational, not computational.
Instead of typed terms and reduction rules, we build characters with clear
perspectives, discourse operators that govern who speaks next, and
guardrails that prevent conversations from derailing.

**Reading order is outside-in.** Hermes flattens co-loaded skills into a
single prompt. The OS layer (first-read, outermost) sets the discussion
format. Characters are introduced in speaking order. Gates (last-read,
innermost) carry the strongest enforcement via recency bias. This is the
opposite of λ-calculus evaluation order (inside-out) — but it's correct
for LLM attention dynamics.

---

## Architectural Decisions

### 1. Characters, Not Functions

The unit of composition is a *perspective*, not a typed term. A character
has: a cognitive posture (skeptical, constructive, summarizing), a
relationship to the previous speaker (what they use, what they ignore),
a way of wrestling with the idea (their protocol), a contribution (what
they add to the discussion), and an exit condition (when they're done).

### 2. The OS Layer as Moderator

The OS layer is the first-loaded skill. It does three things:
1. States the topic (prompt + context)
2. Introduces the cast — who speaks, in what order
3. Defines the discussion rules — each character hears the one before,
   the user's "approved" is the moderator's cue, gates are hard stops

The OS layer does NOT manage runtime transitions — it's read once, at
session start. The rules it encodes are carried forward by the LLM
through the conversation.

### 3. Cumulative Context, Not Isolated

Each character hears everything before them — the original topic, the
OS layer's framing, and every prior character's contribution. Context
is cumulative. This is the opposite of Approach B's isolated-call model,
and it's exactly what makes Approach C work in a single LLM call: the
conversation is the state.

### 4. Outside-In Reading Order

Loading order: OS layer → character 1 → character 2 → ... → two-gate.
The OS layer (first-read) establishes the frame. Characters (mid-read)
provide their perspectives. Two-gate (last-read, highest recency)
provides enforcement. The LLM's attention carries the frame forward
while privileging the most-recently-read enforcement instructions.

### 5. Lint Inspects Character Body

The paper's finding: YAML-only lint gets 54% precision; adding body
inspection raises it to 96-100%. Our lint rules inspect the character's
full skill body — not just frontmatter — to catch missing voice,
ignored predecessors, dead branches, and missing exits.

### 6. Gates as Moderator Interventions

The two-gate pattern remains, but reframed: gates are the moderator
(the OS layer) intervening in the conversation. "Gate 1: stop. The user
must approve before any character acts on files." The gates are not a
computational guard operator — they're a discussion protocol: some
actions require the moderator's explicit cue.

---

## The Character Format

Every character skill follows this structure:

```markdown
## Character: <name>

### Perspective
Your cognitive posture. What lens do you bring? What do you care about?
What do you deliberately ignore? This is the single most important
section — it establishes the voice.

### Relationship to Previous Speaker
How do you use what came before you? Do you build on it? Challenge it?
Filter it through your lens? If you're first, you start from the idea
itself.

### How You Wrestle
Your protocol. The steps you take, the questions you ask, the evidence
you gather. This is process, not product — how you think, not what you
output.

### Your Contribution
What you add to the discussion. The shape of your voice: a survey of
gaps, a propagation plan, a verification report. Described as a
contribution to the conversation, not as output_type.

### You're Done When
Your exit condition. When you've said what you came to say, you stop.
The next character enters. If you have nothing to contribute, you say
so and pass — silence is a valid contribution.
```

---

## Phase 0: The Discussion Model

### Goal

Define the OS layer format and the character lifecycle. Write a reference
conversation that demonstrates characters wrestling with an idea in
sequence.

### Tasks

**Task 0.1: Draft the OS layer format.**

The OS layer is a Hermes skill (SKILL.md) designed to be loaded FIRST.
It establishes:

- The discussion format ("voices will speak in order; each hears the
  one before")
- The cast ("Character 1: Auditor. Character 2: Planner. Character 3:
  Verifier.")
- The transition rule ("when a character exits, the next enters")
- The moderator protocol ("before any character writes to files: present
  plan, await user approval. before committing: verify, summarize, await
  approval")
- The silence rule ("if a character has nothing to add, they say so and
  the moderator moves on")

**Task 0.2: Define the character lifecycle.**

A character's life in the conversation:

1. **Enter:** the moderator announces them. "The Auditor has finished.
   Now the Planner. Planner: you heard the Auditor's findings. What
   should be done about them?"
2. **Wrestle:** the character does their protocol — gathers evidence,
   asks questions, forms a perspective.
3. **Contribute:** the character speaks their piece, adding to the
   discussion.
4. **Exit:** the character declares they're done. "I've surveyed the
   implementation. The Planner has what they need."
5. **Transition:** the moderator checks the exit condition, then
   introduces the next character.

**Task 0.3: Write a reference conversation.**

Take the inter-phase ceremony (HAGA Phase N → Phase N+1 transition) and
script a full conversation with:

- OS layer: sets the scene
- HAGA Context: establishes the world (repo, conventions)
- DriftAuditor: surveys implementation vs plan
- User: approves audit findings
- PropagationPlanner: plans patches from drift findings
- Verifier: confirms patches are correct
- User: approves commit

This reference conversation is the spec that Phase 7 validates against.

### Success Criteria

- The OS layer format is concrete enough that a `λ-skill compose` tool
  could generate one from a cast list
- The character lifecycle is documented with clear entry/exit conditions
- The reference conversation demonstrates voice shifts, cumulative
  context, and gate compliance

---

## Phase 1: Character Format Specification

### Goal

Specify the character brief format in detail. Build a reference character
(the DriftAuditor) and author the skill template.

### Tasks

**Task 1.1: Formalize the character brief structure.**

The 5-section format (Perspective, Relationship, Wrestling, Contribution,
Exit) plus frontmatter fields:

```yaml
---
name: drift-auditor
description: Skeptical auditor — surveys implementation, finds gaps,
  never proposes fixes. Responds to the HAGA Context character.
character:
  perspective: skeptical
  position: 2  # speaks second, after HAGA Context
  speaks_after: haga-lambda
  speaks_only_if: null  # always cast
  max_turns: 1  # speaks once, then exits
---
```

The `character` frontmatter block enables mechanical validation (Phase 4
lint, Phase 6 tooling) while the body provides the voice.

**Task 1.2: Build the DriftAuditor reference character.**

A fully-authored SKILL.md that demonstrates every section:

- Perspective: "You dig for gaps, not solutions. Be specific — file
  paths, function names, structural mismatches. Resist the urge to
  propose fixes. That's the Planner's job."
- Relationship: "You follow the HAGA Context character. Use their repo
  knowledge as your working ground. You do not need to re-establish
  what ~/haga_master/ means."
- Wrestling: search for new classes/files, diff signatures against the
  plan, check dependency ordering
- Contribution: a gap survey organized by severity
- Exit: "I've surveyed the implementation. The Planner takes it from
  here."

**Task 1.3: Document the character template.**

A concise authoring guide in `docs/character-template.md` that future
characters follow.

### Success Criteria

- The DriftAuditor loads as a valid skill (`skill_view("drift-auditor")`)
- The character brief format captures all 5 sections
- The frontmatter `character` block enables mechanical validation
- The template is clear enough that a new character (e.g.,
  PropagationPlanner) can be authored from it without guessing

---

## Phase 2: Discourse Operators

### Goal

Define the operators that govern conversation flow: who speaks after
whom, who responds to what, and when a character is conditionally cast.

### Tasks

**Task 2.1: `speaks-after` — the default chain.**

The simplest operator. Character N's `speaks-after` field names
Character N-1. The moderator's default rule: "when the previous
character exits, the next enters."

In the OS layer:
```
The Auditor finishes. Now the Planner.
Planner: you heard the Auditor's findings. What should be done?
```

**Task 2.2: `responds-to` — targeted address.**

A character explicitly addresses a specific prior contribution, not
just the immediate predecessor. "The Verifier responds to the Planner's
propagation plan, not the Auditor's findings."

This is the conversation-level equivalent of passing a specific prop.
The character states: "I'm responding to CHARACTER.NAME's point about
TOPIC." This is enforced structurally (the moderator checks) but is
still a narrative convention, not a mechanical prop check.

**Task 2.3: `only-if` — conditional casting.**

A character is cast only if a condition holds. "Only cast the
PropagationPlanner if the DriftAuditor found gaps." If no gaps, the
Planner is skipped — the Verifier enters directly.

Conditions are declared in the character's `speaks_only_if` frontmatter:
- `"drift-auditor.found_gaps"` — a named condition produced by the
  previous character
- In Approach C, the LLM evaluates this condition narratively
- In Approach B, the condition is a mechanical check on the previous
  character's structured output

**Task 2.4: The Moderator Protocol (gates as discourse operators).**

Gates are special operators that interrupt the conversation flow:

- `gate(user.approve)` — present to user, await explicit approval.
  Highest precedence. No character may act on files until the moderator
  cues.
- `gate(character.exit)` — standard transition: character declares done,
  moderator checks exit condition, next enters.
- `gate(character.silence)` — if a character has nothing to add, they
  pass and the moderator moves on.

These gates are defined in `two-gate` (the innermost skill) but their
placement in the conversation flow is determined by the OS layer.

### Success Criteria

- Each operator is documented with its frontmatter field and its OS
  layer representation
- The `only-if` operator is demonstrated: a character that is cast vs
  skipped based on condition
- The gate operators are clearly distinguished from character
  transitions

---

## Phase 3: Conversation Guardrails

### Goal

Prevent conversations from derailing: bounded turns, enforceable exits,
silence rules, and character recasting on failure.

### Tasks

**Task 3.1: Bounded turns (`max_turns`).**

A character's `max_turns` field limits how many times they can speak.
Default: 1 (speak once, then exit). For iterative characters (e.g.,
"try to fix the build, check, try again"): max_turns = n.

In the LLM: "You may attempt this at most 3 times. After the 3rd
attempt, exit regardless of outcome." The moderator tracks turns
narratively. In Approach B, the moderator tracks mechanically.

**Task 3.2: Exit enforcement.**

A character's exit condition is checked by the moderator. If the
character exits without having contributed (says "done" but produced
nothing), the moderator flags it. If the character attempts to proceed
to the next character without exiting ("and then the Planner should..."),
the moderator intervenes: "Exit first. Then the Planner enters."

This prevents the most common conversation collapse mode: characters
merging into each other without clear boundaries.

**Task 3.3: Silence rules.**

If a character has nothing to contribute (e.g., the `only-if` condition
wasn't met, or the topic doesn't apply), they say "I have nothing to add"
and the moderator moves on. Silence is not failure — it's a valid path.

**Task 3.4: Recast on failure.**

If a character's contribution is insufficient (the user says "that's not
what I needed"), the moderator recasts the character with clarified
instructions. "The Auditor's survey missed X. Auditor: try again. Focus
specifically on Y."

This is the conversational equivalent of error handling. It replaces
the computational "type error" and "guard failure" with a natural
discussion pattern: "try again, here's what was missing."

### Success Criteria

- A character can be bounded to N turns and forced to exit after N
- The exit condition is observable: character explicitly declares "done"
  and does not attempt to introduce the next character
- Silence is handled gracefully: the moderator acknowledges it and
  moves on
- Recast is demonstrated: insufficient contribution → clarified
  instruction → second attempt

---

## Phase 4: Structural Lint

### Goal

Catch character design defects before they hit the LLM. Rules derived
from the conversational model, validated by fault injection.

### Tasks

**Task 4.1: Lint rule catalog.**

| Rule | Severity | What It Catches | Rationale |
|---|---|---|---|
| C001 | ERROR | Character has no Perspective section | A character without a cognitive posture is a generic instruction set — no voice to inhabit |
| C002 | ERROR | Character references no prior speaker (no `speaks-after`, no `responds-to`, no mention of predecessor in body) | Character enters the conversation with no awareness of what came before — produces disconnected output |
| C003 | WARN | `speaks-only-if` references a condition no prior character declares | Dead branch: this character can never be cast |
| C004 | ERROR | No exit condition: neither "You're Done When" section nor frontmatter `max_turns` | Character never declares completion — conversation hangs |
| C005 | WARN | Character's Contribution section is a verbatim copy of its task description | No added perspective — dead weight (thickness rule violation) |
| C006 | ERROR | `speaks-after` names a character not in the cast | Missing dependency — character references a voice that won't speak |
| C007 | WARN | `max_turns` unbounded (none or 0) with no exit condition | Potential infinite loop — character may spin |
| C008 | ERROR | Character attempts to define gates or transitions (usurping the moderator) | A character should not manage the conversation flow — that's the OS layer's job |
| C009 | WARN | Character instructions reference a specific file path or project (not portable) | Domain leakage — character should be portable across projects |

Rules C001-C004 mirror the paper's L003 (vacuous loop), L004a (infinite loop), L005 (empty branches), and L021 (unbounded turns). Rules C005-C009 are conversational-model-specific.

**Task 4.2: Implement the lint runner.**

A Python script in `tools/lint_character.py` that:
1. Reads a SKILL.md
2. Parses frontmatter for `character:` block
3. Checks each rule against the parsed structure
4. Reports severity + line + fix suggestion

No LLM needed — pure structural analysis. The paper's 96-100% precision
is achievable because all rules are mechanically checkable.

**Task 4.3: Fault injection validation.**

Create deliberately broken character briefs and verify the lint catches
them. Minimum: one broken character per rule. Record in `tests/faults/`.

### Success Criteria

- All 9 lint rules detect their target defect with 100% precision
  on injected faults (no false negatives)
- All existing lambda-skills characters pass lint without modification
  (no false positives)
- The lint runner runs in under 100ms for any skill file

---

## Phase 5: Conversation Coherence

### Goal

Empirically validate that character-structured conversations produce
better outputs than unstructured prompting. Measure, don't assume.

### Tasks

**Task 5.1: Build a benchmark.**

Take the HAGA inter-phase ceremony as the benchmark task:
- Input: Phase N completion log + Phase N+1 plan + repository state
- Expected output: drift findings + propagation plan + verified patches
- Metrics: drift detection rate, patch completeness, conversation
  round count, gate compliance

**Task 5.2: Run structured vs unstructured comparison.**

Three conditions:
1. **Structured (character-based):** OS layer + HAGA Context + DriftAuditor + PropagationPlanner + Verifier + two-gate
2. **Co-loaded (current):** haga-lambda + phase-handling + inter-phase + two-gate (the existing lambda stack)
3. **Unstructured:** single skill with all instructions in one document, no character framing, no gate enforcement

Each condition runs the inter-phase ceremony 5 times on identical
starting state. Measure:

- Drift detection rate: did the Auditor find all known gaps?
- Propagation completeness: did the Planner address every gap?
- Gate compliance: did the process respect both gates?
- Conversation coherence rating (manual): did the conversation feel
  like characters building on each other, or a single voice?

**Task 5.3: Publish results.**

Document in `docs/benchmarks/inter-phase-comparison.md`. Include:
- Raw data (all runs, all conditions)
- Statistical summary
- Qualitative observations (where did the structured condition excel?
  Where did it fall short?)

### Success Criteria

- Structured condition matches or exceeds co-loaded condition on all
  metrics
- Gate compliance is 100% for structured condition (two-gate is
  innermost, highest recency — it should hold)
- At least one metric shows structured > unstructured with statistical
  significance (p < 0.05 via bootstrap of 5 runs per condition)

---

## Phase 6: Tooling

### Goal

Build the CLI tools that make character-based skill authoring practical.

### Tasks

**Task 6.1: `λ-skill validate` — character lint.**

Wraps the lint runner from Phase 4. Usage:

```bash
λ-skill validate drift-auditor
# ✓ Perspective: skeptical
# ✓ speaks-after: haga-lambda
# ✓ Exit condition: present
# 0 errors, 0 warnings
```

**Task 6.2: `λ-skill compose` — OS layer generator.**

Given a cast list, generates the OS layer skill:

```bash
λ-skill compose --cast haga-lambda,drift-auditor,propagation-planner,verifier \
                --topic "inter-phase ceremony" \
                --gates two-gate \
                --output os-inter-phase
```

Produces `OS-inter-phase/SKILL.md` with the moderator's introduction,
cast order, transition rules, and gate references.

**Task 6.3: `λ-skill template` — character scaffold.**

Generates a blank character brief:

```bash
λ-skill template --name dependency-checker --perspective skeptical --after drift-auditor
```

Produces `dependency-checker/SKILL.md` with all 5 sections and
frontmatter, ready to fill in.

### Success Criteria

- `validate` catches all injected faults from Phase 4.3
- `compose` output loads without error and establishes the correct
  cast order
- `template` output is author-complete: nothing to delete, just fill in

---

## Phase 7: Empirical Validation

### Goal

Prove the system works end-to-end: a full inter-phase ceremony driven by
characters, measured against the reference conversation from Phase 0.3.

### Tasks

**Task 7.1: Execute the reference conversation 10 times.**

Using the OS layer + character stack from Phase 6's compose output,
run the full inter-phase ceremony 10 times on identical starting state.
The user observes each run and rates:

- Character distinctness: did the LLM shift voice between characters?
- Contextual awareness: did each character reference the previous one?
- Gate compliance: binary per gate per run
- Output quality: drift detection, propagation completeness

**Task 7.2: Fault injection on the character stack.**

Inject the same structural defects from Phase 4.3 into a running
conversation. Observe:

- Does the defect cause a measurable drop in output quality?
- Does the defect cause gate violations?
- Does the lint rule accurately predict what will go wrong?

**Task 7.3: Document findings.**

In `docs/validation/empirical-results.md`: raw data, statistical
summary, qualitative observations, and specific cases where the
character model succeeded or failed.

### Success Criteria

- 90%+ gate compliance across all runs (10 runs × 2 gates = 20
  compliance checks; pass if ≥ 18)
- At least 7 of 10 runs rated "character voices distinct" by the user
- Fault injection shows measurable degradation — the lint rules catch
  defects that matter
- The reference conversation from Phase 0.3 is achieved or exceeded
  in at least 5 of 10 runs

---

## Phase Dependencies

```
Phase 0 (Discussion Model)
  └─→ Phase 1 (Character Format)
        └─→ Phase 2 (Discourse Operators)
              └─→ Phase 3 (Guardrails)
                    └─→ Phase 4 (Lint)
                    └─→ Phase 5 (Coherence)
                          └─→ Phase 6 (Tooling)
                                └─→ Phase 7 (Validation)
```

Phases 4 and 5 are independent of each other. Phase 6 depends on 4
(lint runner). Phase 7 gates on everything.

---

## Risks & Open Questions

1. **Voice shift fidelity.** Can the LLM reliably shift cognitive
   posture between characters in a single prompt? The paper's 4% gap
   (96% precision vs 100% theoretical maximum) suggests occasional
   failure. Mitigation: empirical measurement in Phase 5.

2. **Outside-in reading order.** The OS layer (first-read) must
   establish rules that the LLM carries forward through the entire
   conversation. If the LLM forgets the OS layer's framing after
   reading multiple character skills, the structure collapses.
   Mitigation: the OS layer should be short (screen-height) and
   the character skills should re-state their relationship to the
   conversation (not depend on the OS layer for context).

3. **Character authoring burden.** Writing a good character is harder
   than writing a checklist — it requires creative-writing discipline:
   defining a voice, a perspective, a relationship. Risk: authors
   produce "thin" characters that are just instructions with different
   headers. Mitigation: the C005 lint rule catches thin characters.
   The template (Phase 6.3) provides scaffolding.

4. **Cumulative context vs approach B.** The conversational model
   passes full context forward; Approach B isolates each call.
   Characters designed for Approach C may not work in Approach B
   (they may depend on conversational context that Approach B
   won't provide). Mitigation: the character format includes
   "Relationship to Previous Speaker" — making the dependency
   explicit. In Approach B, this becomes a mechanical prop-in
   declaration. A character that needs "everything everyone said" is
   a red flag — it's either doing too much or badly scoped.

5. **The OS layer as a skill vs. as a tool.** Is the OS layer a static
   SKILL.md, or does `λ-skill compose` generate it dynamically per
   cast? If static, the cast is fixed and can't be reconfigured without
   editing the OS layer. If generated, the OS layer is ephemeral and
   can be composed per-task. Decision: Phase 0 writes a reference OS
   layer by hand (static). Phase 6 builds the generator (dynamic).
   Both are valid; the generator just removes boilerplate.

---

## Relation to Existing lambda-skills Stack

The existing stack (haga-lambda, phase-handling, inter-phase, two-gate)
predates this plan. It was designed as a primitive/adapter/orchestrator
composition, not as characters. The inter-phase ceremony expressed in
the new character model becomes:

```
OS-inter-phase (OS layer — generated by λ-skill compose)
haga-lambda (context character — unchanged)
drift-auditor (new — splits inter-phase's audit steps into a character)
propagation-planner (new — splits inter-phase's implementation into a character)
verifier (new — splits inter-phase's verification into a character)
two-gate (enforcement — unchanged)
```

The inter-phase skill dissolves into three characters, each with a
single perspective. phase-handling dissolves into the OS layer's
moderator protocol. Two-gate remains as the enforcement layer.

This is a breaking change to the existing skills — but one that
simplifies them. Each character does less and has a clearer voice.

---

## Architectural Appendix: Mechanical Reduction (Approach B)

For future reference: the conversational model (Approach C) is our
current execution strategy. A mechanical reduction engine (Approach B),
modeled on the λA paper's lambdagent runtime, would replace the LLM's
narrative-driven conversation with deterministic subagent chaining.

| Property | Approach C (this plan) | Approach B (future) |
|---|---|---|
| **Reducer** | LLM attention (narrative) | Python AST walker (deterministic) |
| **Character evaluation** | One LLM call, all characters in context | One LLM call per character, isolated context |
| **Context per character** | Cumulative: full conversation | Isolated: character instructions + prior character's structured output |
| **Composition (speaks-after)** | OS layer's narrative rule | Mechanical: `eval(B, structured_output(A))` |
| **only-if casting** | LLM evaluates condition narratively | Mechanical check on prior output |
| **Termination** | Gate stops + user approval | fix_n guarantees N-turn bound |
| **Provability** | Empirical (gate compliance, coherence rating) | Type safety, termination — mechanically provable |
| **API calls** | 1 per conversation turn | N per cast (one per character) |

Approach B implementation would use Hermes `delegate_task` to spawn
isolated subagents — each character gets its own call, receives only
the prior character's structured output, produces its own structured
output. The parent agent is the moderator/director.

The character format in this plan is designed to be forward-compatible:
a character's "Relationship to Previous Speaker" section declares what
context it needs, which in Approach B becomes the mechanical prop-in.
The "Your Contribution" section describes what it produces, which
becomes the mechanical prop-out. The "You're Done When" section becomes
the end-of-evaluation marker.

---

## Phase 0 Deliverable (Next Step)

Before any implementation, write two documents:

1. `docs/spec/discussion-model.md` — The Discussion Model: OS layer
   format, character lifecycle, voice chaining rules, gate integration.
   This is the formal spec for the conversational system.

2. `docs/examples/reference-conversation.md` — A scripted reference
   conversation: the HAGA inter-phase ceremony expressed as characters
   wrestling with the idea. This is the spec that Phase 7 validates
   against.
