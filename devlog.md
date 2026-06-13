---
---

# Dev Log

[Home](index)

## 13th June 2026

### Summary
Today I finished off the UI and made the game feel a bit more gamey. It will allow for testing new items a lot easier going forward. I can now select and deselect an item in the hotbar using the number keys, saving the scroll bar for zoom in and out later. CoderRabbit is a very good tool, keeps catching things that would have caused an issue further down the line and is allowing faster learning of C#.

+ Added unit testing and decided to use the delegate injection pattern going forward to make more code testable.

Next was to add an optional field that would allow `Placeables` to have a collisons, this was the simplest task all day.

Added a chunk system to the world to allow to loading and unloading later down the line. Also implemented my first strut to replace Vector2I to allow CLI unit testing. Have raised another tech debt story to start replace this where we can.

### Basic hotbar UI — polish, inventory reactivity, layout (MAK-2)
`SelectedHotbarSlot` moved from `HUD` to `PlayerInventory` — selection state is player data not UI state. `HUD.SelectHotbarSlot` now delegates to `PlayerInventory.HotbarSelectSlot` and only handles the visual highlight. Re-selecting the active slot deselects it.

`RemoveItem(ItemStack, int)` was refacted in `PlayerInventory` — finds slot by reference equality, decrements quantity, nulls slot at zero. `InventoryChanged` C# event emitted on mutation and `HUD` subscribes in `SetInventory`, redrawing all 10 slots on any change.

`Hud` wrapped in a `CanvasLayer` in `World.tscn` so it stays fixed to the viewport independent of camera movement. `HotbarContainer` anchored to bottom-centre within a Full Rect root control.

Deferred: `RemoveItem` does not account for placement signal result — optimistic decrement only. Blocking placement on non-placeable items captured as a separate card.

### Cursor C# language server fix
godot-tools extension was conflicting with the C# language server, breaking go-to-definition. Disabled godot-tools for the workspace and installed the Anysphere C# extension — full intellisense and go-to-definition now working.

### Initial unit test suite (MAK-48)
xUnit test project added at `tests/Makwright.Tests.csproj`, referenced in the existing solution. 35 tests across 7 files covering all testable plain C# simulation classes: `PlayerInventory`, `ResourceNode`, `ItemStack`, `ItemDefinition`, `PlaceableDefinition`, `RecipeDefinition`, and `TupleConverter`.

`PlayerInventory` required two small production refactors to unlock testing. `AddItem` previously called `ItemRegistry.Instance` directly — replaced with an injected `Func<string, ItemDefinition?>` constructor parameter, defaulting to the real registry at the `Player.cs` call site. `GD.Print` in the full-inventory path was made injectable via an optional `Action<string>` log delegate to avoid an `AccessViolation` outside the Godot runtime.

Classes with Godot dependencies (commands, registries, `JsonLoader`, all Nodes) are not covered — they require either a running Godot runtime or non-trivial refactoring. The delegate injection pattern established by `PlayerInventory` is the documented approach for new simulation classes going forward. Blocked refactors captured in [docs/testing.md](testing).

CI wiring and `.coderabbit.yaml` updated to suppress test suggestions for Godot-dependent classes.

ADR-020 added documenting the xUnit strategy and the plain-C#-only test boundary.

### Add optional collision to placeables (MAK-6)
`Collidable` bool field added to `PlaceableDefinition` (default `false`). When true, `PlaceableNode.Initialize` dynamically constructs a `StaticBody2D` with a child `CollisionShape2D` sized to the placeable's footprint (`Size * CellSize`), centred on the node origin. Player collision works via shared layer 1 defaults — no explicit layer configuration required. Relevant JSON files (machines, trees) updated to set `"Collidable": true`.

### Chunk data structure (MAK-9)

`CellCoord` plain C# struct added at `WorldLogic/CellCoord.cs` — Godot-free coordinate type implementing `IComparable<CellCoord>` and `IEquatable<CellCoord>`, ordered X then Y. Replaces `Vector2I` as dictionary key wherever Godot independence is required.

`Chunk` plain C# class added at `WorldLogic/Chunk.cs` — dumb container owning a `Dictionary<CellCoord, Placeable>` for placeables and a `SortedDictionary<CellCoord, Placeable>` for tickables within a 16×16 cell region. No knowledge of its own world position. `AddPlaceable` returns false on double-occupation rather than throwing. `RemovePlaceable` returns false if cell is empty.

