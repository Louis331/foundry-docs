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

### Refactors
- [x] **`World.cs` refactor** - extract player input/mining logic into a `PlayerController` class. `World` shouldn't own input handling.
- [x] `GetMousePositionInGrid()` → make `public` on `World`, call from `PlayerController`

### Registers
- [x] **Item registry** - items are the leaf dependency; machines and recipes reference them. Stable string IDs, JSON-driven.
- [x] ~~**Machine registry** - references items (I/O slots, valid inputs). Includes fixed-timestep `Tick` driving.~~ This is now being handled by placeables, Ticking has been implemented at a basic level
- [x] **Recipe registry** - references both items and machines.

---

## What's Next

### Machines
- [ ] **Make them placeable**
- [ ] **Recipes** - How do machines use recipes?
- [ ] **Simple machine ui**
- [ ] **Power system** - some machines will use steam some will use power
- [ ] **Collison with player** - players can't walk over machines

### Inventory
- [ ] **Storing items** - Picked up items will be stored in the player inventory
- [ ] **Size of inventory**
- [ ] **Inventory crafting** - Allow players to craft using items in their inventory

### Storage
- [ ] **Box/chest** - this will be used to store items get output from machines

### Pipe system
- [ ] **Item routing**
- [ ] **Liquid/gas routing**
- [ ] **Power routing**

### Simple Ui
- [ ] **A basic HUD**
- [ ] **A simple inventory system** - this will be for a main inventory and scroll bar
- [ ] **Pick up system**
- [ ] **Item stack size**

### Saving / Loading
- [ ] **Save world state**
- [ ] **Load world state**
- [ ] **Menu** - Used for loading an creating new games

### World gen
- [ ] **Load resource nodes in** - Generate world
- [ ] **Remove grid texture** - Replace with something a bit more pleasing

### Resources
- [ ] **Create copper**
- [ ] **Create coal**
- [ ] **Create sand**
- [ ] **Create clay**
- [ ] **Create water**
- [ ] **Create wood/trees**
