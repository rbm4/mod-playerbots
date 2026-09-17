# Extending the playerbot AI

## First decision: choose the narrowest scope

| Behavior scope | Location and registration |
|---|---|
| Shared by many classes and modes | `src/Ai/Base/` and base contexts |
| Specific to one class or specialization | `src/Ai/Class/<Class>/` and `<Class>AiObjectContext` |
| Specific to a raid | `src/Ai/Raid/<Raid>/`, raid contexts, map activation |
| Specific to a dungeon | `src/Ai/Dungeon/<Dungeon>/`, dungeon contexts, map activation |
| Open-world autonomous behavior | `src/Ai/World/` plus base registrations as needed |
| Shared process service or cache | Existing manager pattern in `src/Mgr/`, not an action singleton |

Prefer a neighboring implementation with the same scope. Do not register encounter-only keys globally unless they are truly reusable.

## Design the behavior chain

Write the intended chain before coding:

```text
observation or command
  -> trigger or direct action
  -> values required
  -> action candidate and relevance
  -> usefulness and possibility conditions
  -> mutation
  -> follow-up, response, or persisted state
```

Also identify:

- Intended bot states: combat, non-combat, dead
- Intended classes, specs, roles, maps, and bot categories
- Execution thread
- Expected check interval and population cost
- Exact string keys
- Configuration and SQL needs

## Add an action

1. Derive from the nearest suitable action base.
2. Give it an exact lower-case key through `getName()` or the established constructor pattern.
3. Implement `isUseful()` for whether the result is needed.
4. Implement `isPossible()` for whether execution can work now.
5. Implement a bounded `Execute(Event)` that returns true only on success.
6. Reuse context values for target selection and expensive state.
7. Add prerequisites, alternatives, or continuers through an action-node factory when needed.
8. Register the creator in the correct action context.
9. Connect the action to a trigger, default list, command, packet handler, or action-node edge.

Common registration locations:

- Global: `src/Ai/Base/ActionContext.h`
- Chat: `src/Ai/Base/ChatActionContext.h`
- Packet: relevant world-packet action context
- Class: `<Class>AiObjectContext.cpp`
- Encounter: `<Encounter>ActionContext.h` plus `BuildSharedActionContexts.cpp`

## Add a trigger

1. Derive from `Trigger` or a focused existing trigger base.
2. Implement a cheap observation through `IsActive()` or `Check()`.
3. Choose a check interval appropriate to reaction speed.
4. Register the exact key in the correct trigger context.
5. Add a `TriggerNode` to a strategy with one or more action handlers and explicit relevance.
6. Verify that strategy is active in the intended engine state.

Registration alone does not schedule checks. A strategy must include the trigger node.

## Add a value

1. Choose mutable `Value<T>` or cached `CalculatedValue<T>` based on ownership.
2. Define stable qualifier semantics if parameterized.
3. Implement calculation, formatting, reset, and persistence only as needed.
4. Register the creator in the correct value context.
5. Use `AI_VALUE`, `AI_VALUE2`, or direct context access consistently with neighboring code.
6. Select a cache interval that balances freshness and server cost.
7. Find mutations that must invalidate the cached result.

If `Save()` and `Load()` are implemented, the key and serialized form become persistent data contracts.

## Add a strategy

1. Derive from the nearest strategy base.
2. Define an exact `getName()` key.
3. Set `GetType()` flags accurately for combat role and range.
4. Add default actions only for safe fallback behavior.
5. Add trigger handlers in `InitTriggers()`.
6. Add relevance modifiers in `InitMultipliers()` only when broad suppression or promotion is required.
7. Register the strategy creator in the right strategy context.
8. Decide whether it is default, command-activated, persisted, or map-activated.
9. Ensure every strategy change rebuilds engine state through the established engine methods.

For mutually exclusive modes, use an existing sibling-enabled factory or create a separate sibling context rather than manual removal logic.

## Add class behavior

A class feature commonly touches:

- Action and trigger declarations and definitions
- `<Class>AiObjectContext.cpp` creator maps
- One or more specialization or non-combat strategies
- `AiFactory` only when default strategy selection changes
- Text SQL for player-visible responses

Verification matrix:

- Every affected specialization
- Combat and non-combat transitions
- Solo, master-led group, and autonomous random bot where applicable
- Level or spell-rank boundaries
- Target missing, dead, immune, out of range, and out of line of sight

## Add a raid or dungeon encounter

A complete encounter integration can require:

