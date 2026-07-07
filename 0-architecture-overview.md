# Hex Sum — Technical Architecture Overview

Birds-eye view of the game's system design and execution pipeline. Establishes the guidelines governing how data and control flow through the codebase.

**Status tags** (used throughout): ✅ Implemented · 🟡 Partial · 📋 Spec-only. Hedging words ("ideally", "should") mark design *flexibility*, not build status.

The **Feature Status Matrix** and the **dependency graph** are maintained in Doc A (`A-appendices.md`) as single sources of truth. This file references them rather than duplicating them.

---

## Core Design Patterns
* **State Isolation:** Each gameplay responsibility belongs to a dedicated module. Modules request data from one another but avoid mutating another module's internal state unless that interface is intentionally exposed.
* **Single Source of Truth:** Systems derive live board data from the centralized grid state (`GridManager`) rather than caching duplicate copies.
* **Communication Flow:** Structural ownership flows downward. Upstream coordination happens via loosely coupled callbacks/events.

Much of Doc X remains 📋 Spec-only. The shipped core is the selection→validation→scoring loop, its round structure, and the **memorization phase + letter identifiers**. Still unbuilt: win condition (🟡), mistake budget, mistake compensation, dynamic board updates, and solution reveal/blacklist. See Doc A for the per-feature breakdown.

---

## Directory Map
Document numbers are **file ordering** (0 = overview, 4 = visual layer), not layer-depth labels. Lettered files sit outside the numeric sequence.

* **Doc 0** (`0-architecture-overview.md`): This file. Execution flow, layer model, guardrails.
* **Doc 1** (`1-orchestrators.md`): Composition root (`LocalScript`) + runtime managers (`RoundManager`, `InputHandler`).
* **Doc 2** (`2-state.md`): State modules (`GridManager`, `SelectionManager`).
* **Doc 3** (`3-utils.md`): Logic processors (`AnsChecker`, `TimerTicker`, `TargetGenerator`).
* **Doc 4** (`4-presentation.md`): Visual layer (`HexAnimator`, `DisplayHandler`).
* **Doc A** (`A-appendices.md`): Feature Status Matrix + dependency graph.
* **Doc H\*** (`H*.md`): Critical context from previous chats.
* **Doc X** (`X-game-design-spec.md`): Full game design spec (target state).

---

## Execution Flow
Steps marked 📋 are spec-only. `LocalScript.startGame()` drives the entry: `initAll()` (every module's `init`/`reset`, the single startup-state owner) → `createGrid()` → `startMemorization()`, whose timer chains into the first round.

```text
initAll() seeds all module state (LocalScript)                 [✅]
   ↓
Board generated (GridManager.createGrid)                       [✅]
   ↓
Memorization phase (30s exposed values → letters)              [✅]
   ↓
RoundManager starts round → TargetGenerator produces target    [✅]
   ↓
InputHandler captures clicks → SelectionManager updates state  [✅]
   ↓
Third hex → 1s confirmation window                             [✅]
   ↓
Window expires → AnsChecker validates                          [✅]
   ↓
RoundManager processes result → DisplayHandler / HexAnimator   [✅]
   ↓
Next round (no win check — loops indefinitely)                 [🟡]
```

---

## Dependency Model
Top-down hierarchy: orchestrators delegate downward to base modules; the `require()` graph is acyclic. **Graph and edge list live in Doc A.**

**Acyclicity is load-bearing.** `InputHandler` requires `RoundManager`, but `RoundManager` reaches back only through a callback injected by `LocalScript` — never a `require`. That single inversion is why the graph has no cycle. Making it a direct `require` would create one.

**Layer depth** (bottom-up; matches the FigJam legend):

| Layer | Name | Members |
| :-- | :-- | :-- |
| 1 | Base Modules | GridManager, SelectionManager, TimerTicker, HexAnimator* |
| 2 | Dependent Utils | AnsChecker, TargetGenerator, DisplayHandler |
| 3 | Input & Control | InputHandler |
| 4 | Orchestrators | RoundManager, LocalScript |

\*`HexAnimator` depends only on the engine's `TweenService`, not a project module. Layer (dependency depth) and category (`state`/`util`/`presentation`) are orthogonal.

---

## Architectural Guidelines
1. `GridManager` is the reference point for coordinates and cell values.
2. `SelectionManager` tracks highlighted instances and active selection coordinates.
3. `AnsChecker` owns rule + arithmetic validation. It reads board values from `GridManager` but holds no gameplay-loop state (no score/round/timer). Node-count is enforced upstream in `InputHandler` (Doc 3).
4. `HexAnimator` and `DisplayHandler` consume data to produce visuals; logic stays with orchestrators.
5. `TimerTicker` is gameplay-agnostic — the one util requiring no project module.

---

## Blueprint for Future Systems
📋 Spec-only. Starting patterns for unbuilt mechanics.

* **Dynamic Board Updates:** Mutate the grid reference in `GridManager`; dispatch a state-change event so `DisplayHandler`/`HexAnimator` reflect it.
* **Mistake Allowance:** Track in `RoundManager`. Manages the single-mistake budget, shifts victory 5→6 rounds, adjusts `TimerTicker` (+5s), reduces board-update intensity.
* **Solution Reveal & Target Filtering:** On error, `AnsChecker` surfaces the correct line; `TargetGenerator` filters it from the next target pool.
