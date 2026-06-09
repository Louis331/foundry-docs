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

This pays off for save/load, simulation integrity, multiplayer determinism, and future performance with 500+ machines.

---

### Command Pipeline

All state mutations flow through commands. No code outside of `Execute()` / `Rollback()` is permitted to mutate `GridWorld` directly.

```
Player input
    ↓
PlayerController
    ↓
FactoryOrchestrator  (creates commands)
    ↓
FactoryManager       (queues, ticks, executes, tracks history)
    ↓
Command.Execute()    (mutates GridWorld)
    ↓
Signal emitted       (World updates visuals)
```

This is the foundation for lockstep multiplayer — commands are serialisable units of work that can be sent to peers and executed identically on both machines.

---

### Core Classes

#### `CommandBase` (abstract partial C# class, extends GodotObject)
Base class for all commands. Every state mutation in the game is a command.

| Property | Type | Description |
|---|---|---|
| `Tick` | `int` | Simulation tick the command was created on |
| `PlayerId` | `int` | ID of the player who issued the command (placeholder, hardcoded to 0) |
| `IsUndoable` | `bool` | Whether the command participates in undo/redo history. Default `true` |

- Declares `abstract Execute()` and `abstract Rollback()`
- Extends `GodotObject` so instances can be passed through Godot signals
- Must be `partial` due to Godot source generator requirements

**Subclasses:**
- `PlaceCommand` — places a `Placeable` at a cell
- `RemoveCommand` — removes a `Placeable` from a cell, stashes it for undo
- `ExtractCommand` — extracts from a `ResourceNode`, non-undoable

---

#### `PlaceCommand` (partial C# class, extends CommandBase)
Places a placeable in the world.

| Property | Type | Description |
|---|---|---|
| `CellPosition` | `Vector2I` | Target cell |
| `Placeable` | `Placeable` | The placeable instance to place |

- `Execute()` calls `GridWorld.Instance.PlaceInWorld(CellPosition, Placeable)`
- `Rollback()` calls `GridWorld.Instance.RemoveFromWorld(CellPosition)`

---

#### `RemoveCommand` (partial C# class, extends CommandBase)
Removes a placeable from the world.

| Property | Type | Description |
|---|---|---|
| `CellPosition` | `Vector2I` | Target cell |
| `Placeable` | `Placeable` | Stashed on construction via `GridWorld.GetPlaceableAt` |

- Constructor throws `InvalidOperationException` if cell is empty at construction time
- `Execute()` calls `GridWorld.Instance.RemoveFromWorld(CellPosition)`
- `Rollback()` calls `GridWorld.Instance.PlaceInWorld(CellPosition, Placeable)` restoring the stashed instance

---

#### `ExtractCommand` (partial C# class, extends CommandBase)
Extracts resources from a `ResourceNode`.

| Property | Type | Description |
|---|---|---|
| `ResourceNode` | `ResourceNode` | The node to extract from |
| `amount` | `int` (private) | Amount to extract, must be >= 1 |

- `IsUndoable` is always `false` — extraction is a permanent game action
- `Rollback()` is a no-op
- Constructor throws `InvalidOperationException` if amount < 1

---

#### `FactoryManager` (Godot Node, Autoload)
Owns the command pipeline. The central nervous system of the simulation.

| Field | Type | Description |
|---|---|---|
| `commandQueue` | `PriorityQueue<CommandBase, int>` | Pending commands sorted by tick, drained each physics frame |
| `commandHistory` | `List<CommandBase>` | Full history of executed undoable commands |
| `cursor` | `int` | Points to the last executed command in history. -1 = empty |
| `CurrentTick` | `int` | Incremented every `_PhysicsProcess` call |

**Signals:**
- `CommandExecuted(CommandBase command)` — emitted after a command executes
- `CommandRolledBack(CommandBase command)` — emitted after a command rolls back

**Methods:**
- `AddCommand(CommandBase)` — enqueues a command for execution
- `Undo()` — rolls back the command at cursor, decrements cursor
- `Redo()` — re-executes the command after cursor, increments cursor