1. A strategy class and key.
2. Trigger classes for observable mechanics.
3. Action classes for responses.
4. Trigger and action context creator maps.
5. Strategy registration in `RaidStrategyContext` or `DungeonStrategyContext`.
6. Context registration in the shared trigger and action builders.
7. Optional multipliers to suppress generic movement or casting.
8. The strategy key in `allInstanceStrategies`.
9. The map ID mapping in `PlayerbotAI::ApplyInstanceStrategies()`.
10. An encounter script registration in `AddPlayerbotsScripts()` if custom core events are needed.
11. Configuration and text SQL when behavior is tunable or user-visible.

Encounter verification should cover all roles, major phases, wipe and reset, player-led and bot-led groups, entrance or late join, death and resurrection, and conflict with generic follow, flee, attack, and loot behavior.

## Add a packet-driven behavior

1. Identify whether the packet originates from the master or bot and whether it is incoming or outgoing.
2. Register opcode to action key in the appropriate `PlayerbotAI` handler table.
3. Register the action in a context available to the intended engines.
4. Parse only packet fields valid for the matching custom-core opcode definition.
5. Handle stale object GUIDs and missing world objects.
6. Confirm packet copies and queues remain bounded.

Do not assume an opcode and packet layout from stock AzerothCore without checking the required custom branch.

## Add a world-thread operation

Use a queued operation when shared-world APIs must not run in the bot's map update.

1. Derive from `PlayerbotOperation`.
2. Store GUIDs and immutable input instead of delayed raw pointers.
3. Implement `GetName()`, validity, and execution. The base exposes priority, but the current processor is FIFO and does not use it for ordering.
4. Re-resolve world objects at execution time.
5. Make disappearance a safe no-op or explicit failure.
6. Queue through `PlayerbotWorldThreadProcessor`.
7. Test queue delay, logout before execution, duplicate scheduling, and shutdown.

Do not split one invariant across direct map-thread mutation and later queued cleanup.

## Add configuration

For a normal AI setting, use the complete config contract:

```text
conf/playerbots.conf.dist
  <-> PlayerbotAIConfig field
  <-> exact GetOption key and default
  -> validation and optional reload
  -> feature use site
```

A narrowly scoped script integration setting may read `sConfigMgr` directly instead of adding a `PlayerbotAIConfig` field. Keep that read local and keep its use-site default synchronized with `conf/playerbots.conf.dist`.

Document units, valid range, default, restart or reload semantics, and interactions with related keys.

## Add persistent data or text

- Choose the correct database.
- Add a forward SQL update with the repository naming convention.
- Update clean-install base data according to established release practice.
- Use a repository or manager boundary rather than embedding SQL in a trigger or hot action.
- For localized output, use `PlayerbotTextMgr` and preserve default fallback and placeholders.
- Define whether data is authoritative, derived, or disposable cache.

## Debugging checklist

### Unknown or inactive behavior

- Search the exact key across creator maps and strategy bindings.
- Confirm shared contexts are built after registration.
- Confirm the class-specific context is selected.
- Confirm strategy activation in the current bot state.
- Confirm trigger interval and condition.
- Inspect multiplier suppression.
- Inspect minimal mode and relevance threshold.
- Check prerequisites and higher-priority candidates.

### Wrong bot or owner

- Confirm holder membership.
- Confirm random, addclass, self-bot, or real-player classification.
- Confirm `PlayerbotAI::master` and group leader.
- Confirm security level and account-link checks.

### Intermittent crash or state corruption

- Trace the caller thread.
- Look for a pointer crossing callback, operation queue, map transition, or logout.
- Check duplicate login guards and registry removal order.
- Check group or session mutation outside established thread paths.
- Reproduce during teleport, logout, death, instance reset, and server shutdown.

### Performance regression

- Measure with realistic active bot counts.
- Use `PerfMonitor` by action, trigger, and value.
- Check calculated-value cache intervals.
- Count nearby-world and group iterations.
- Check database queries or logging in update paths.

## Completion checklist

- [ ] Implementation is in the narrowest correct scope.
- [ ] Every exact key is registered once in the intended context.
- [ ] Strategy binding and activation are complete.
- [ ] Intended engine states are explicit.
- [ ] Thread and owner are documented.
- [ ] Config declaration, loading, default, and documentation match.
- [ ] SQL update, base data, repository, and cache provenance are complete.
- [ ] Localized text and placeholders are complete.
- [ ] Performance cost is bounded and measured where needed.
- [ ] Feature or subsystem documentation is updated.
- [ ] Formatting, static checks, custom-core build, and manual scenarios are reported accurately.