`GridWorld` refactored from flat dictionaries to a `SortedDictionary<CellCoord, Chunk>`. All public methods (`PlaceInWorld`, `RemoveFromWorld`, `GetPlaceableAt`) unchanged in signature — now route to the correct chunk internally via `CellToChunkCoord` and `CellToCellCoord` private helpers. Chunks are lazy-initialised on first write. `Tickables` flat property removed, replaced with `GetActiveChunks()` returning the full chunk dictionary.

`FactoryManager` tick loop updated to iterate `GetActiveChunks()` then each chunk's `Tickables` — deterministic order maintained. Seam is in place for future active/inactive chunk filtering without touching `FactoryManager`.

Unit tests blocked by `Placeable` depending on `Vector2I` — captured as MAK-55. `Chunk.cs` has a TODO comment referencing the ticket.

---

## 12th June 2026

### Summary
Very sort day today. Implemented the first half of the basic hud. This will help with debugging and making it so the game feels like a game. Played a bit of Foundry which made me have some new ideas about what a factory game does/shoudl have. Scale is one of the biggest things, how can you make it so the recipes are balance and have a need for big factories? How can you make it so the player is always thinking about what they can be doing? You want to encorage players to keep expanding, thinking about the diffrences between modded Minecraft and Factorio for example. Modded Minecraft a lot of players are happy to wait and don't build a big number of machines unlike in Factorio where the famous saying `The factory must grow` comes from. I want Makwright to come in the 2nd camp where the place is always building machines.

### Basic hotbar UI — initial implementation (MAK-2)
Built `HUD.tscn` as a `Control` scene with an `HBoxContainer` of 10 `PanelContainer` slots. `HUD.cs` renders inventory slot labels from `PlayerInventory`. Number keys 1-0 call `SelectHotbarSlot` on `HUD`, updating `SelectedIndex` with yellow modulate highlight on the active slot. `ItemDefinition.IsPlaceable` added and cross-referenced from `PlaceableRegistry` at load time.

---

## 11th June 2026

### Summary
Shorter day today. Fixed a bug where placeables could be rendered twice on the same cell - the command signal was firing regardless of whether placement succeeded. Turned out to be a C# property shadowing bug hiding the real fix.

### Prevent stacking placeables on occupied cells (MAK-51)
`PlaceInWorld` already had an occupancy check but `PlaceCommand.Execute` was ignoring the return value, and `FactoryManager` was emitting `CommandExecuted` unconditionally. Fixed by adding a `Success` flag to `CommandBase` (defaulting `true`), setting it from the return value of `PlaceInWorld`, and gating `EmitSignal` on it in `_PhysicsProcess`. Same guard

### Deterministic Tickable iteration order
Swapped `GridWorld.Tickables` from `Dictionary<Vector2I, Placeable>` to `SortedDictionary` with an explicit `(X, then Y)` comparer — ensures identical tick order across clients for lockstep. Also fixed `RemoveFromWorld` to guard the `Tickables.Remove` call to machines only, and corrected a subtle ordering bug where the type check was happening after `occupiedCells.Remove` had already dropped the reference.

---

## 10th June 2026

### Summary
I started the day by starting to work on MAK-1 which was a card to implement a player inventory. I also experimented with Coderabbit to review my PRs to feel more like a professional work stream. It caught a few issues with lack of defensive code, I was then able to fix these now hopefully leading to less issues in the future.

### Player inventory
Designed and implemented `PlayerInventory` and `ItemStack` this gives configurable stacking sizes uses the item JSON. This can then be used by `PlayerInventory` to stack items together and keep things organised. Updated `ExtractCommand` to use the `PlayerInventory` which is passed through to it from `FactoryOrchestrator`. This meant I could decouple the amount logic from the `PlayerController` and use the drop definition to decide what and how much drops. For now this is just simply outputting to the logs. Did raise a bug card to track a current bug for when the players inventory is full, this keeps removing amounts from the `ResourceNode`. 

### JSON Loader Refactor (MAK-4)
Added `LoadManyJson<T>` and `LoadManyJsonFromFile<T>` to `JsonLoader` to support array-per-file JSON. Items and recipes now group by family (e.g. `ingots.json`, `smelting.json`) rather than one object per file. Both `ItemRegistry` and `RecipeRegistry` migrated across, with duplicate-id guards added to both — all duplicates collected and reported before quit, consistent with the existing startup validation pattern.

---

## 9ht June 2026

