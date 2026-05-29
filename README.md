# lambda-skills

A typed lambda calculus for [Hermes agent](https://github.com/nous-research/hermes-agent) skill composition. Formal semantics, static lint, and compositional operators for co-loaded agent skills.

## Motivation

LLM agent frameworks lack formal semantics — there is no principled way to determine whether a skill composition is well-formed or will terminate. This project applies the insights of [λA: A Typed Lambda Calculus for LLM Agent Composition](papers/typed_lambda.pdf) (Liu, 2026) to the Hermes skill system, building on the co-loading composition pattern developed in [meta-lambda](https://github.com/mcndjxlefnd/lambda-skills/tree/main/meta-lambda).

The key insight, from both λA and our own work: **agent configurations already encode a lambda calculus — making this structure explicit yields type safety, termination guarantees, static lint, and pipeline algebra.**

## Architecture

Skills compose via three layers, co-loaded in outer-first order:

```
orchestrator → adapter → primitive
```

| Layer | Role | λA Analogue |
|---|---|---|
| **Primitive** | Pure mechanism — typed operators (gates, case, guard, fix_n). No domain knowledge. | Core reduction rules (E-Fix-Step, E-Case, E-Guard) |
| **Adapter** | Domain-specific lens on a primitive. Types the gates, provides audit steps. | Type constraints (T-Case, T-Guard) — what evidence each gate requires |
| **Orchestrator** | Full lifecycle — when to invoke adapters, continuations, archival. | fix_n with continuations, mem (store threading) |

## Skills

| Skill | Layer | Description |
|---|---|---|
| `two-gate` | Primitive | Present plan → await approval → implement → verify → commit. The canonical synchronization operator. |
| `inter-phase` | Adapter | Phase-transition ceremony: drift detection, plan patching, scope boundaries. |
| `phase-handling` | Orchestrator | Full phase lifecycle: initiate → execute → complete → inter-phase ceremony → next phase. |
| `haga-lambda` | Adapter | HAGA codebase context: repo layout, test commands, constants. |

## From the λA Paper

The paper validates this approach at scale. Key findings we apply:

- **Static lint from operational semantics.** λA derives 26 lint rules that catch structural defects before execution. 94.1% of 835 production GitHub agent configs were structurally incomplete under λA semantics.
- **Type safety.** A simply typed calculus (Str, Str→Str, variant, refinement) suffices for agent composition. Skills map to typed terms; composition operators (», case, guard, fix_n) have formal reduction rules.
- **Bounded fixpoint termination.** fix_n e v terminates in ≤ n unfoldings. Every loop in a skill composition has a guaranteed bound.
- **Semantic entanglement.** YAML-only lint precision is 54%; joint YAML+Python analysis raises it to 96%. For skills, this means: the lint must inspect the skill *body* (the "Python code"), not just the frontmatter (the "YAML").
- **Pipeline algebra.** Composition (») satisfies monoid laws — associativity and identity — enabling algebraic optimization of skill chains.

## Roadmap

- [ ] Type declarations in SKILL.md frontmatter (`input_type`, `output_type`)
- [ ] Effect declarations (`effects: [llm, terminal, file, ...]`)
- [ ] Composition operators formalized (», |, case, guard, fix_n)
- [ ] Static lint tool — detect missing delegation instructions, vacuous loops, type mismatches, non-exhaustive dispatch
- [ ] Bounded iteration (`bounds: {max_iterations: N}`)
- [ ] Guard/refinement — post-condition predicates on skill output
- [ ] Pipeline algebra — skill identity, associativity, optimization
- [ ] Type-and-effect system — effect tracking for composition safety

## Usage

```bash
hermes -s lambda-skills/phase-handling,lambda-skills/inter-phase,lambda-skills/two-gate
```

Loading order is **outer-first**: orchestrator first, adapter middle, primitive last. The primitive (enforcement) gets recency bias — the LLM reads the gates last, giving them the highest attention weight.

## Reference

Liu, Q. (2026). *λA: A Typed Lambda Calculus for LLM Agent Composition*. arXiv:2604.11767v2. Nanjing University.

---

Built on [Hermes Agent](https://github.com/nous-research/hermes-agent). Co-loaded skills live at `~/.hermes/skills/lambda-skills/`.