**Tick behaviour:**
- Runs at Godot's physics tick rate (default 60Hz = 60 TPS)
- Each tick: increments `CurrentTick`, drains all queued commands whose tick <= `CurrentTick`, then ticks all `GridWorld.Tickables`
- Commands are dequeued in tick order (lowest first) via `PriorityQueue`

**Undo/Redo model:**
- History is a `List` with a cursor — not a stack. Cursor moves back on undo, forward on redo.
- New commands truncate history after the cursor, discarding redo history (standard linear history)
- Non-undoable commands (`IsUndoable = false`) execute but are never added to history

---

#### `FactoryOrchestrator` (plain C# singleton)
Sits between `PlayerController` and `FactoryManager`. Translates player intent into commands. The only class permitted to construct command instances.

| Method | Description |
|---|---|
| `AddPlaceable(Vector2I, Placeable)` | Creates and queues a `PlaceCommand` |
| `RemovePlaceable(Vector2I)` | Creates and queues a `RemoveCommand` |
| `ExtractResource(ResourceNode, int)` | Creates and queues an `ExtractCommand` |

- Instantiated by `FactoryManager._Ready()` via `new FactoryOrchestrator()`
- Not an Autoload — lives as a plain C# singleton, no Godot lifecycle needed

---

#### `Placeable` (plain C# class)
Base class for all objects that exist in the simulation.

| Property | Type | Description |
|---|---|---|
| `GridPosition` | `Vector2I` | Cell the object occupies |
| `Id` | `string` | Stable string key, matches registry |

- Has a virtual `Tick()` method - overridden by subclasses for machine behaviour
- Not a Node - no scene tree dependency whatsoever

**Subclasses:**
- `ResourceNode` - mineable resource (iron ore, etc.)
- `Machine` - tickable machine (furnace, etc.) - has `Tier` and `PowerConsumption`

---

#### `PlaceableDefinition` (plain C# class)
Static data for a type of placeable, loaded from JSON. One definition per *type*, many `Placeable` instances.

| Property | Type | Description |
|---|---|---|
| `Id` | `string` | Stable string key |
| `Name` | `string` | Display name |
| `SpritePath` | `string` | Path to sprite asset |
| `Size` | `(int X, int Y)` | Grid footprint |
| `Type` | `PlaceableType` | `Resource` or `Machine` |
| `Hardness` | `float` | Mine time multiplier (default `1.0`) |
| `Tier` | `MachineTier?` | Optional. Steam, LV, MV, HV |
| `PowerConsumption` | `int?` | Optional. Positive = consumes, negative = produces |
| `Speed` | `float` | Processing speed multiplier (default `1.0`). Effective recipe time = `Ticks / Speed` |

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

#### `PlaceableType` (enum)
Categorises placeables for simulation routing.

| Value | Description |
|---|---|
| `Resource` | Mineable world resource, not tickable |
| `Machine` | Tickable, added to `FactoryManager` tick loop |

---

#### `MachineTier` (enum)
Defines the power tier of a machine.

| Value | Description |
|---|---|
| `Steam` | Powered by fuel combustion |
| `LV` | Low voltage electric |
| `MV` | Medium voltage electric |
| `HV` | High voltage electric |

---

#### `Machine` (plain C# class)
Subclass of `Placeable`. Base class for all tickable machines.

| Property | Type | Description |
|---|---|---|
| `GridPosition` | `Vector2I` | Inherited from `Placeable` |
| `Id` | `string` | Inherited from `Placeable` |
| `Type` | `PlaceableType` | Inherited from `Placeable` |
| `Recipes` | `List<RecipeDefinition>` | Candidate recipes fetched from `RecipeRegistry` on construction |

- Overrides `Tick()` — will process active recipes when recipe registry exists
- Added to `GridWorld.Tickables` on placement, ticked by `FactoryManager` each physics frame
- Fetches candidate recipes via `RecipeRegistry.Instance.GetRecipes(id)` on construction and caches the result

---

#### `RecipeDefinition` (plain C# class)
Static data for a crafting recipe, loaded from JSON. Describes one transformation a machine can perform.

