# Utilities (Logic Processors)

Utility modules compute over their inputs. Two of the three (`AnsChecker`, `TargetGenerator`) `require` `GridManager` and read board state — they're Layer-2 dependent utils. Only `TimerTicker` is dependency-free. "Decoupled" here means decoupled from the gameplay *loop* (no score/round/timer state), not from the grid.

## AnsChecker
Validates submitted paths on two criteria. It `require`s `GridManager` and reads `grid[...].value` inside `isCorrect` — it depends on live grid state for values.

* **`isValid(ans)`:** Structural rules — connectivity and straight-line. Failures are invalid inputs and don't consume the mistake budget.
* **`isCorrect(ans, target)`:** Arithmetic — whether the selected values sum to the target. Failures are genuine mistakes.

**Node-count lives in `InputHandler`, not here.** `isValid` runs connectivity + straight-line only; the exactly-three guarantee is enforced upstream via `lockedCount`. Consequence: don't call `isValid` directly with a non-3 set — it won't reject on count, since it assumes the caller already guaranteed three.

### Geometry
* **Connectivity (BFS):** every selected hex must be reachable from the first via selected-neighbour hops.
* **Straight-line:** via cube coordinates. The first two nodes fix which axis ($q$, $r$, or $s$) stays constant; the rest must preserve it.

---

## TimerTicker
Dependency-free countdown. Isolated from scoring, rounds, and input; communicates via callbacks:
```lua
timer.onTick(remainingTime) -- per-frame display updates
timer.onEnd()               -- fires on expiry
```

---

## TargetGenerator
Produces the per-round target sum. `require`s `GridManager` and samples the board so targets are always solvable on the current grid.

### Pipeline
1. **Frequency analysis:** iterate the `ALL_LINES` table (27 fixed three-node lines), sum each over current grid values, and count how many lines produce each sum.
2. **Weighted lottery:** bucket sums by their line-count frequency (1–4), pick a frequency bucket by weight, then pick a sum uniformly from that bucket.

`ALL_LINES` is hardcoded because the radius-2, 19-hex topology is invariant — the line set never changes, so there's no reason to re-derive it each round.

### Weighting
Each frequency bucket $f \in \{1,2,3,4\}$ has a base weight `{113, 74, 43, 20}`, scaled by a multiplier $M$ based on bucket size $l$ (the number of distinct sums in that bucket):

$$\text{weight}(f) = \text{base}(f) \times M, \qquad M = 1 + \frac{a(l-1)}{l+b}, \quad a=6,\ b=2$$

* $l = 1 \Rightarrow M = 1$ (a singleton bucket gets no boost).
* As $l \to \infty$, $M \to 1 + a = 7$. So $a$ sets the ceiling; $b$ tunes how fast it's approached without moving the ceiling.

The multiplier keeps large buckets from drowning out rare ones while still weighting by commonality. If no bucket is selectable, the target falls back to a **single** `math.random(6, 30)` — a guard whose trigger condition is astronomically rare on a real grid (~2×10⁻¹²). There is no nested or "squared" fallback beyond this one line.
