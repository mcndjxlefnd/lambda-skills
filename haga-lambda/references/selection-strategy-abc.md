# ISelection/IReproduction ABC: Population-Aligned Architecture

No padding, no truncation. Population size is determined by IPopulationSize
and selection+reproduction are balanced to hit it exactly.

## Ownership Split

- **IPopulationSize** owns: the target population N (via `compute()`).
- **ISelection** owns: how many parents (`num_parents(N) -> int`).
- **IReproduction** owns: pairing pattern, sigma inheritance. Accepts `N`
  as target offspring count — must produce exactly N rows.
- **Pipeline** owns: nothing. Passes N through, asserts offspring
  count matches. No padding, no truncation.

## ABC Signatures

```python
class ISelection(ABC):
    @abstractmethod
    def select(self, genotype: Genotype) -> np.ndarray: ...
    @abstractmethod
    def num_parents(self, N: int) -> int: ...

class IReproduction(ABC):
    @abstractmethod
    def reproduce(self, n_parents: int, N: int, rng: IRNG) -> np.ndarray: ...
```

## Alignment Guarantee

`SelectionAlignedScaling` wraps `SqrtScaling` to ensure N is a multiple of
parent_count: `N = ceil(N_policy / parent_count) * parent_count`. This
guarantees reproduction can fill N exactly with integer pair cycles.

`parent_count` is set in entrypoint.py after loading selection:
`population_size.parent_count = selection.num_parents(N_policy)`.

## Implementation Notes

- `num_parents()` is called by pipeline to verify, but also at init time
  by entrypoint to align N with SelectionAlignedScaling.
- `StriatedReproduction` cycles column-split pairs to fill exactly N
  offspring. Pairs repeat modulo mid as needed.
- `TrivialStubSelection.num_parents(N)` returns `N // 2` (standard 50%).
- No `offspring_per_parent` anywhere — it's derived, not stored.

## Edge Cases

### N must be multiple of 4

`StriatedReproduction.reproduce()` writes 4 offspring per pair
(`offspring_map[idx..idx+3]`). If N is not divisible by 4, the loop
overshoots the array → `IndexError`. `SelectionAlignedScaling` rounds
to `parent_count` multiples, but that alone doesn't guarantee 4-divisibility
(e.g. `parent_count=11` → N=121). **Fix:** post-round to multiple of 4:

```python
N = ((N_policy + p - 1) // p) * p
N = ((N + 3) // 4) * 4          # added
```

Baked into `SelectionAlignedScaling.compute()`.

### mid=0 with small populations

When N ≤ 9 and Top10Selection selects `max(1, N//10) = 1` parent,
`mid = n_parents // 2 = 0` → `ZeroDivisionError`. **Fix:** clamp mid ≥ 1
and bound `right` to `n_parents - 1`:

```python
mid = max(1, n_parents // 2)
right = min(mid + pair, n_parents - 1)
```

Applied in `StriatedReproduction.reproduce()`.

## Anti-Patterns

- **Padding with best-parent clones dilutes selection signal.** Never add
  non-reproduced individuals to fill a gap between reproduction output
  and population size.
- **Truncation discards unevaluated offspring.** For f=0.75 with N=200,
  P=150 produces 300 offspring with O=2; truncating to 200 throws away 100
  unevaluated offspring. Fix: `SelectionAlignedScaling` rounds N up to 300
  so all offspring are evaluated and participate in selection.
- **`offspring_per_parent` as a class attribute.** Forces a
  one-size-fits-all value. Instead, let `IPopulationSize` determine N,
  then reproduction fills N by cycling pairs. The offspring-per-parent
  value is implicit in the math, not stored on any strategy.
