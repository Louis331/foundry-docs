---
---

# Dev Log

[Home](index)

## 08 June 2026

### Summary
A long day - roughly 12 hours of solid development. Built the machine and recipe simulation foundation, added startup validation, and updated the documentation/project workflow. The architecture is in a genuinely solid place.

### Mining progress bar fix
Started the day by fixing the mining overlay progress bar which wasn't rendering correctly.

### Documentation site
Set up a public `foundry-docs` repo with GitHub Pages using the Jekyll Hacker theme with dark mode. Split documentation into `index.md`, `architecture.md`, and `plan.md`. Configured the site to suppress download buttons and the "View on GitHub" header. Live at `https://louis331.github.io/foundry-docs/`. Also set up a GitHub Projects kanban board with Backlog, In Progress, and Done columns with 22 cards ordered by dependency. Established a PR-based workflow for traceability.

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