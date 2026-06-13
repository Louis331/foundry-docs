---
---
## Architecture Decision Records

[Home](index) | [Architecture](architecture)

---

## Table of Contents

1. [ADR-001: Placeable is a plain C# class, not a Godot Node](#adr-001-placeable-is-a-plain-c-class-not-a-godot-node)
2. [ADR-002: Definitions are data-driven (JSON), behaviour is code](#adr-002-definitions-are-data-driven-json-behaviour-is-code)
3. [ADR-003: String IDs over integer indices for registry keys](#adr-003-string-ids-over-integer-indices-for-registry-keys)
4. [ADR-004: Lockstep co-op multiplayer is a design goal](#adr-004-lockstep-co-op-multiplayer-is-a-design-goal)
5. [ADR-005: Use "/" not Path.Combine for Godot resource paths](#adr-005-use--not-pathcombine-for-godot-resource-paths)
6. [ADR-006: World.cs is a signal-driven presenter, not a coordinator](#adr-006-worldcs-is-a-signal-driven-presenter-not-a-coordinator)
7. [ADR-007: Fluid covers liquids and gases](#adr-007-fluid-covers-liquids-and-gases)
8. [ADR-008: Placeable ItemType for deployable items](#adr-008-placeable-itemtype-for-deployable-items)
9. [ADR-009: Item IDs can match Placeable IDs without conflict](#adr-009-item-ids-can-match-placeable-ids-without-conflict)
10. [ADR-010: No separate MachineRegistry — machines extend PlaceableDefinition](#adr-010-no-separate-machineregistry--machines-extend-placeabledefinition)
11. [ADR-011: Negative PowerConsumption for power-producing machines](#adr-011-negative-powerconsumption-for-power-producing-machines)
12. [ADR-012: Full error list before crash on startup validation failure](#adr-012-full-error-list-before-crash-on-startup-validation-failure)
13. [ADR-013: Recipe duration is floored to nearest tick](#adr-013-recipe-duration-is-floored-to-nearest-tick)
14. [ADR-014: Command pipeline for all state mutations](#adr-014-command-pipeline-for-all-state-mutations)
15. [ADR-015: Cursor-based undo/redo history, not a stack](#adr-015-cursor-based-undoredo-history-not-a-stack)
16. [ADR-016: Simulation tick driven by physics frame, not a Timer](#adr-016-simulation-tick-driven-by-physics-frame-not-a-timer)
17. [ADR-017: DropDefinition separates drop amount from total stock](#adr-017-dropdefinition-separates-drop-amount-from-total-stock)
18. [ADR-018: Array-per-file is the standard shape for items and recipes](#adr-018-array-per-file-is-the-standard-shape-for-items-and-recipes)
19. [ADR-019: Separate Tick Systems for Networks vs Placeables](#adr-019-separate-tick-systems-for-networks-vs-placeables)

---

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

### ADR-017: `DropDefinition` separates drop amount from total stock

**Decision:** `DropDefinition` owns `ItemId` and `DropAmount` (per mine). Total stock lives on `PlaceableDefinition.MaxStock`, not on the drop.

**Rationale:**
- `TotalAmount` was doing double duty — it was both the per-mine yield and the total stock of the node
- These are independent values: a node might have 100 stock but drop 10 per mine
- Separating them makes both values independently tunable in JSON

**Consequence:** `ResourceNode` initialises `RemainingStock` from `PlaceableDefinition.MaxStock`. `DropDefinition.DropAmount` is the per-extract yield only.

---

### ADR-018: Array-per-file is the standard shape for items and recipes

**Decision:** Item and recipe JSON files contain a top-level array, even if the file has only one entry. Placeables remain one-object-per-file.

**Rationale:**
- Items and recipes come in families (ingots, ores, smelting recipes) — grouping them by family in a single file makes authoring and diffing practical
- A consistent array shape means the loader never needs to inspect file contents to decide how to deserialise
- Placeables are substantial standalone definitions tweaked in isolation — one-per-file matches that authoring pattern and there is no reason to disturb `PlaceableRegistry`

**Consequence:** `ItemRegistry` and `RecipeRegistry` use `LoadManyJson<T>`. `PlaceableRegistry` continues to use `LoadJson<T>`. Any new registry for family-style data should default to array-per-file.

---

### ADR-019: Separate Tick Systems for Networks vs Placeables

**Status:** Accepted

**Context:**
`GridWorld.Tickables` handles per-placeable tick logic (machines etc). As the sim grows, power and pipe networks need tick logic too — but they behave differently. A power network doesn't move energy cell-by-cell; it resolves as a whole graph each tick. Same for pipe networks. Lumping these into `Tickables` would force network logic into individual placeables and make the tick order semantics muddy.

**Decision:**
Tick systems are separated by concern:

- **`Tickables`** — discrete placeables with per-tick logic (machines). Iterated by world-space origin cell (X then Y) for deterministic order.
- **Power network** — ticks as a single system (BFS sweep). Gets its own dedicated tick call in `FactoryManager`, separate from `Tickables`.
- **Pipe networks** — same pattern as power; ticks as a system, not per-pipe.

**Consequences:**
- Tick ordering between systems is explicit and controlled in `FactoryManager`
- Network logic lives in the network system, not in individual placeables
- Future network types follow the same pattern — add a system-level tick call, not entries in `Tickables`
- Cache locality and iteration order concerns for `Tickables` do not apply to network systems, which manage their own data structures

---

### ADR-020: xUnit for unit testing; plain C# classes only

**Decision:** The test suite uses xUnit via a separate .NET project (`tests/Makwright.Tests.csproj`). Tests target plain C# simulation classes only. Classes with Godot dependencies are not tested at the unit level.

**Rationale:**
- xUnit runs via `dotnet test` with no Godot runtime — CI is trivial to wire up
- The simulation/presentation split (ADR-001) means the most important logic is already in plain C# classes with no scene tree dependency
- Attempting to test Godot-dependent classes (Nodes, GodotObject subclasses, classes calling Autoload singletons) requires either a running Godot runtime or significant mocking infrastructure — cost outweighs benefit at this stage

**What is testable:**
- Any class with no `using Godot` imports and no calls to Autoload singletons (e.g. `PlayerInventory`, `ResourceNode`, `ItemStack`, definition classes)
- New simulation classes should be written to this standard by default

**What is not testable (currently):**
- Any class extending a Godot type (`Node`, `Node2D`, `GodotObject`, etc.)
- Any class calling Godot APIs (`DirAccess`, `FileAccess`, `GD.Print`, etc.)
- Any class accessing a static Autoload singleton (`ItemRegistry.Instance`, `GridWorld.Instance`, etc.)

**Pattern for new testable classes:**
Where a class needs registry or Godot API access, inject it as a delegate rather than calling the singleton directly. See `PlayerInventory` as the established example — it takes `Func<string, ItemDefinition?>` rather than calling `ItemRegistry.Instance` directly. Production call sites pass the real lookup; tests pass a dictionary-backed lambda.

**Consequence:** Some existing classes (registries, commands, `JsonLoader`) cannot be unit tested without a refactor. These are not being retrofitted now — the pattern will be applied to new implementations going forward. The blockers and proposed fixes are documented in [testing.md](testing).