# Orchestrators (Runtime Controllers)

Orchestrators wire the system together, capture inputs, track game state, and delegate specialized work downward. Ordered wiring-first: the composition root before the modules it wires.

---

## LocalScript
The **composition root**. First thing to run; establishes every cross-module wire, then starts the game.

* **Dependency Injection:** Loads orchestrators and state systems and maps their callbacks — including the `RoundManager → InputHandler` callbacks (`onNextRound`, `onTimerEnd`, `getCheckTimerState`, `allowClicking`) that keep the dependency graph acyclic (Doc A → A2). Without them the two orchestrators would `require` each other and cycle.
* **`initAll()`:** Single visible owner of startup state. Calls `GridManager.init` · `InputHandler.init` · `RoundManager.init` · `SelectionManager.reset` · `DisplayHandler.init`. No module self-inits at load anymore.
* **`startGame()`:** `initAll()` → `GridManager.createGrid(...)` (passing the input callbacks) → `RoundManager.startMemorization()`. The memorization timer's `onEnd` chains into the first `startRound()`.

Kept stateless: it binds modules, seeds init, and starts the loop. Its role is structural.

---

## RoundManager
Coordinates progression, scores, and round-state transitions. Requires `GridManager`, `TimerTicker`, `HexAnimator`, `DisplayHandler`, `TargetGenerator`.

* **State (`RoundManager.init` seeds all):** `currRound` (local), `roundActive`, `targetSum`, `score`.
* **Two timers:**
  * `memorizationTimer` (30s, no tick): on end, swaps every hex value→letter, seeds peripheral displays to default, reverts the memorization part-tween (`deselect`), flips hex font white, arms clicking, sets `roundActive`, and calls `startRound()`.
  * `roundTimer` (15s): per-tick, formats `remainingTime` to a number and drives the upper display — `check` state during a confirmation window (showing `math.min` of round vs. check time), else `default`. On end, fires `onTimerEnd`.

`currRound` increments in `startRound()` but is **not yet read by any win check** — win condition is 🟡 (Doc A).

### Scoring
Score is floored and scales the timer value at selection-lock onto a 0–1000 range:

$$\text{scoreGain} = \left\lfloor 1000 \times \frac{\text{ansTime}}{\text{maxTime}} \right\rfloor$$

`ansTime` is `roundTimer.remainingTime` snapshotted by `InputHandler` when the third hex locks (passed as `lockedTime`), not the value at evaluation. The 1s confirmation window elapses between those points, so a player is scored on when they *committed*.

---

## InputHandler
Bridges physical input (clicks/touches) to gameplay logic. Also the **node-count gate**: additions are blocked past `count == 3`, and submission fires only at exactly three — so `AnsChecker` never checks count. `init()` seeds `submissionID`, `canClick`, and the exposed check-timer state.

### Confirmation Window
```text
Click -> SelectionManager updates -> third hex selected
   |
Snapshot selection + lockedTime
   |
Start 1s check timer
   |
Evaluate if selection unchanged
```

### Race & Edge-Case Guards
* **`lockedCount`:** count snapshotted at lock; a deselection before the window closes aborts the submission safely.
* **`submissionID`:** board interactions increment it; a pending evaluation whose `storedID` no longer matches is discarded.
* **Timer-expiry priority:** `handleTimerEnd` — if the round timer hits zero with exactly three selected, evaluation fires immediately. **Otherwise (fewer than three), it currently runs the overtime tween and *resumes play*** — there is no game-over-on-timeout yet (spec wants one; see H2 §3).

This section is the strongest-specified part of the codebase.