| Property | Type | Description |
|---|---|---|
| `Id` | `string` | Stable string key |
| `Inputs` | `Dictionary<string, int>` | Item id to quantity required |
| `Outputs` | `Dictionary<string, int>` | Item id to quantity produced |
| `MachineIds` | `List<string>` | Which machine types can run this recipe |
| `Ticks` | `int` | Base duration of the transformation |

Effective time = `Ticks / machine.Speed`. The recipe defines the work; the machine defines the rate.

JSON lives at: `res://Objects/Recipes/Data/*.json`  
Example: [smelting.json](#smeltingjson)

---

#### `RecipeRegistry` (Autoload Node)
Loads all `RecipeDefinition` JSON files at startup. Builds two indexes for fast lookup.

- `Recipes` - `Dictionary<string, List<RecipeDefinition>>` reverse index keyed by machine id. Built at load time so machine instances never scan the full registry per tick.
- `RecipeById` - `Dictionary<string, RecipeDefinition>` keyed by recipe id. Used for validation.
- Exposes `GetRecipes(machineId)` - returns candidate recipe list for a machine, empty list if none found.
- `Machine` instances call `GetRecipes` once on construction and cache the result.

---

#### `GridWorld` (Godot Node2D, singleton)
Authoritative simulation state. Owns the occupancy map.

| Method | Description |
|---|---|
| `PlaceInWorld(Vector2I, Placeable)` | Atomic check-and-commit placement |
| `RemoveFromWorld(Vector2I)` | Atomic removal |
| `GetPlaceableAt(Vector2I)` | Returns the `Placeable` at a cell, or null |
| `WorldToGrid(Vector2)` | Converts world position to grid cell |
| `GridToWorld(Vector2I)` | Returns cell centre in world coords |
| `GetCellRect(Vector2I)` | Returns full cell Rect2 |

- Occupied cells: `Dictionary<Vector2I, Placeable>` — the ground truth of what's placed where
- Tickables: `Dictionary<Vector2I, Placeable>` — public, all tickable placeables currently in the world, iterated by `FactoryManager` each tick
- Accessible globally via `GridWorld.Instance`

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

#### `World` (Godot Node2D — scene coordinator)
Presentation coordinator. Listens to `FactoryManager` signals and updates visuals. Owns no simulation state.

- Connects to `FactoryManager.CommandExecuted` and `FactoryManager.CommandRolledBack` in `_Ready()`
- On `CommandExecuted`: pattern matches command type — `PlaceCommand` → `PlaceNodeAt`, `RemoveCommand` → `RemoveNodeAt`
- On `CommandRolledBack`: inverse — `PlaceCommand` rollback → `RemoveNodeAt`, `RemoveCommand` rollback → `PlaceNodeAt`
- Tracks `Dictionary<Vector2I, PlaceableNode>` (`placedNodes`) for fast node lookup
- Exposes `GetPlaceableNodeAt(Vector2I)` for mining overlay access

---

#### `PlayerController` (Godot Node2D)
Handles player input and delegates to `FactoryOrchestrator`. Owns mining state.

- Left click → `FactoryOrchestrator.Instance.AddPlaceable(...)`
- Middle click → `FactoryOrchestrator.Instance.RemovePlaceable(...)` (dev tool)
- Right click held → mining loop, calls `FactoryOrchestrator.Instance.ExtractResource(...)` on completion
- Ctrl+Z → `FactoryManager.Instance.Undo()`
- Ctrl+Y → `FactoryManager.Instance.Redo()`
- Tracks mining state via `(float Timer, Placeable Placeable)? activeMining` nullable tuple
- Does **not** mutate `GridWorld` directly — all mutations go through `FactoryOrchestrator`

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

## Game Startup

All cross-registry validation runs at startup via `GameStartup` before the game is playable. This ensures broken references are caught immediately with a full error list rather than failing silently at runtime.

#### `GameStartup` (Autoload Node)
Orchestrates all validators in `_Ready`. Listed last in Autoloads so all registries are guaranteed loaded.

- Collects errors from all validators into one combined list
- Prints every error via `GD.PrintErr` before stopping
- Calls `GetTree().Quit()` if any errors found - game refuses to start with broken data

#### `RecipeValidator` (static class)
Validates all loaded recipes against `ItemRegistry` and `PlaceableRegistry`.

- Checks every `Inputs` key exists in `ItemRegistry`
- Checks every `Outputs` key exists in `ItemRegistry`
- Checks every `MachineIds` entry exists in `PlaceableRegistry`
- Reports every invalid reference individually - not just the first failure

#### `ResourceValidator` (static class)
Validates all placeable drops against `ItemRegistry`.

- Iterates all `PlaceableDefinition` entries that have a `Drop`
- Checks `Drop.ItemId` exists in `ItemRegistry`

---

### Scene Structure

```
World.tscn
├── GridWorld
├── GridRenderer
├── Player (CharacterBody2D)
│   ├── Camera2D
│   └── PlayerController
└── [PlaceableNodes spawned at runtime]
    ├── Sprite2D
    └── MiningOverlay
```

**Autoloads (in load order):**
```
PlaceableRegistry
ItemRegistry
RecipeRegistry
GameStartup
FactoryManager        ← instantiates FactoryOrchestrator in _Ready
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

### ADR-004: Lockstep co-op multiplayer is a design goal

**Decision:** The game is being built with lockstep multiplayer as a target architecture, even before networking is implemented.

**Rationale:**
- Lockstep requires all state mutations to be deterministic and reproducible from a sequence of commands
- Building the command pipeline now costs little in single-player and pays off significantly when networking is added
- The simulation/presentation split already established makes this natural

**What this means practically:**
- All state mutations flow through `CommandBase` subclasses via `FactoryManager`
- Commands carry a tick number so peers can execute them at the same simulation moment
- `FactoryOrchestrator` is the only class that constructs commands — future network layer sends serialised commands directly to `FactoryManager`
- Known future concerns: undo/redo in multiplayer (what happens when one peer undoes something another depends on?), late-join world state sync via snapshot rather than command replay, server-side validation of range checks

---

### ADR-005: Use `"/"` not `Path.Combine` for Godot resource paths

**Decision:** Path concatenation for `res://` paths uses `directoryPath + "/" + file`, not `System.IO.Path.Combine`.

**Rationale:**
- `Path.Combine` produces OS-specific separators (`\` on Windows)
- Godot's `res://` virtual filesystem requires forward slashes
- Using `Path.Combine` works in the editor on Windows but silently breaks on export

---

### ADR-006: `World.cs` is a signal-driven presenter, not a coordinator

**Status:** Resolved.

**Problem:** Earlier versions of `World.cs` handled player input, mining state, proximity checks, and direct `GridWorld` mutation. These were player concerns, not world concerns.

**Resolution:** 
- Input and mining state extracted to `PlayerController`
- Direct `GridWorld` mutation replaced by command pipeline
- `World` now only connects to `FactoryManager` signals and updates `PlaceableNode` visuals in response

---

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

### ADR-010: No separate MachineRegistry — machines extend PlaceableDefinition

**Decision:** There is no `MachineRegistry`. Machine-specific fields (`Tier`, `PowerConsumption`) are optional nullable fields on `PlaceableDefinition`.

**Rationale:**
- A separate registry would duplicate the loading infrastructure for little gain
- Machines are already identified by `PlaceableType.Machine`
- Optional nullable fields keep the definition clean — non-machine placeables simply omit them

**Consequence:** `PlaceableRegistry` remains the single source of truth for all placeable definitions including machines.

---

### ADR-011: Negative PowerConsumption for power-producing machines

**Decision:** `PowerConsumption` uses sign to indicate direction — positive consumes power, negative produces it.

**Rationale:**
- A single `int?` field handles both consumers and producers
- No need for a separate `PowerProduction` field
- Simple convention, consistent with how power networks work in similar games

**Consequence:** All code reading `PowerConsumption` must treat negative values as production. A comment on the field documents the convention.

---

### ADR-012: Full error list before crash on startup validation failure

**Decision:** `GameStartup` collects all validation errors across all validators before stopping, rather than failing on the first error.

**Rationale:**
- Failing on the first error forces a fix-run-fix-run loop - a modder or developer with ten broken references has to fix them one at a time
- Collecting all errors and printing them before quitting surfaces everything at once
- Inspired by the modded Minecraft experience of chasing errors one at a time

**Consequence:** All validators must return `List<string>` rather than throwing immediately. `GameStartup` owns the decision to stop.

---

### ADR-013: Recipe duration is floored to nearest tick

**Decision:** Effective recipe duration is calculated as `Math.Floor(recipe.Ticks / machine.Speed)` and stored as an `int`.

**Rationale:**
- `recipe.Ticks` is an integer, `machine.Speed` is a float, producing a fractional result
- Machines track progress as an integer tick counter, so the duration must resolve to a whole number
- `Math.Floor` avoids float comparison issues at completion and keeps the tick counter simple
- Sub-tick precision has no gameplay value at 20 TPS

**Consequence:** A furnace at 1.5x speed running a 40 tick recipe completes in 26 ticks, not 26.67. Speeds that produce non-integer durations lose a small amount of time — this is acceptable.

---

### ADR-014: Command pipeline for all state mutations

**Decision:** All `GridWorld` mutations must go through a `CommandBase` subclass, queued via `FactoryManager`. No code outside of `Execute()` / `Rollback()` may call `PlaceInWorld` or `RemoveFromWorld` directly.

**Rationale:**
- Lockstep multiplayer requires every state change to be a serialisable, replayable unit of work
- Commands carry tick number and player ID, making them attributable and orderable across peers
- The `PriorityQueue` ensures out-of-order network commands are executed in tick order on all peers

**Consequence:** Adding a new type of state mutation requires a new `CommandBase` subclass. Direct `GridWorld` calls outside of commands are a bug.

---

### ADR-015: Cursor-based undo/redo history, not a stack

**Decision:** Command history is a `List<CommandBase>` with an integer cursor, not a `Stack`.

**Rationale:**
- A stack loses history on pop — once you undo, you cannot redo
- A cursor allows traversal in both directions without destroying entries
- New commands truncate history after the cursor (standard linear history — no branching)

**Consequence:** Undo/redo in multiplayer is a known future complexity — what happens when one peer undoes a command another peer's commands depend on is deferred to the multiplayer card.

---

### ADR-016: Simulation tick driven by physics frame, not a Timer

**Decision:** `FactoryManager` uses `_PhysicsProcess` to drive the simulation tick, replacing the previous `Timer`-based `SimulationManager`.

**Rationale:**
- `_PhysicsProcess` runs at a fixed rate (configurable in Project Settings, default 60Hz) with no float drift
- Timer-based ticking drifts slightly and is not deterministic enough for lockstep
- Tick rate is now a single Project Settings value rather than a hardcoded constant

**Consequence:** Simulation TPS is tied to Godot's physics tick rate. Changing TPS means changing Project Settings → Physics → Common → Physics Ticks Per Second.

---

## Example data
All data examples will live here to support building of new data files to add to the project.

### Placeable example

Placeables are items that can be placed in the world, they all have attributes that are loaded at runtime using the registry.

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

### furnace.json

```json
{
  "Id": "furnace",
  "Name": "Furnace",
  "Type": "Machine",
  "SpritePath": "res://Objects/Placeables/Assets/furnace.png",
  "Size": {
    "X": 1,
    "Y": 1
  }
}
```

### Item examples

Items are goods that exist in inventory and flow through the factory network.

#### iron_ingot.json

```json
{
  "Id": "iron_ingot",
  "Name": "Iron ingot",
  "Description": "A block of iron ingot.",
  "SpritePath": "res://Objects/Items/Assets/iron_ingot.png",
  "MaxStackSize": 100,
  "ItemType": "Solid"
}
```

### Recipe examples

Recipes describe transformations machines perform. Time is in simulation ticks. Effective time = `Ticks / machine.Speed`.

#### smelting.json

```json
{
  "Id": "smelting",
  "Inputs": {
    "iron_ore": 3,
    "coal": 1
  },
  "Outputs": {
    "iron_ingot": 1
  },
  "MachineIds": ["furnace"],
  "Ticks": 40
}
```