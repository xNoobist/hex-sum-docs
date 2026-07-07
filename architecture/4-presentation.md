# Presentation Layer (Visual & Feedback)

Presentation modules render UI and feedback from data supplied by orchestrators.

## HexAnimator
Converts gameplay events into UI tweens. Consumes the `{Part,…}` instance array from `SelectionManager` (or `GridManager.allHexes` for group effects) directly, so it never does coordinate math. Rendering only.

* **`tween(tweenType, hex)`** — single-hex effects: `deselect`, `select`, `hover`. Each drives a colour tween + a size tween off the hex's `baseSize` attribute.
* **`tweenGroup(tweenType, hexes)`** — whole-group effects: `correct`, `incorrect`, `overtime`, `memorization`. `overtime` and `memorization` early-branch and run their own loop; `correct`/`incorrect` share a trailing loop.

Colour set lives in the `hexColours` table (`base`, `selected`, `correct`, `incorrect`, `hovered`, `memorization`, `black`). The `memorization` goal is near-white with a slight shrink (×0.95); it is reverted externally by `RoundManager`'s memorization-end loop (via a `deselect` tween), not from inside the animator.

**Font colour is not owned here.** `HexAnimator` tweens *part* colour only. The memorization *font* colour flip lives in `RoundManager` as a direct `TextColor3` write — a known boundary split (Doc A, and H2 §5).

---

## DisplayHandler
A small **state-plus-render engine** for the two peripheral text labels (`upper`: score/timer; `lower`: target). State tables hold *what to show*; one function decides *how to paint it*.

### Structure
Each label owns a state table (`upperDisplay`, `lowerDisplay`) with a `currState` field and one sub-table per state (`defaultData`, `scoreData`, `checkData`, `memorizationData`). **Each state sub-table carries its own `getProjectedDisplay(self)` formatter**, so adding a state is a local addition, not a new branch in the render path. Formatting happens at render time from stored fields — this is what keeps numbers as numbers upstream and is why the old `math.min` string-coercion fragility is gone.

### Flow (single writer)
```text
modifyDisplay(displayContext, contextArgs)   -- public router
   |  (routes on displayContext[1]: "upper" | "lower")
modify*State(directedState, contextArgs)     -- mutates state table only
   |
updateDisplay(directedDisplay)               -- sole .Text writer
```
`modifyDisplay` uses `if "upper" … elseif "lower"` — an unknown display key is a safe no-op.

### State priority (score overlay)
`modifyUpperState` guards: if `currState == "score"` and a non-score update arrives, it returns early — the score-gain alert holds visual priority over timer ticks. The score state sets a `scoreID`, then a 1s `task.delay` restores `default` only if no newer score-gain has superseded it (the **stale-drop guard**, same pattern as `InputHandler`'s `submissionID`). Note `scoreData` holds only `scoreGain`; the score *value* lives in `defaultData.score`, so the score state is an overlay on default, not self-contained.

### Init
`DisplayHandler.init()` re-seeds every state field (`scoreID`, both `currState`s, all data fields) to zero/`default`. Called by `LocalScript.initAll()`. The module no longer paints or self-inits at load; first paint is driven by `startMemorization()`.

**Known live bugs (see H2 §2):** the score-restore path passes the state *table* to `updateDisplay` instead of the `"upper"` string, cross-painting the lower label (masked by 60fps timer repaints); and `updateDisplay`'s `else` branch is fail-silent where `modifyDisplay`'s dispatch is hardened.
