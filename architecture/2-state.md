# State Management (Data Layer)

State modules hold and expose the game's primary data models — the internal source of truth. Both are Layer-1 base modules (no project-module dependencies).

## GridManager
Instantiates and maintains the hex grid. `init()` clears `grid` and `allHexes` for replay; `createGrid()` builds the 19 hexes and their per-hex `SurfaceGui`/`TextLabel`.

### Coordinate Space
19 hexagons on **axial coordinates** $(q, r)$; cube coordinate $s = -q - r$. Cube form enables straight-line checks without row/column indexing.

### Grid Structure
Exposed **by reference**:
```lua
grid[q][r] = {
    part = hexInstance,   -- Roblox Instance
    value = value,        -- integer 2-10
    valueDiff = 0,        -- pending change amount (dynamic board updates)
    letter = letter,      -- permanent identifier
    displayType = "none"  -- "value" | "letter" | "change" | "none"
}
```
Because the table is passed by reference, updates are visible to all querying subsystems without manual sync — this backs the single-source-of-truth principle (e.g. `AnsChecker` reads `grid[q][r].value` directly). The `valueDiff` + `displayType == "change"` path in `updateHexDisplay` is **existing partial scaffolding** for dynamic board updates (Doc X), not yet driven.

### Letter Identifiers
`AVAILABLE_LETTERS` is a **hand-ordered** 19-entry set. The ordering is deliberately non-alphabetical to counter creation order (`q` iterates -2->2, bottom-to-top), so letters paint in reading order. **Do not alphabetize it** — that reintroduces the reversed-letter bug. (Spec wants I/O omitted; the current set is hand-picked and does not yet enforce that rule.)

### Display & Utilities
* `modifyHexDisplay(coords, displayType)`: sets a hex's `displayType` and repaints its label. Stays in `GridManager` (grid-out decision, final).
* `coordsToHex()` / `hexToCoords()`: translate between spatial position and grid keys.
* `coordKey()`: format a coordinate as `"q,r"`.
* `getNeighbours()`: adjacent coordinates for a node.

---

## SelectionManager
Tracks the active selection, separate from grid data. `reset()` clears all structures (called on init and after each submission). Held in **three parallel structures**, each shaped for one consumer:

1. **Coordinate array** `{{q,r},…}` — validation-facing. `AnsChecker` does geometry/arithmetic on raw coords.
2. **Instance array** `{Part,…}` — visual-facing. `HexAnimator` tweens `Part`s directly.
3. **Lookup table** `lookup["q,r"] = true` — membership set for "already selected?" checks.

Plus a `count` integer. **Cost:** every mutation must update all structures in lockstep. `remove()` runs a dual linear-scan splice (coordinate + instance arrays) plus a lookup-key delete. Trivial at $n \le 3$, but the structures must never be updated independently — a partial update desyncs the selection.
