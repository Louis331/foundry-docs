---
---

# Dev Log

[Home](index)

## 09 June 2026

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

## 08 June 2026

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

## 07 June 2026

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