# Population Sizing Edge Cases

## Bug 1: SelectionAlignedScaling Breaks N % 4 == 0

`SelectionAlignedScaling` rounds N up to the nearest multiple of `parent_count`.
But `StriatedReproduction.reproduce()` expects N to be a multiple of 4
(4 offspring per parent pair). When `parent_count` is not a multiple of 4,
the rounded N fails this constraint.

**Example:** M=100, N_policy=112, parent_count=11 → N=121 → IndexError
(loop overshoots offspring_map at idx+3).

**Fix (selection_aligned_scaling.py `compute()`):** after rounding to
`parent_count` multiple, also round to multiple of 4:
```python
N = ((N_policy + p - 1) // p) * p
return ((N + 3) // 4) * 4
```

## Bug 2: StriatedReproduction Crashes When n_parents < 2

`StriatedReproduction.reproduce()` computes `mid = n_parents // 2`. When
`n_parents` is 0 or 1, `mid = 0`, causing `ZeroDivisionError` at
`(idx // 4) % mid`.

This occurs when population size N is very small (≤9 with Top10Selection;
`select()` returns `max(1, N // 10)` = 1 parent). Deep leaf processes with
small `local_city_count` are the typical trigger.

**Fix (striated_reproduction.py `reproduce()`):** guard `mid` and clamp
`right`:
```python
mid = max(1, n_parents // 2)
right = min(mid + pair, n_parents - 1)
```

## Bug 3: TripleIndependentSigma Dead Zone (Mutation Intensity Frozen)

`TripleIndependentSigma.adapt()` uses `tau = 1.0 / sqrt(M)` and rounds
`mut_intensity` to int. For M=318, tau≈0.056, the log-normal multiplier
is ~0.90-1.12 for typical noise, so `int(round(1.0 * (0.90...1.12)))` is
always 1. Every generation stays at mut_intensity=1.0 regardless of
selection pressure or sigma adaptation.

This makes selection-fraction experiments produce identical results —
the exploration rate is frozen regardless of parent pool size.

**Fix exists on `elite-delete` branch (commit `021b03a`):**
- Decouple `tau_mi = 1.0 / sqrt(min(M, 30))` — meaningful step size at large M
- Remove `int(round())` — continuous float
- Remove upper clamp — keep only lower clamp at 1.0

**Files:** `src/haga/strategies/adaptation/triple_independent_sigma.py`,
`tests/unit/strategies/test_triple_independent_sigma.py`
