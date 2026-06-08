---
---
## Architecture

[Home](index)

---

## Table of Contents

1. [Architecture](#architecture)
   - [Simulation / Presentation Split](#simulation--presentation-split)
   - [Core Classes](#core-classes)
   - [JSON Loading](#json-loading)
   - [Scene Structure](#scene-structure)
2. [Architecture Decision Records (ADRs)](#architecture-decision-records)
3. [Example data](#example-data)

---

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

- Has a virtual `Tick()` method - overridden by subclasses for machine behaviour
- Not a Node - no scene tree dependency whatsoever

**Subclasses:**
- `ResourceNode` - mineable resource (iron ore, etc.) - has `Hardness`

---

#### `PlaceableDefinition` (plain C# class)
Static data for a type of placeable, loaded from JSON. One definition per *type*, many `Placeable` instances.

| Property | Type | Description |
|---|---|---|
| `Id` | `string` | Stable string key |
| `Name` | `string` | Display name |
| `Sprite` | `string` | Path to sprite asset |
| `Size` | `(int X, int Y)` | Grid footprint |
| `Type` | `string` | Maps to a C# subclass |
| `Hardness` | `float` | Mine time multiplier (default `1.0`) |

JSON lives at: `res://Objects/Data/Placeables/*.json`  
Example: [iron_ore.json](#iron_orejson)

---

#### `PlaceableRegistry` (Autoload Node)
Loads all `PlaceableDefinition` JSON files at startup. Globally accessible via Godot Autoload.

- Registered under: Project → Project Settings → Globals → Autoloads
- Uses `JsonLoader` + `TupleConverter` helpers for deserialisation
- Lookup: `PlaceableRegistry.Get("iron_ore")` → `PlaceableDefinition`

---

#### `ItemDefinition` (plain C# class)
Static data for a type of item, loaded from JSON. One definition per *type*.

| Property | Type | Description |
|---|---|---|
| `Id` | `string` | Stable string key |
| `Name` | `string` | Display name |
| `Description` | `string` | Flavour/tooltip text |
| `SpritePath` | `string` | Path to sprite asset |
| `MaxStackSize` | `int` | Max items per stack |
| `Type` | `ItemType` | `Solid`, `Fluid`, or `Placeable` |

JSON lives at: `res://Objects/Items/Data/*.json`  
Example: [iron_ore.json](#iron_ingotjson)

---
#### `ItemRegistry` (Autoload Node)
Loads all `ItemDefinition` JSON files at startup. Globally accessible via Godot Autoload.

- Registered under: Project → Project Settings → Globals → Autoloads
- Uses the shared `JsonLoader` helper for deserialisation
- `ItemType` deserialises from string values using `[JsonConverter(typeof(JsonStringEnumConverter))]`
- Lookup: `ItemRegistry.GetItem("iron_ore")` → `ItemDefinition`
---

#### `ItemType` (enum)
Controls how an item is routed and where it can exist.

| Value | Description |
|---|---|
| `Solid` | Travels through item pipes |
| `Fluid` | Travels through fluid pipes — covers both liquids and gases |
| `Placeable` | Travels through item pipes and can also be deployed in the world |

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

- Dictionary: `Dictionary<Vector2I, Placeable>` - the ground truth of what's placed where

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

- Has a `MiningProgress` property - setter calls `QueueRedraw()`
- `_Draw` renders a dark rect growing from the bottom of the cell upward

---

#### `World` (Godot Node - main scene coordinator)
Previously named `GridTest`. The scene coordinator / presenter - not a simulation class.

- Holds `GridWorld` instance
- Handles player input: left click = place, right click = mine, middle click = remove (dev tool)
- Tracks mining state via `(float Timer, Placeable Target)? activeMining` nullable tuple
- Holds `Dictionary<Vector2I, PlaceableNode>` (`placedNodes`) for fast node lookup
- Mining logic: held right-click, proximity check, `hardness / miningSpeed` duration

---

#### `Player` (Godot CharacterBody2D)
- `MiningRange` - exported field, set in Inspector
- `MiningSpeed` - encapsulated property
- Owns a `Camera2D` as a child (camera follows player)

---

#### `GridRenderer` (Godot Node2D)
- Draws the checkerboard grid using `_Draw`
- Viewport-culled, camera-aware via `GetCanvasTransform().AffineInverse()`
- Calls `QueueRedraw()` in `_Process` so it updates as the camera moves

---

## JSON Loading

Placeables are loaded at startup by `PlaceableRegistry` using two helper classes.

### JsonLoader

A static utility class that handles reading and deserialising JSON files from the Godot virtual filesystem.

- Uses Godot's `FileAccess` and `DirAccess` APIs to read files
- Scans a directory for all `.json` files and deserialises each one
- Generic method constrained to reference types: `Load<T> where T : class`
- Uses `private static readonly` for the options object to avoid recreating it on every call

### TupleConverter

A custom `JsonConverter<(int X, int Y)>` from `System.Text.Json.Serialization`.

- Required because `System.Text.Json` cannot deserialise C# ValueTuples out of the box
- Reads a JSON object with `X` and `Y` fields and maps it to a C# tuple
- Used for the `Size` field on `PlaceableDefinition`

### Path Note

Always use string concatenation for `res://` paths rather than `System.IO.Path.Combine`. See [ADR-005](#adr-005-use--not-pathcombine-for-godot-resource-paths).

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

## Architecture Decision Records

### ADR-001: `Placeable` is a plain C# class, not a Godot Node

**Decision:** All simulation objects extend a plain C# base class rather than `Node` or `Node2D`.

**Rationale:**
- Nodes carry Godot lifecycle overhead (scene tree, signals, `_Process`)
- Simulation doesn't need any of that - it just needs state and a `Tick()`
- Keeping simulation free of scene tree dependencies makes save/load, determinism, and future performance work much easier
- Target of 500+ simultaneous machines makes this a meaningful concern

**Consequence:** There are now two parallel representations of a placed object - `Placeable` (data) and `PlaceableNode` (visual). This is intentional. The presentation layer knows about the data layer; the data layer knows nothing about the presentation layer.

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

### ADR-004: No multiplayer architecture - dropped

**Decision:** Multiplayer compatibility is not a design goal.

**Rationale:**
- This is a first game; multiplayer is a large scope expansion with significant complexity cost
- The good architectural decisions already made (simulation/presentation split, deterministic state, string IDs) serve single-player goals on their own merits (save/load, simulation integrity)
- The command/action routing pattern that would enable networking adds complexity without single-player payoff - dropped

**What this means practically:** Fixed-timestep tick driving is still worth doing for simulation consistency, but the command/action layer pattern is not being built.

---

### ADR-005: Use `"/"` not `Path.Combine` for Godot resource paths

**Decision:** Path concatenation for `res://` paths uses `directoryPath + "/" + file`, not `System.IO.Path.Combine`.

**Rationale:**
- `Path.Combine` produces OS-specific separators (`\` on Windows)
- Godot's `res://` virtual filesystem requires forward slashes
- Using `Path.Combine` works in the editor on Windows but silently breaks on export

---

### ADR-006: `World.cs` currently owns too much - refactor completed

**Status:** Known issue, resolved.

**Problem:** `World.cs` handles player input, mining state, and proximity checks. These are player concerns, not world concerns.

**Planned resolution:** Extract into a `PlayerController` class. `World` should only coordinate the grid state and spawning/despawning of `PlaceableNode`s.

### ADR-007: `Fluid` covers liquids and gases

**Decision:** There is a single `Fluid` item type rather than separate `Liquid` and `Gas` types.

**Rationale:**
- There is no meaningful difference in routing behaviour between liquids and gases in this game — both travel through fluid pipes
- Splitting them would add complexity without any current mechanical payoff

**Consequence:** If a future mechanic requires distinguishing liquids from gases (e.g. pressure, temperature), a new enum value can be added at that point.

---

### ADR-008: `Placeable` ItemType for deployable items

**Decision:** Machines, vehicles, and other world-deployable items use `ItemType.Placeable`.

**Rationale:**
- These items need to travel through item pipes for autocrafting and live in player inventory
- They also need to be deployable in the world
- A single enum value handles both routing and deployment eligibility cleanly

**Consequence:** The inventory system and pipe routing system use `Type == Placeable` as the flag for deployment eligibility. Items the player should never be able to place (e.g. `iron_ore`) use `Solid` regardless of their in-world representation as a `ResourceNode`.

---

### ADR-009: Item IDs can match Placeable IDs without conflict

**Decision:** `ItemRegistry` and `PlaceableRegistry` are separate dictionaries — the same string ID (e.g. `"iron_ore"`) can exist in both without conflict.

**Rationale:**
- `iron_ore` as a `PlaceableDefinition` describes the mineable node in the world
- `iron_ore` as an `ItemDefinition` describes the item that drops from it and lives in inventory
- Lookup always targets a specific registry, so there is no ambiguity

---

## Example data
All data examples will live here to support building of new data files to add to the project.

### Placeable example

Placeables are items that can be place in the world, they all have attributes that are loaded at runtime using the registry.

#### iron_ore.json

```json
{
  "Id": "iron_ore",
  "Name": "Iron Ore",
  "Type": "Resource",
  "SpritePath": "res://Objects/Placeables/Assets/iron_ore.png",
  "Size": {
    "X": 1,
    "Y": 1
  },
  "Drop": {
    "ItemId": "iron_ore",
    "TotalAmount": 10
  },
  "Hardness": 1.0
}
```

### Item examples

Items are goods that exist in inventory and flow through the factory network.

#### iron_ingot.json

```json
{
  "Id": "iron_ingot",
  "Name": "Iron ingot",
  "Description": "A block of iron ngot.",
  "SpritePath": "res://Objects/Items/Assets/iron_ingot.png",
  "MaxStackSize": 100,
  "ItemType": "Solid"
}
```