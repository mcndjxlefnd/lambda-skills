# TripleIndependentSigma Mutation Intensity Dead Zone

## Discovery

lin318 run (`lin318_2026-05-30-155802.db`) showed `best_mut_intensity = 1.0` and
`mean_mut_intensity = 1.0` for every generation (0–1000) on D0 (root master, 318
cities). D1 showed the same. D2+ saw only rare movement.

## Root Cause

In `src/haga/strategies/adaptation/triple_independent_sigma.py`, the adaptation:

```
tau = 1.0 / sqrt(M)            # M = local_city_count
multiplier = exp(tau * N(0,1))
new_mi = int(round(mut_intensity * multiplier))   # clamped to [1, M//2]
```

Starting from `init_sigma = 1.0` (from `TrivialStubSigmaInit`), the issue is:

- `new_mi = int(round(1.0 * exp(tau * noise)))` needs `multiplier >= 1.5` to reach 2
- This requires `exp(tau * noise) >= 1.5`, so `noise >= ln(1.5) / tau`
- For D0 (M=318): tau=0.056, threshold noise ≈ 7.2σ — impossible from N(0,1)
- For D1 (M≈159): tau=0.079, threshold noise ≈ 5.1σ — also impossible
- For D2 (M≈80): tau=0.112, threshold noise ≈ 3.6σ — rare in 1000 gens
- For D3 (M≈40): tau=0.158, threshold noise ≈ 2.6σ — occasional
- For D4 (M≈20): tau=0.224, threshold noise ≈ 1.8σ — sometimes
- For D5 (M≈10): tau=0.316, threshold noise ≈ 1.3σ — possible but not common

Even when multiplier pushes past threshold, `int(round())` quantizes to integers,
so intensity jumps in steps of 1.0 with no fractional values — losing granularity.

## Implications

- **Large M adaptation is dead.** Root master's mut_intensity is effectively frozen
  for any instance > ~100 cities.
- **The clamping `max(1, ...)` prevents it from ever going below 1.**
- **Mutation rate (mut_rate) and crossover rate (cross_rate)** don't have this problem
  because they're clamped to continuous `[0.01, 1.0]` — they adapt fine.
- **This is specific to `int(round())`** on the intensity. If intensity were a float
  (like rate parameters), it would adapt correctly.

## Experimental Results (lin318, D0, 1000 gens)

All runs on lin318.tsp, 1000 gens, default config, dashboard@8a169db.
Five variants survived after deleting intermediate clamped runs that were
replaced by noclamp/loclamp equivalents. Complete experiment preserved at
`2026-05-30-mi-sweep/` in the experiments repo.

### Baseline (original code — frozen)

| Metric | Value |
|--------|-------|
| DB | `lin318_2026-05-30-155802.db` |
| Tour length | 65,440.55 |
| Distinct intensities (D0) | **1** (always 1.0) |
| Max best_intensity | 1.0 |
| Mean intensity range | 1.0 → 1.0 |

### No-Clamp Variants (no upper, no lower)

Both collapse — selection pressure crushes intensity to near-zero without
a lower bound. Larger tau_mi accelerates the collapse.

| Variant | DB | Tour | Max best mi | Mean mi @ 1000 |
|---------|-----|------|-------------|----------------|
| Opt1 (tau=0.056) | `lin318-opt1-noclamp_2026-05-30-162342.db` | 64,624 | 2.5 | **0.019** |
| Opt2 (tau_mi=0.183) | `lin318-opt2-noclamp_2026-05-30-162553.db` | 66,319 | 0.96 | **1.9e-07** |

Mechanism: log-normal drift is symmetric (median multiplier = 1.0), but
selection is asymmetric — lower intensity means fewer swaps, less
tour disruption, higher fitness. Once an individual dips below 1.0,
there's no pressure to recover. The population converges to epsilon.
`AdjacentSwap` still forces 1 swap at usage, so the GA doesn't break,
but self-adaptation is dead.

### Lower-Clamp-Only Variants (lower clamp at 1.0, no upper bound)

The lower clamp prevents collapse. Decoupled tau_mi provides real step size.

| Variant | DB | Tour | Max best mi | Mean mi range |
|---------|-----|------|-------------|---------------|
| Opt1 (tau=0.056) | `lin318-opt1-loclamp_2026-05-30-163215.db` | 65,687 | 2.5 | 1.04 → 1.08 |
| **Opt2 (tau_mi=0.183)** | `lin318-opt2-loclamp_2026-05-30-163424.db` | **66,483** | **9.9** | **1.11 → 2.65** |

Opt1 is alive but sluggish — tau=0.056 means very slow upward drift.
Opt2 actually self-adapts: mean intensity reaches 2.65, best individual
hits 9.87 swaps. Removing the upper clamp is irrelevant here (M//2=159).

### Full Matrix (lin318 D0, 1000 gens)

| Variant | tau_mi | Clamps | Format | Tour | Max best mi | Mean mi @ 1000 |
|---------|--------|--------|--------|------|-------------|----------------|
| Baseline | 0.056 | both | int | 65,441 | **1.0** | 1.0 |
| Opt1 noclamp | 0.056 | none | float | 64,624 | 2.5 | 0.019 |
| Opt2 noclamp | 0.183 | none | float | 66,319 | 0.96 | 1.9e-07 |
| Opt1 loclamp | 0.056 | lower | float | 65,687 | 2.5 | 1.08 |
| **Opt2 loclamp** | **0.183** | **lower** | **float** | **66,483** | **9.9** | **2.65** |

### Key Conclusions

1. **`int(round())` is the primary bug.** Float format allows continuous drift.
2. **Lower clamp is necessary.** Without it, selection collapses mi to zero.
3. **tau must be decoupled for mi.** `tau_mi = 1/sqrt(min(M,30))` gives 3.3x
   larger steps than shared tau for M=318.
4. **Upper clamp never binding.** For lin318, M//2=159, never approached.
5. **Tour length converges regardless.** All variants produce tours in
   64.6k–66.5k range — selection pressure dominates over mutation strategy.
6. **Open question:** is K=30 the right ceiling for decoupled tau? Smaller K
   gives larger tau → faster adaptation. Needs calibration.

## Possible Fixes (ordered by minimality)

1. **Float only** — remove `int(round())`. Alive but sluggish for large M.
2. **Decoupled tau_mi** — `tau_mi = 1/sqrt(min(M,30))`. Real adaptation on
   experiment timescales.
3. **Lower clamp is mandatory** — `max(1.0, ...)`. Without it, collapse.
4. **Higher init sigma** — start mi at `M // 50` instead of 1.0 so it has
   room to drift both directions.
5. **Scale-independent sigma** — constant sigma_c instead of 1/sqrt(M),
   removing problem-size dependence entirely.