### Summary
Implemented a full command pipeline as the foundation for future lockstep co-op multiplayer. All world state mutations now flow through serialisable command objects rather than directly mutating GridWorld. Added cursor-based undo/redo history and wired up Ctrl+Z and Ctrl+Y.
Spent the remainder of the day migrating to Jira and refining the backlog. The board is now fully sorted in priority order for MVP with a total of 104 points — roughly 5 to 6 weeks of focused work, realistically 2 to 3 months accounting for life. The foundation is solid enough that new content after MVP should be mostly data authoring rather than engineering.
Also had a long design session exploring a potential USP feature: an Earth-based world generation mode using real GIS data, where your starting region shapes available resources and tech path based on actual geology and climate. Devon has tin and copper but no oil. Yorkshire has coal. The North Sea has oil. Still post-MVP but it is the kind of feature that makes the game worth talking about.

### Command pipeline
Designed and built `CommandBase` as an abstract class extending `GodotObject`, with `Execute()` and `Rollback()` abstract methods plus shared fields for tick number and player ID. Built `PlaceCommand`, `RemoveCommand`, and `ExtractCommand` as subclasses. `RemoveCommand` stashes the `Placeable` instance at construction time so it can be restored on rollback. `ExtractCommand` is non-undoable by design — resource extraction is a permanent game action.

Key decision: dropped the `ICommand` interface in favour of a pure abstract base class. No interface was needed since every command in the codebase inherits from `CommandBase` directly — the interface was abstraction for its own sake.

### FactoryManager
Built `FactoryManager` as a Godot Autoload owning the command queue, command history, and tick loop. Commands are stored in a `PriorityQueue<CommandBase, int>` keyed by tick number, ensuring out-of-order commands (future network use) execute in the correct order. Replaced the old `Timer`-based `SimulationManager` with `_PhysicsProcess` for deterministic fixed-rate ticking. `SimulationManager` deleted.

Emits `CommandExecuted` and `CommandRolledBack` signals so the presentation layer can react without being in the execution path.

### Undo/redo
Implemented cursor-based undo/redo history — a `List<CommandBase>` with an integer cursor rather than a stack. New commands truncate history after the cursor (standard linear history). Ctrl+Z calls `Undo()`, Ctrl+Y calls `Redo()`. Built as a correctness smoke test for the whole pipeline.

Key decision: cursor over stack because popping a stack destroys redo history. The list is the authoritative record of everything that happened.

### FactoryOrchestrator
Built `FactoryOrchestrator` as a plain C# singleton instantiated by `FactoryManager._Ready()`. Sits between `PlayerController` and `FactoryManager` — the only class permitted to construct command instances. Exposes `AddPlaceable`, `RemovePlaceable`, and `ExtractResource`.

### World refactored
`World` now connects to `FactoryManager` signals in `_Ready()` and updates visuals reactively. No longer called directly by `PlayerController`. This is the hook point for multiplayer — remote commands arrive at `FactoryManager` via the same path as local ones, and `World` renders them identically regardless of origin.

### PlayerController cleaned up
Stripped all direct `GridWorld` mutations from `PlayerController`. All placement and removal now delegates to `FactoryOrchestrator`. Mining extraction similarly routed through `ExtractCommand`.

### Known future concerns noted
- Undo/redo in multiplayer (what happens when one peer undoes something another depends on?)
- Late-join world state sync via snapshot rather than command replay
- Server-side validation of range checks
- `PlayerController` still owns too much — mining logic refactor is a separate card

---

## 8th June 2026

### Summary
A long day - roughly 12 hours of solid development. Built the machine and recipe simulation foundation, added startup validation, and updated the documentation/project workflow. The architecture is in a genuinely solid place.

### Mining progress bar fix
Started the day by fixing the mining overlay progress bar which wasn't rendering correctly.

### Documentation site
Set up a public `makwright-docs` repo with GitHub Pages using the Jekyll Hacker theme with dark mode. Split documentation into `index.md`, `architecture.md`, and `plan.md`. Configured the site to suppress download buttons and the "View on GitHub" header. Live at `https://louis331.github.io/makwright-docs/`. Also set up a GitHub Projects kanban board with Backlog, In Progress, and Done columns with 22 cards ordered by dependency. Established a PR-based workflow for traceability.

### Architecture documentation
Wrote the full `architecture.md` covering the simulation/presentation split, all core classes, JSON loading helpers, scene structure, and ADRs 001-011.

