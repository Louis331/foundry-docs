---
---

# Foundry — Project Documentation

> Factory automation game · Godot 4.5 .NET · C#  
> Last updated: June 2026

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
   - [Simulation / Presentation Split](#simulation--presentation-split)
   - [Core Classes](#core-classes)
   - [Scene Structure](#scene-structure)
3. [What's Done](#whats-done)
4. [What's Next](#whats-next)
5. [Architecture Decision Records (ADRs)](#architecture-decision-records)

---

## Project Overview

Foundry is a 2D top-down factory automation game. The player manually mines raw resources, places machines, and eventually builds automated production pipelines.

**Technology:**
- Godot 4.5 .NET
- C# (primary language)
- `System.Text.Json` for data loading
- IDE: Cursor

---

## Architecture

### Simulation / Presentation Split

The most important architectural principle in the codebase. **Simulation and rendering are intentionally separated.**

```
Simulation layer          Presentation layer
──────────────            ──────────────────
Plain C# classes          Godot Nodes / Scenes
No scene tree deps        References simulation data
Owns game state           Knows how to draw it
Tickable                  No logic, just display
```

The rule: **data knows nothing about nodes. Nodes know their data.**

This pays off for save/load, simulation integrity, and future performance with 500+ machines.

---

### Core Classes

#### `Placeable` (plain C# class)
Base class for all objects that exist in the simulation.

| Property | Type | Description |
|---|---|---|
| `GridPosition` | `Vector2I` | Cell the object occupies |
| `Id` | `string` | Stable string key, matches registry |

- Has a virtual `Tick()` method — overridden by subclasses for machine behaviour
- Not a Node — no scene tree dependency whatsoever

**Subclasses:**
- `ResourceNode` — mineable resource (iron ore, etc.) — has `Hardness`

---

#### `PlaceableDefinition` (plain C# class)
Static data for a type of placeable, loaded from JSON. One definition per *type*, many `Placeable` instances.

| Property | Type | Description |
|---|---|---|
| `id` | `string` | Stable string key |
| `name` | `string` | Display name |
| `sprite` | `string` | Path to sprite asset |
| `size` | `(int X, int Y)` | Grid footprint |
| `type` | `string` | Maps to a C# subclass |
| `hardness` | `float` | Mine time multiplier (default `1.0`) |

JSON lives at: `res://Objects/Data/Placeables/*.json`  
Example: `iron_ore.json`

---

#### `PlaceableRegistry` (Autoload Node)
Loads all `PlaceableDefinition` JSON files at startup. Globally accessible via Godot Autoload.

- Registered under: Project → Project Settings → Globals → Autoloads
- Uses `JsonLoader` + `TupleConverter` helpers for deserialisation
- Lookup: `PlaceableRegistry.Get("iron_ore")` → `PlaceableDefinition`

---

#### `GridWorld` (plain C# class)
Authoritative simulation state. Owns the occupancy map.

| Method | Description |
|---|---|
| `PlaceInWorld(Vector2I)` | Atomic check-and-commit placement |
| `RemoveFromWorld(Vector2I)` | Atomic removal |
| `WorldToGrid(float x, float y)` | Converts world position to grid cell |
| `GridToWorld(int x, int y)` | Returns cell centre in world coords |
| `GetCellRect(int x, int y)` | Returns full cell Rect2 |

- Dictionary: `Dictionary<Vector2I, Placeable>` — the ground truth of what's placed where

---

#### `PlaceableNode` (Godot Node2D)
Presentation layer for a single placed object.

- Holds a reference to its `Placeable` instance
- Delegates `MiningProgress` down to `MiningOverlay`
- Knows nothing about `World.cs` or game logic
- Child hierarchy: `Sprite2D` → `MiningOverlay`

---

#### `MiningOverlay` (Godot Node2D)
Visual overlay that shows mining progress on a `PlaceableNode`.

- Has a `MiningProgress` property — setter calls `QueueRedraw()`
- `_Draw` renders a dark rect growing from the bottom of the cell upward

---

#### `World` (Godot Node — main scene coordinator)
Previously named `GridTest`. The scene coordinator / presenter — not a simulation class.

- Holds `GridWorld` instance
- Handles player input: left click = place, right click = mine, middle click = remove (dev tool)
- Tracks mining state via `(float Timer, Placeable Target)? activeMining` nullable tuple
- Holds `Dictionary<Vector2I, PlaceableNode>` (`placedNodes`) for fast node lookup
- Mining logic: held right-click, proximity check, `hardness / miningSpeed` duration

---

#### `Player` (Godot CharacterBody2D)
- `MiningRange` — exported field, set in Inspector
- `MiningSpeed` — encapsulated property
- Owns a `Camera2D` as a child (camera follows player)

---

#### `GridRenderer` (Godot Node2D)
- Draws the checkerboard grid using `_Draw`
- Viewport-culled, camera-aware via `GetCanvasTransform().AffineInverse()`
- Calls `QueueRedraw()` in `_Process` so it updates as the camera moves

---

### Scene Structure

```
World.tscn
├── GridRenderer
├── Player (CharacterBody2D)
│   └── Camera2D
└── [PlaceableNodes spawned at runtime]
    ├── Sprite2D
    └── MiningOverlay
```

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
- [x] Cursor as IDE

---

## What's Next

### Immediate
- [ ] **`World.cs` refactor** — extract player input/mining logic into a `PlayerController` class. `World` shouldn't own input handling.
- [ ] `GetMousePositionInGrid()` → make `public` on `World`, call from `PlayerController`

### Registries (planned as one unit, in dependency order)
- [ ] **Item registry** — items are the leaf dependency; machines and recipes reference them. Stable string IDs, JSON-driven.
- [ ] **Machine registry** — references items (I/O slots, valid inputs). Includes fixed-timestep `Tick` driving.
- [ ] **Recipe registry** — references both items and machines.

> The tick system will be folded into the machine registry phase rather than treated as a separate refactor step.

---

## Architecture Decision Records

### ADR-001: `Placeable` is a plain C# class, not a Godot Node

**Decision:** All simulation objects extend a plain C# base class rather than `Node` or `Node2D`.

**Rationale:**
- Nodes carry Godot lifecycle overhead (scene tree, signals, `_Process`)
- Simulation doesn't need any of that — it just needs state and a `Tick()`
- Keeping simulation free of scene tree dependencies makes save/load, determinism, and future performance work much easier
- Target of 500+ simultaneous machines makes this a meaningful concern

**Consequence:** There are now two parallel representations of a placed object — `Placeable` (data) and `PlaceableNode` (visual). This is intentional. The presentation layer knows about the data layer; the data layer knows nothing about the presentation layer.

---

### ADR-002: Definitions are data-driven (JSON), behaviour is code

**Decision:** `PlaceableDefinition` is loaded from JSON files. C# subclasses own behaviour.

**Rationale:**
- JSON can describe *what* something is (name, sprite, stats). It cannot describe *what it does*.
- Separating the two means content iteration (tweaking hardness, adding new ore types) doesn't require recompiling.
- The `type` field in JSON maps to a C# subclass, bridging data → behaviour.

**Consequence:** Adding a new placeable type requires both a JSON file and a C# subclass. For purely data-differentiated objects (e.g. two ore types with different hardness but identical behaviour), only a JSON file is needed.

---

### ADR-003: String IDs over integer indices for registry keys

**Decision:** All registry lookups use stable string keys (e.g. `"iron_ore"`), not array indices.

**Rationale:**
- Integer indices break silently when items are reordered, added, or removed
- String IDs survive reordering and are self-documenting in save files and JSON

---

### ADR-004: No multiplayer architecture — dropped

**Decision:** Multiplayer compatibility is not a design goal.

**Rationale:**
- This is a first game; multiplayer is a large scope expansion with significant complexity cost
- The good architectural decisions already made (simulation/presentation split, deterministic state, string IDs) serve single-player goals on their own merits (save/load, simulation integrity)
- The command/action routing pattern that would enable networking adds complexity without single-player payoff — dropped

**What this means practically:** Fixed-timestep tick driving is still worth doing for simulation consistency, but the command/action layer pattern is not being built.

---

### ADR-005: Use `"/"` not `Path.Combine` for Godot resource paths

**Decision:** Path concatenation for `res://` paths uses `directoryPath + "/" + file`, not `System.IO.Path.Combine`.

**Rationale:**
- `Path.Combine` produces OS-specific separators (`\` on Windows)
- Godot's `res://` virtual filesystem requires forward slashes
- Using `Path.Combine` works in the editor on Windows but silently breaks on export

---

### ADR-006: `World.cs` currently owns too much — refactor pending

**Status:** Known issue, not yet resolved.

**Problem:** `World.cs` handles player input, mining state, and proximity checks. These are player concerns, not world concerns.

**Planned resolution:** Extract into a `PlayerController` class. `World` should only coordinate the grid state and spawning/despawning of `PlaceableNode`s.