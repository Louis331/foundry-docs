---
---
## Architecture

[Home](index) | [ADRs](adr) |

---

## Table of Contents

1. [Architecture](#architecture)
   - [Simulation / Presentation Split](#simulation--presentation-split)
   - [Core Classes](#core-classes)
   - [JSON Loading](#json-loading)
   - [Scene Structure](#scene-structure)
2. [Example data](#example-data)

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
Extracts resources from a `ResourceNode` and deposits the drop into a `PlayerInventory`.

| Property | Type | Description |
|---|---|---|
| `ResourceNode` | `ResourceNode` | The node to extract from |
| `Inventory` | `PlayerInventory` | Destination inventory to deposit the drop into |

- `IsUndoable` is always `false` — extraction is a permanent game action
- `Rollback()` is a no-op
- `Execute()` looks up the drop definition from `PlaceableRegistry`, resolves the item from `ItemRegistry`, calls `ResourceNode.Extract()` and deposits the result into `Inventory`

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
| `ExtractResource(ResourceNode, PlayerInventory)` | Creates and queues an `ExtractCommand` |

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
- `Machine` - tickable machine (furnace, etc.) - has `Tier` and `PowerConsumption`

#### `ResourceNode` (plain C# class, extends Placeable)
Subclass of `Placeable`. Represents a mineable world resource.

| Property | Type | Description |
|---|---|---|
| `RemainingStock` | `int` | Current mineable stock, initialised from `PlaceableDefinition.MaxStock` |
| `IsDepleted` | `bool` | True when `RemainingStock <= 0` |

- `Extract(int amount)` — returns `Math.Min(amount, RemainingStock)` and decrements stock accordingly
- Not tickable — added to world occupancy map but not to `GridWorld.Tickables`

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
| `MaxStock` | `int?` | Total mineable stock for resource nodes. Null for machines. |

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
| `ItemType` | `ItemType` | `Solid`, `Fluid`, or `Placeable` |

JSON lives at: `res://Objects/Items/Data/*.json`  
Example: [iron_ore.json](#iron_ingotjson)

---

#### `ItemRegistry` (Autoload Node)
Loads all `ItemDefinition` JSON files at startup. Globally accessible via Godot Autoload.

- Registered under: Project → Project Settings → Globals → Autoloads
- Uses the shared `JsonLoader` helper for deserialisation
- `ItemType` deserialises from string values using `[JsonConverter(typeof(JsonStringEnumConverter))]`
- Lookup: `ItemRegistry.GetItem("iron_ore")` → `ItemDefinition`
- Item JSON files use array-per-file format (e.g. `ingots.json` contains multiple `ItemDefinition` objects). Single-item files use a single-element array.
- Duplicate id guard: if two files define the same `Id`, all duplicates are collected and reported before the game quits (consistent with ADR-012).

---

#### `ItemStack` (plain C# class)
Represents a quantity of a single item type. Used as the unit of storage in `PlayerInventory`.

| Property | Type | Description |
|---|---|---|
| `ItemId` | `string` | Stable string key matching `ItemRegistry` |
| `Quantity` | `int` | Number of items in this stack |

---
#### `PlayerInventory` (plain C# class)
Owned by `Player`. Slot-based item storage with stack size enforcement and hotbar selection state.

| Property | Type | Description |
|---|---|---|
| `ItemSlots` | `List<ItemStack?>` | Current inventory contents, null slots represent empty positions |
| `MaxSlots` | `int` | Maximum number of occupied slots (default `10`) |
| `SelectedHotbarSlot` | `int` | Currently selected hotbar index. -1 means nothing selected |
| `InventoryChanged` | `event Action` | Fired after any inventory mutation |

- `AddItem(ItemStack)` — fills an existing partial stack of the same item first; spills remainder into a new slot recursively; logs a message if inventory is full
- `HotbarSelectSlot(int index)` — selects the slot at index; if index is already selected, deselects (sets to -1); returns `ItemStack?` at the slot
- `GetHotbarSelectedItem()` — returns `ItemStack?` at `SelectedHotbarSlot`, null if -1 or empty
- `removeItem(ItemStack stack, int quantity)` — finds slot by reference equality, decrements quantity, nulls slot if zero, emits `InventoryChanged`
- Stack size enforced against `ItemDefinition.MaxStackSize` via `ItemRegistry` lookup
- Known limitation: items are silently lost if inventory is full — physical drop behaviour is post-MVP
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
- Recipe JSON files use array-per-file format grouped by family (e.g. `smelting.json`, `washing.json`).
- Duplicate id guard: same collect-all behaviour as `ItemRegistry`.

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
- Tickables: `SortedDictionary<Vector2I, Placeable>` — public, all tickable placeables currently in the world, iterated by `FactoryManager` each tick. Keyed by world-space origin cell, ordered by X then Y for deterministic iteration order across clients.
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
- Sets `Hud.SetInventory(Player.Inventory)` in `_Ready` after node references are resolved

---

#### `Hud` (Godot Control, child of CanvasLayer)
Presentation layer for the player hotbar. Subscribes to `PlayerInventory.InventoryChanged` and redraws on mutation.

| Method | Description |
|---|---|
| `SetInventory(PlayerInventory)` | Wires inventory reference and subscribes to `InventoryChanged` |
| `drawHotbar()` | Iterates 10 slots, updates each `PanelContainer` label from current inventory state |

- Anchored to bottom-centre of viewport via parent `CanvasLayer`
- Redraws all 10 slots on any inventory change — no diffing, acceptable at hotbar scale

---

#### `PlayerController` (Godot Node2D)
Handles player input and delegates to `FactoryOrchestrator`. Owns mining state.

- Left click → `FactoryOrchestrator.Instance.AddPlaceable(...)`
- Middle click → `FactoryOrchestrator.Instance.RemovePlaceable(...)` (dev tool)
- Right click held → mining loop, calls `FactoryOrchestrator.Instance.ExtractResource(resourceNode, player.Inventory)` on completion
  - Acquires `player` reference via `GetParent<Player>()` (parent node lookup, since `PlayerController` is a child of `Player` in the scene tree)
  - Accesses `player.Inventory` property to pass the player's `PlayerInventory` instance
- Ctrl+Z → `FactoryManager.Instance.Undo()`
- Ctrl+Y → `FactoryManager.Instance.Redo()`
- Tracks mining state via `(float Timer, Placeable Placeable)? activeMining` nullable tuple
- Does **not** mutate `GridWorld` directly — all mutations go through `FactoryOrchestrator`

---

#### `Player` (Godot CharacterBody2D)
- `MiningRange` - exported field, set in Inspector
- `MiningSpeed` - encapsulated property
- `Inventory` - `PlayerInventory` instance, initialised in `_Ready()`
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
- Generic methods constrained to reference types: `where T : class`
- Uses `private static readonly` for the options object to avoid recreating it on every call

**Methods:**
- `LoadJson<T>(string directoryPath)` — scans directory, deserialises each file as a single `T`. Used by `PlaceableRegistry`.
- `LoadJsonFromFile<T>(string filePath)` — deserialises a single file as `T`.
- `LoadManyJson<T>(string directoryPath)` — scans directory, deserialises each file as `List<T>`, returns a flat merged `List<T>`. Used by `ItemRegistry` and `RecipeRegistry`.
- `LoadManyJsonFromFile<T>(string filePath)` — deserialises a single file as `List<T>`.

File grouping is an authoring convenience — the registry receives a flat list with no knowledge of which file each definition came from.

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
├── HudLayer
│   └── Hud
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

# Example data
All data examples will live here to support building of new data files to add to the project.

## Placeable example

Placeables are items that can be placed in the world, they all have attributes that are loaded at runtime using the registry.

#### iron_ore.json

```json
{
  "Id": "iron_ore",
  "Name": "Iron Ore",
  "Type": "Resource",
  "SpritePath": "res://Objects/Placeables/Assets/iron_ore.png",
  "Size": { "X": 1, "Y": 1 },
  "Drop": {
    "ItemId": "iron_ore",
    "DropAmount": 10
  },
  "MaxStock": 100,
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

## Item examples

Items are goods that exist in inventory and flow through the factory network.

#### ingots.json

```json
[
  {
    "Id": "iron_ingot",
    "Name": "Iron Ingot",
    "Description": "An ingot of iron that can be used to craft other items.",
    "SpritePath": "res://Objects/Items/Assets/iron_ingot.png",
    "MaxStackSize": 100,
    "ItemType": "Solid"
  }
]
```

## Recipe examples

Recipes describe transformations machines perform. Time is in simulation ticks. Effective time = `Ticks / machine.Speed`.

#### smelting.json

```json
[
  {
    "Id": "smelting",
    "Inputs": {
      "iron_ore": 3,
      "coal": 1
    },
    "Outputs": {
      "iron_ingot": 1
    },
    "MachineId": ["furnace"],
    "Ticks": 40
  }
]
```