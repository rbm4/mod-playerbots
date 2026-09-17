# AI engine architecture

## Mental model

The AI framework is a per-bot, string-keyed decision graph. C++ classes provide behavior, creator maps expose those classes under exact names, strategies connect observations to actions, and an engine selects one action per update.

```text
Strategy
  -> TriggerNode: trigger name + action handlers
  -> default NextAction entries
  -> optional Multiplier entries
  -> optional ActionNode prerequisite, alternative, continuer graph

Engine tick
  -> resolve and check Trigger objects
  -> enqueue handlers and default actions by relevance
  -> resolve Action object
  -> apply Multiplier objects
  -> isUseful -> isPossible -> prerequisites -> Execute
  -> enqueue continuers on success or alternatives on failure
```

## Construction

### Shared creator registries

At startup, `PlayerbotAIConfig::Initialize()` calls `AiObjectContext::BuildAllSharedContexts()`.

`AiObjectContext` builds shared registries for:

- `Strategy`
- `Action`
- `Trigger`
- `UntypedValue`

The base registries are assembled in:

- `src/Bot/Engine/BuildSharedStrategyContexts.cpp`
- `src/Bot/Engine/BuildSharedActionContexts.cpp`
- `src/Bot/Engine/BuildSharedTriggerContexts.cpp`
- `src/Bot/Engine/BuildSharedValueContexts.cpp`

`BuildAllSharedContexts()` then builds each class-specific registry. Class contexts call the base builder and add their own creators.

### Per-bot context

`AiFactory::createAiObjectContext()` selects an `AiObjectContext` subclass from the bot class. Each bot receives a context with access to the shared creator maps and per-bot caches of created objects.

A repeated lookup of the same qualified name normally returns the same per-bot action, trigger, strategy, or value object. Do not treat resolved objects as short-lived temporaries.

### Three engines

The `PlayerbotAI` constructor creates:

- `BOT_STATE_COMBAT`
- `BOT_STATE_NON_COMBAT`
- `BOT_STATE_DEAD`

Each engine has its own active strategies, trigger nodes, multipliers, action-node factories, and action queue. State-specific behavior must be registered and activated in every state where it should run.

## Named-object resolution

`NamedObjectFactory<T>` maps an exact string to a creator function. `NamedObjectContext<T>` adds creation caching. Shared and per-bot context lists combine creator sources.

### Naming rules

- Use lower-case, space-separated keys: `low health`, `frostbolt`, `follow`.
- Parameterized names use `name::qualifier`.
- `AiObjectContext::GetValue<T>(name, param)` constructs a qualified key.
- Class names and file names are not lookup keys unless explicitly registered under that name.
- Creator-map order can shadow an existing key. Search globally before adding or renaming one.

### Name dependency example

```text
Mage strategy InitTriggers()
  -> TriggerNode("brain freeze", NextAction("frostfire bolt", relevance))
  -> Mage trigger context creator["brain freeze"]
  -> Mage action context creator["frostfire bolt"]
```

All strings must match. This linkage is checked at runtime, not compile time.

## Strategies

`Strategy` is the composition unit. It can define:

- `getDefaultActions()` for filler or baseline work
- `InitTriggers()` for observation-to-action bindings
- `InitMultipliers()` for context-sensitive relevance changes
- `actionNodeFactories` for prerequisites, alternatives, and continuers
- `GetType()` flags for combat, non-combat, tank, heal, DPS, melee, or ranged traits

Relevance constants are defined in `src/Bot/Engine/Strategy/Strategy.h`. Higher relevance wins. Emergency actions normally outrank raid reactions, which outrank normal and default behavior.

### Activation

Strategies can be:

- Added by `AiFactory` as class and role defaults
- Added or removed by player commands
- Loaded from persisted bot state
- Applied automatically for an instance map
- Mutually exclusive through sibling-enabled contexts

Any strategy-set change must cause `Engine::Init()` so trigger, multiplier, and action-node collections are rebuilt.

### Sibling strategies

A context created with sibling support treats its registered strategies as mutually exclusive. This is used for class specializations and similar modes. Adding one causes the engine to remove its siblings. Register unrelated strategies in a separate context.

## Triggers and events

A `Trigger` observes current bot state. Its `Check()` returns an `Event` when active. `needCheck()` uses a check interval to avoid evaluating every trigger on every tick.

A `TriggerNode` belongs to a strategy and combines:

- Trigger name
- One or more `NextAction` handlers

`Event` transports a source name, parameter, packet, and owner. Actions reached from a trigger receive that event.

### Trigger design

- Keep `IsActive()` observational and cheap.
- Put reusable or expensive state in a calculated value.
- Use an appropriate check interval for conditions that need not react every tick.
- Do not mutate shared game state from a trigger.
- Bind the trigger in the intended strategy and state. Registration alone does not make it run.

## Actions and action nodes

`Action` is the mutation unit. The important lifecycle is:

1. `isUseful()`
2. Multiplier application
3. `isPossible()`
4. Prerequisites, if any
5. `Execute(event)`
6. Continuers on success or alternatives on failure

