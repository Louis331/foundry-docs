---
---
## Plan

---

[Home](index)

## Table of Contents

1. [What's Done](#whats-done)
2. [What's Next](#whats-next)

---

## What's Done

### Foundation
- [x] Player movement + camera follow
- [x] Checkerboard `GridRenderer` with viewport culling
- [x] `GridWorld` with atomic place/remove and coordinate helpers
- [x] Scene rename from `GridTest` → `World`

### Placeable System
- [x] `Placeable` base class (simulation, no Node dependency)
- [x] `PlaceableDefinition` with JSON loading
- [x] `PlaceableRegistry` Autoload
- [x] `JsonLoader` helper + `TupleConverter` for `(int, int)` tuples
- [x] `iron_ore.json` definition + pixel art sprite
- [x] `ResourceNode` subclass with `Hardness`

### Gameplay
- [x] Left click → place ore
- [x] Right click (held) → mine with progress
- [x] Middle click → remove (dev tool)
- [x] Proximity check before mining (`MiningRange`)
- [x] Mining duration: `hardness / miningSpeed`
- [x] `MiningOverlay` visual (dark rect, bottom-up fill)
- [x] `placedNodes` dictionary (replaced fragile float position lookup)
- [x] Null guard on `activeMining` to prevent crash on fast mouse movement

### Tooling
- [x] `.editorconfig` with 4-space indentation

- [x] **`World.cs` refactor** - extract player input/mining logic into a `PlayerController` class. `World` shouldn't own input handling.
- [x] `GetMousePositionInGrid()` → make `public` on `World`, call from `PlayerController`

---

## What's Next

### Registries (planned as one unit, in dependency order)
- [ ] **Item registry** - items are the leaf dependency; machines and recipes reference them. Stable string IDs, JSON-driven.
- [ ] **Machine registry** - references items (I/O slots, valid inputs). Includes fixed-timestep `Tick` driving.
- [ ] **Recipe registry** - references both items and machines.

## Simple Ui
- [ ] **A basic HUD**
- [ ] **A simple invetory system** - this will be for a main invetroy and scroll bar

> The tick system will be folded into the machine registry phase rather than treated as a separate refactor step.