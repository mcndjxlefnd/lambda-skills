# Cross-Branch Audit Checklist

Template for the structured per-branch summary produced by a cross-branch
experiment audit. One entry per experiment branch, then a synthesis section.

---

## Per-Branch Entry

```
### Branch: <name>
**Fork point:** <sha> (<date>)
**Commits:** <N> unique commits above fork point
**Last modified:** <date>

**Hypothesis:** <one-line research question>

**Key findings:**
- <finding 1>
- <finding 2>

**Status:** complete | incomplete | negative-control | rerun-planned

**Document locations:**
- experiment.json: <path>
- lab-notebook: <path>
- ideas-open/closed: <path> (content: yes/no)
- logbook entry: <path>
- plans: <path>

**Cross-branch dependencies:**
- Needs cherry-pick from: <branch>@<commit>
- Fix provides to: <branch>

**Open questions:**
- <question>
```

---

## Synthesis Section

### Cross-Cutting Observations

- [ ] Common conclusions across experiments
- [ ] Naming/location inconsistencies
- [ ] Fix gaps between branches
- [ ] Incomplete experiments

### Branch Dependency Map

`<branch A>` (fix X) → needs cherry-pick to → `<branch B>`

### Next-Experiment Recommendations

- What's the highest-leverage next question?
- What fix must be included before the next run?