`ActionNode` provides dependency edges around an action name:

- Prerequisites must run before the action.
- Alternatives are considered when the action is impossible or fails.
- Continuers are considered after success.

A strategy's action-node factory can override the plain node for a given action. If no factory provides a node, `Engine::CreateActionNode()` creates one without edges.

### Action design

- `isUseful()` should answer whether the desired result is currently needed.
- `isPossible()` should answer whether execution can currently succeed.
- `Execute()` should perform one bounded operation and report its real result.
- Avoid repeating expensive target selection across all three methods. Use values.
- Set the next AI delay through established action or AI patterns when timing matters.
- Shared-world mutation may need a queued `PlayerbotOperation` rather than direct execution.

## Values

`Value<T>` exposes state through `Get`, `Set`, `Reset`, and related methods. `CalculatedValue<T>` computes and caches a result for a configured interval.

Common access macros are in `src/Script/Playerbots.h`:

- `AI_VALUE(type, name)`
- `AI_VALUE2(type, name, param)`
- `AI_VALUE_LAZY(type, name)`
- `SET_AI_VALUE(type, name, value)`

### Value design

- Prefer one reusable value over duplicate scans in triggers and actions.
- Choose cache intervals from freshness needs and population cost.
- Make qualifier semantics explicit and stable.
- Reset dependent cached values when a mutation invalidates them, following neighboring patterns.
- Only values with meaningful `Save()` output participate in generic context persistence.

`AiObjectContext::Save()` serializes created values whose `Save()` does not return `?`. `Load()` resolves the same name and calls `Load(text)`. Renaming such a value can strand persisted state.

## Multipliers

A `Multiplier` changes an action's relevance. Each active strategy can add multipliers during engine initialization. The engine multiplies relevance in sequence and stops considering the candidate when relevance reaches zero or below.

Use multipliers for broad suppression or promotion of existing actions under a mode or encounter. Use a trigger for a positive observation that should schedule a specific response.

## Engine tick in detail

`Engine::DoNextAction()` performs:

1. Optional value logging.
2. `ProcessTriggers(minimal)`.
3. `PushDefaultActions()`.
4. Compute an iteration budget from queue size and `iterationsPerTick`.
5. Pop the highest-relevance basket.
6. Resolve the action through `AiObjectContext`.
7. Evaluate usefulness.
8. Apply all active multipliers.
9. Evaluate possibility.
10. Push prerequisites and requeue the original action if needed.
11. Execute through action listeners.
12. Push continuers on success or alternatives on failure.
13. Stop after a successful action or when the budget is exhausted.
14. Remove expired queued work.

Minimal mode only considers relevance at or above 100. Behavior that must run while a bot is passive or throttled must account for this threshold deliberately.

## Class-specific pattern

A class context such as `MageAiObjectContext` contains internal factories and shared registries. Its builders first import base creators, then add class-specific strategy, action, trigger, and value creators.

`AiFactory` selects default strategies based on class, specialization, role, and bot state. For a class feature, inspect both the class context and `AiFactory`; registration and default activation are separate decisions.

## Raid and dungeon pattern

Encounter behavior generally includes:

- A strategy registered in `RaidStrategyContext` or `DungeonStrategyContext`
- Trigger classes and a trigger context
- Action classes and an action context
- Optional multipliers
- Shared builder registration for action and trigger contexts
- Map activation in `PlayerbotAI::ApplyInstanceStrategies()`
- For some encounters, an additional AzerothCore script registered by `AddPlayerbotsScripts()`

`ApplyInstanceStrategies()` removes every known instance strategy from combat and non-combat engines, maps the current map ID to one strategy key, and adds it to both engines. A new map strategy is incomplete if it is registered but absent from that method's known list and switch.

## Runtime custom strategies

`CustomStrategy` reads rows from `playerbots_custom_strategy`. It provides data-driven action lines without recompiling, but every referenced action and trigger name still needs a C++ creator. Treat the database format and qualifier semantics as a compatibility contract.

## Diagnostics

Useful inspection paths include:

- Engine action logs and unknown-action messages
- `check values` and value formatting actions
- `PerfMonitor` metrics for action, trigger, value, and AI costs
- Strategy list and change commands
- Exact creator-map searches for a failing string key

When a behavior does not run, check in this order:

1. Is the strategy active in the current engine state?
2. Was `Engine::Init()` called after activation?
3. Is the trigger registered under the exact name?
4. Is the trigger active and due for checking?
5. Is the action registered under the exact handler name?
6. Did a multiplier reduce relevance to zero?
7. Did `isUseful()` or `isPossible()` reject it?
8. Did a prerequisite or higher-relevance action consume the tick?
9. Is the bot in minimal mode?

## Performance rules

- Never query a database from a trigger, multiplier, or frequently read value.
- Avoid allocating large collections every tick when a context value can cache them.
- Bound nearby-object and group scans.
- Reuse exact context values rather than introducing parallel caches.
- Consider the product of active bots, active strategies, trigger count, and check frequency.
- Use `PerfMonitor` and a realistic bot population for verification.
