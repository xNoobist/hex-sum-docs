# Hex Sum — Game Design Specification

Target-state design. Written in present tense as the finished game's intended behavior — **not** current build state. Sections carry a status tag; Doc A holds the authoritative matrix. Roughly 30% here is 🟡 Partial or 📋 Spec-only.

## Overview
A mental arithmetic and memory game under time pressure. Players memorize a grid of hidden numbers, then solve target-sum challenges while the board mutates after each cleared round.

---

## Objective & Victory 🟡
Clear **5 rounds** by finding lines of three hexagons whose hidden values sum to a displayed target. **One mistake** is permitted: it doesn't end the game but raises the requirement to 6 rounds. A second mistake, or the timer hitting zero, is game over.

---

## Grid & Letter Identifiers ✅
The board is **19 hexagons** in a large hexagon, each holding a hidden value 2–10.

Every hex gets a permanent letter identifier, mapped sequentially, with **"I" and "O" omitted** to avoid confusion during fast calculation.

---

## Lifecycle ✅

### Memorization Phase
At start, every hex shows its value. The player gets **30 seconds** to memorize (no visible countdown). When time expires, numbers vanish and are replaced permanently by letter identifiers.

### Standard Round
Each round shows a new target. The player selects three hexes, opening a 1-second confirmation window; clicking a selected hex during it deselects and cancels, guarding against misclicks. If unchanged when the window expires, the selection auto-submits. Round limit: **15 seconds**.

---

## Answer Requirements ✅
A clearing submission must satisfy all four:
* **Quantity:** exactly three hexes.
* **Connectivity:** all three mutually adjacent.
* **Alignment:** a straight line along one grid axis.
* **Arithmetic:** hidden values sum to the target.

(Quantity is enforced in `InputHandler`; connectivity/alignment/arithmetic in `AnsChecker` — see Doc 3.)

---

## Invalid Answers vs. Mistakes
* **Invalid** ✅ — breaks a layout rule (disconnected, non-straight, wrong count). Rejected without penalty; doesn't count against the mistake limit.
* **Mistake** 🟡 — a valid straight line of three whose values miss the target. On a mistake: the round isn't cleared, the actual line sum is revealed, and the correct solution is briefly shown then temporarily blacklisted from being the next target. *(Detection exists; reveal + blacklist are 📋.)*

---

## Dynamic Board Modification 📋

### Post-Success Updates
After each cleared round, **3 hexes** are randomly chosen to change value:
* Each shifts by **-3 to +3** (excluding 0).
* Values are capped to 2–10.
* For that round, the change is shown on the affected hexes, temporarily replacing their letters so the player can update their mental map.

### Mistake Compensation
Applies only to the specific uncleared round where a mistake occurred:
* Round timer extends 15 → **20 seconds**.
* Fewer hexes are modified, easing memory load.

Once that round is cleared, compensation lapses and rounds return to 15s / 3 modified hexes.
