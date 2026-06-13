---
---
## Testing

[Home](index) | [Architecture](architecture) | [ADRs](adr)

---

## Running the tests

```bash
dotnet test tests/Makwright.Tests.csproj
```

Tests run without a Godot runtime. No editor or export step required.

---

## What we test

Plain C# simulation classes with no Godot dependencies. If a class has no `using Godot` imports and doesn't call an Autoload singleton, it's a test target.

Current coverage:
- `PlayerInventory` — stacking, overflow, hotbar selection, `InventoryChanged` event
- `ResourceNode` — extract, depletion, overshoot clamping
- `ItemStack` — pure data class
- `ItemDefinition`, `PlaceableDefinition`, `RecipeDefinition` — JSON deserialisation, enum parsing
- `TupleConverter` — custom JSON converter for `(int X, int Y)`

---

## What we don't test (and why)

Anything with a Godot dependency cannot be unit tested without a running Godot runtime:

| Class | Blocker |
|---|---|
| `PlaceCommand`, `RemoveCommand`, `ExtractCommand` | Extend `GodotObject`; call `GridWorld.Instance` |
| `GridWorld` | Godot `Node2D` singleton |
| `PlaceableRegistry`, `ItemRegistry`, `RecipeRegistry` | Godot Autoload lifecycle; use `FileAccess`/`DirAccess` |
| `JsonLoader` | `DirAccess.GetFilesAt`, `FileAccess.Open` |
| `GameStartup`, validators | Godot lifecycle; read Autoload singletons |
| All Nodes (`World`, `Hud`, `PlayerController`, etc.) | Godot scene tree |

These are not being retrofitted. See below for the pattern to apply on new implementations.

---

## Pattern for new testable classes

If a new class needs registry or Godot API access, inject it as a delegate rather than calling the singleton directly.

**`PlayerInventory` is the template:**

```csharp
// Testable — lookup is injected
public PlayerInventory(Func<string, ItemDefinition?> getItemDefinition, Action<string>? logMessage = null)

// Production call site in Player.cs
Inventory = new PlayerInventory(id => ItemRegistry.Instance.GetItem(id));

// Test
var lookup = new Dictionary<string, ItemDefinition> { ... };
var inventory = new PlayerInventory(id => lookup.GetValueOrDefault(id));
```

The rule: **new simulation classes accept lookup delegates, not Autoload singletons.**

---

## Blocked refactors (future work)

The following refactors would unlock tests for currently blocked classes. Not being done now — captured here for when the relevant code is being touched anyway.

- **`JsonLoader`** — split Godot I/O from deserialisation; inject file source as delegates
- **Registry builders** — extract plain builder classes from Autoload Node shells; accept `IEnumerable<TDefinition>` instead of reading files directly
- **`GridSimulation`** — extract placement/removal logic from `GridWorld` Node into a plain C# class; inject into commands
- **Command handlers** — extract `Execute`/`Rollback` logic from `GodotObject` command wrappers into plain handler classes
- **Validators** — parameterise with registry dictionaries instead of reading `*.Instance`

See ADR-020 for the full decision and rationale.