### Machine support
Decided against a separate `MachineRegistry` — machine-specific fields (`Tier`, `PowerConsumption`, `Speed`) sit as nullable fields on `PlaceableDefinition`. Built the `Machine` class extending `Placeable`, added `SimulationManager` to drive the tick loop at 20 TPS, and wired `tickablePlaceables` into `GridWorld`. Smoke tested with a furnace printing on every tick.

### Recipe registry
Designed and built `RecipeDefinition` with `Inputs`/`Outputs` as `Dictionary<string, int>`, `MachineIds` as `List<string>`, and `Ticks` for base duration. `RecipeRegistry` builds two indexes at load time — a reverse index keyed by machine id for fast runtime lookup, and `RecipeById` for validation. `Machine` instances fetch and cache their recipe list on construction. Smoke tested with the smelting recipe printing correctly on a placed furnace.

Key decisions: recipe owns the machine relationship not the other way around, `Speed` lives on the machine not the recipe, effective time = `Math.Floor(Ticks / Speed)` (ADR-013).

### Game startup and validation
Built a full startup validation pipeline. `GameStartup` orchestrates all validators, collects every error into one combined list, prints them all, then calls `GetTree().Quit()` — the game refuses to start with broken data. `RecipeValidator` checks inputs, outputs, and machine ids. `ResourceValidator` checks drop item ids. Inspired by the modded Minecraft experience of chasing errors one at a time (ADR-012).

### Tech debt cards added
- `MachineDefinition` split from `PlaceableDefinition`
- JSON loader refactor for array-per-file support
- Factory logic out of `PlayerController`

### Next session
Rewrite the JSON loader to support array-per-file loading, then begin the player inventory system.

---

## 7th June 2026

The foundation day. Everything from the base scene through to a working mining mechanic with visual feedback.

### Placeable system
Established the core architectural decision — `Placeable` is a plain C# class, not a Godot Node. Designed the definition/instance split: `PlaceableDefinition` holds static data loaded from JSON, `Placeable` instances hold runtime state. Built `PlaceableRegistry` as an Autoload using `JsonLoader` (a static utility using Godot's `FileAccess` and `DirAccess` APIs) and a custom `TupleConverter` for deserialising `Size` as a C# ValueTuple. Smoke tested end to end with `iron_ore.json` printing on load. Created the iron ore pixel art sprite.

### ResourceNode
Built `ResourceNode` extending `Placeable` with `RemainingStock`, `IsDepleted`, and `Extract(int amount)`. Added `DropDefinition` (ItemId + TotalAmount) as a nullable field on `PlaceableDefinition`. Refactored `GridWorld` from `Godot.Collections.Dictionary` to `System.Collections.Generic.Dictionary` to support plain C# objects. Created `PlaceableNode.tscn` with `PlaceableNode.cs` loading sprites from the registry. Iron ore sprites rendering correctly on placement.

### World scene and player movement
Renamed `GridTest` to `World`. Wired player movement with Camera2D parented to the player. Fixed grid rendering to update with camera movement via `QueueRedraw()` in `_Process`. Reorganised input — left click places, right click mines, middle click removes (dev tool).

### Mining mechanic
Implemented proximity-checked, held-input mining in `_Process` using `Input.IsActionPressed`. Added `(float Timer, Placeable Target)? activeMining` nullable tuple for state tracking. Mine time calculated as `hardness / miningSpeed`. Added `placedNodes` dictionary for O(1) node lookup. Fixed a rapid mouse-movement crash by guarding with `HasValue` before accessing the nullable tuple.

### Mining overlay
Added `MiningOverlay` as a child of `PlaceableNode` below `Sprite2D`. Renders a dark rect growing from the bottom of the cell upward based on `MiningProgress`. `PlaceableNode` delegates progress through to the overlay, keeping `World.cs` unaware of it.

### PlayerController refactor
Extracted player input handling from `World.cs` into a new `PlayerController` class extending `Node2D`. Set up `dotnet format` as a pre-build step and configured `.editorconfig` for consistent formatting.

### Item registry
Designed `ItemDefinition` with `Id`, `Name`, `Description`, `SpritePath`, `MaxStackSize`, and `ItemType` enum (`Solid`, `Fluid`, `Placeable`). `Placeable` type added so machines and vehicles can be carried in inventory and routed through item pipes. Built `ItemRegistry` following the same Autoload pattern as `PlaceableRegistry`.