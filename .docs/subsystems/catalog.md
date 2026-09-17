# Subsystem catalog

## How to use this catalog

This is a routing map, not an exhaustive description of every class. Find the concern being changed, inspect its primary owners and integration points, then follow its links into the AI engine, lifecycle, configuration, and data layers.

If a subsystem acquires complex state or new cross-cutting behavior, create `subsystems/<name>.md` from [`../templates/subsystem.md`](../templates/subsystem.md) and link it from this page.

## Core runtime subsystems

### AzerothCore integration

**Purpose:** Register module scripts and translate custom core hooks into playerbot behavior.

**Primary owners:** `src/Script/Playerbots.cpp`, `src/Script/playerbots_loader.cpp`

**Inputs:** Database lifecycle, world startup and update, player update, chat, packets, battleground events, bot-session hooks, shutdown.

**Calls into:** Config initialization, global registries, bot managers, AI updates, world-thread processor, guild tasks.

**Change cautions:** A script class is inert until registered. Hook thread context differs. The API comes from the custom AzerothCore fork.

### Player-owned bot lifecycle

**Purpose:** Allow a real player to log in and control eligible alternate characters.

**Primary owner:** `PlayerbotMgr`, derived from `PlayerbotHolder`, in `src/Bot/PlayerbotMgr.*`

**Inputs:** `.playerbots bot` commands, real-player login and logout, incoming and outgoing master packets.

**Calls into:** Character login query holder, `WorldSession`, global `PlayerbotsMgr`, `PlayerbotAI`, group operations, account-link checks.

**Persistent data:** Character data in `CharacterDatabase`; selected AI state in `playerbots_db_store`; account links in the dedicated database.

**Thread:** Commands and callbacks initiate login; final holder registration is queued through the world-thread operation processor; bot AI later runs through map updates.

### Random bot population

**Purpose:** Maintain an autonomous server population and schedule bot activity.

**Primary owner:** `RandomPlayerbotMgr` in `src/Bot/RandomPlayerbotMgr.*`

**Supporting owners:** `RandomPlayerbotFactory`, `RandomBotLevelMgr`, `PlayerbotFactory`

**Inputs:** World update, config counts and timing, event rows, classified bot accounts, player login and logout, admin commands.

**Calls into:** Common holder login and logout, guild and battleground behavior, character generation, gear and talent initialization.

**Persistent data:** `playerbots_random_bots`, `playerbots_account_type`, character and account records.

**Thread:** Population management runs from the world update. Per-bot AI runs on map updates.

**Change cautions:** Database and login work is multiplied by population. Preserve event expiration, count convergence, and login throttling.

### Bot AI ownership and packet adaptation

**Purpose:** Own one bot's contexts, state engines, master relation, command queue, and packet-to-action handlers.

**Primary owner:** `PlayerbotAI` in `src/Bot/PlayerbotAI.*`

**Inputs:** Player map update, chat commands, master packets, bot packets, map and combat state.

**Calls into:** `Engine`, `AiObjectContext`, `PlayerbotSecurity`, values, actions, and domain managers.

**State:** Three engines plus per-bot named-object instances and packet queues.

## AI framework subsystems

### Named-object contexts

**Purpose:** Resolve stable string keys into per-bot strategy, action, trigger, and value objects.

**Primary owners:** `AiObjectContext`, `NamedObjectContext` templates, shared context builders in `src/Bot/Engine/`.

**Inputs:** Exact names, optional `::qualifier`, bot class.

**Calls into:** Global, class, raid, and dungeon creator maps.

**Change cautions:** Names are runtime contracts. Registration order can shadow keys. Created objects are cached.

### Decision engine

**Purpose:** Combine active strategies and execute the highest-relevance viable action.

**Primary owner:** `Engine` in `src/Bot/Engine/Engine.*`

**Inputs:** Strategies, trigger events, defaults, multipliers, action-node edges.

**Outputs:** One successful action per normal decision tick, plus queued follow-up candidates.

**Change cautions:** Engine state isolation, minimal-mode threshold, queue lifetime, action-node ownership, and per-bot performance.

### Class combat behavior

**Purpose:** Implement spells, roles, specializations, resources, buffs, and class-specific conditions.

**Primary paths:** `src/Ai/Class/Dk`, `Druid`, `Hunter`, `Mage`, `Paladin`, `Priest`, `Rogue`, `Shaman`, `Warlock`, `Warrior`.

**Registration:** Each `<Class>AiObjectContext` imports base contexts and adds class creators. `AiFactory` selects default strategies.

**Inputs:** Talents, spells, target and group values, combat state, strategy commands.

**Change cautions:** Specialization strategies may be siblings. Spell rank and game-data assumptions come from the 3.3.5a custom core.

### Raid and dungeon encounters

**Purpose:** React to instance mechanics with map-specific strategies, triggers, actions, and multipliers.

**Primary paths:** `src/Ai/Raid/`, `src/Ai/Dungeon/`

**Registration:** Strategy contexts, shared action and trigger builders, `PlayerbotAI::ApplyInstanceStrategies()`, and sometimes `AddPlayerbotsScripts()`.

**Inputs:** Map ID, creature or game object state, auras, positions, role, encounter packets or scripts.

**Calls into:** Movement, targeting, class actions, group coordination, generic boss helpers.

**Change cautions:** Strategy registration and map activation are separate. Test every role and phase with a group, not one bot alone.

## Gameplay service subsystems

### Movement and positioning

**Purpose:** Follow, stay, chase, flee, avoid hazards, reach targets, mount, taxi, and navigate encounters.

**Primary behavior:** `MovementActions`, `FollowActions`, `ReachTargetActions`, movement values in `src/Ai/Base/`.

**Supporting manager:** `FleeManager` in `src/Mgr/Move/`.

**Inputs:** Target positions, follow distance, line of sight, map collision, combat role, encounter strategies.

**Calls into:** AzerothCore movement generators and map APIs.

**Change cautions:** Movement commands can conflict. Check current movement, teleport, transport, root, and map-removal state.

### Travel graph and autonomous destinations

**Purpose:** Select long-distance quest, service, RPG, and exploration destinations and route bots through the world.

**Primary owner:** `TravelMgr` and types in `src/Mgr/Travel/`.

**Behavior adapters:** `ChooseTravelTargetAction`, `MoveToTravelTargetAction`, travel triggers and values.

**Persistent data:** `playerbots_travelnode`, `playerbots_travelnode_link`, `playerbots_travelnode_path`, and teleport cache.

**Inputs:** Map topology, quests, NPC services, level, strategy, travel target state.

**Change cautions:** Graph generation and loading can be expensive. Stored node and path formats are compatibility contracts.

### Open-world RPG simulation

**Purpose:** Make autonomous bots alternate among questing, grinding, travel, rest, social, economy, and PvP activities.

**Primary paths:** `src/Ai/World/Rpg/`, `NewRpgInfo`, `NewRpgStrategy`, and legacy RPG strategy code.

**Inputs:** Random-bot state, nearby targets, needs, level, travel targets, activity configuration.

**Calls into:** Travel, quests, economy actions, social actions, and PvP.

**Change cautions:** There are legacy and newer RPG flows. Trace which strategy is active before modifying shared actions.

### Items, equipment, loot, and maintenance

**Purpose:** Evaluate item use, loot, equip gear, maintain inventory, choose consumables, and initialize bot gear.

**Primary behavior:** Inventory, loot, equip, outfit, bank, item-use values and actions in `src/Ai/Base/`.

**Primary services:** `RandomItemMgr`, `BisListMgr`, `StatsWeightCalculator`, `PlayerbotFactory`.

**Persistent and cached data:** Enchants, weight scales, BIS data, equipment cache, random-item cache, rarity cache, item-info cache.

**Inputs:** World item templates, class, spec, level, budget, loot rules, config.

**Change cautions:** Determine source versus derived cache. Gear quality affects battleground matching, random bot maintenance, and combat behavior.

### Quests

**Purpose:** Accept, share, track, complete, reward, and drop quests and choose quest travel targets.

**Primary behavior:** Quest actions and values in `src/Ai/Base/Actions` and `src/Ai/Base/Value`.

**Inputs:** Master quest packets, quest log, world quest data, group state, travel candidates.

**Calls into:** Travel, inventory, NPC interaction, packet handlers, RPG state.

**Change cautions:** Master-driven and autonomous quest paths overlap. Verify packet and direct-action paths.

### Groups, LFG, and coordination

**Purpose:** Invite or remove members, lead, form raids, follow, ready-check, select roles, and enter LFG.

**Primary behavior:** Group and LFG actions, triggers, and values in `src/Ai/Base/`.

**Shared-world operations:** Group operations in `src/Script/WorldThr/PlayerbotOperations.h`.

**Inputs:** Master commands and packets, group state, random population, roles, instance plans.

**Change cautions:** Group mutation is thread-sensitive and bot or master logout can invalidate membership between scheduling and execution.

### Guilds and guild tasks

**Purpose:** Create and populate bot guilds, manage membership and bank interactions, and track guild tasks.

**Primary owners:** `PlayerbotGuildMgr`, `GuildTaskMgr` in `src/Mgr/Guild/`.

**Inputs:** Startup initialization, random bots, player kills, guild commands and actions.

**Persistent data:** Guild records in the character database, generated name pools, `playerbots_guild_tasks`.

**Change cautions:** Separate module cache, AzerothCore guild ownership, and database persistence.

### Battlegrounds, arenas, and PvP

**Purpose:** Queue appropriate bots and players, balance participation, form arena teams, and execute map tactics.

**Primary behavior:** `BattleGroundJoinAction`, `BattleGroundTactics`, PvP actions, triggers, and values.

**Primary owner for population data:** `RandomPlayerbotMgr::BattlegroundData`.

**Hook integration:** `PlayerBotsBGScript` in `src/Script/Playerbots.cpp` assigns per-instance faction strategies.

**Inputs:** Queue brackets, team and role counts, rating, bot gear and level, battleground map state.

**Change cautions:** Counters must remain balanced across join, invite, leave, logout, and battleground end. Test both factions and mixed real-player populations.

### Chat, commands, and text

**Purpose:** Parse player requests, expose administrative controls, convert packets to actions, and generate localized responses.

**Primary owners:** `PlayerbotCommandScript`, `PlayerbotAI::HandleCommand`, `ChatFilter`, `ChatHelper`, `PlayerbotTextMgr`.

**Inputs:** Whisper, party, raid, guild, server command, packet event, and optional TCP remote command.

**Persistent data:** Localized text, chance tables, account-link data, custom strategies.

**Change cautions:** There are multiple authorization paths. Exact action names and placeholders are contracts.

### Security and account linking

**Purpose:** Decide who may talk to, invite, or fully control a bot and which accounts may access each other's characters.

**Primary owner:** `PlayerbotSecurity` in `src/Mgr/Security/`.

**Lifecycle checks:** `PlayerbotHolder::AddPlayerBot()`, account commands in `PlayerbotMgr`.

**Persistent data:** Account keys, reciprocal links, and account types.

**Inputs:** Master identity, GM state, faction, group leadership, level or gear checks, config flags.

**Change cautions:** Do not bypass checks in a new command. Remote commands require a separate threat review.

### Text and localization

**Purpose:** Load named bot responses, choose locale and variation, apply chance, and replace placeholders.

**Primary owner:** `PlayerbotTextMgr` in `src/Mgr/Text/`.

**Persistent data:** `ai_playerbot_texts` and `ai_playerbot_texts_chance`.

**Inputs:** Active session locale distribution, caller-provided placeholder values.

**Change cautions:** Preserve default text fallback. Add SQL updates for runtime text changes.

### Performance diagnostics

**Purpose:** Measure AI, trigger, action, value, and related costs and expose detailed action logging.

**Primary owner:** `PerfMonitor` in `src/Bot/Debug/`.

**Inputs:** Instrumented engine and AI operations, config logging options.

**Change cautions:** Diagnostics can themselves become expensive at large population. Test with representative bot counts.

## Data-flow examples

### Autonomous questing

```text
RandomPlayerbotMgr population
  -> PlayerbotAI non-combat engine
  -> RPG or travel strategy
  -> quest and travel values
  -> choose travel target action
  -> TravelMgr route
  -> movement actions
  -> quest NPC or objective interaction
  -> inventory and quest state updates
```

### Encounter reaction

```text
map ID activates instance strategy
  -> encounter Trigger reads aura, unit, position, or role values
  -> TriggerNode queues encounter Action
  -> Multiplier suppresses conflicting generic behavior
  -> movement or class Action executes
  -> group and packet state affects next tick
```

### Master command

```text
chat hook
  -> PlayerbotAI::HandleCommand
  -> security and chat filtering
  -> named action or strategy change
  -> current or selected engines
  -> response through PlayerbotTextMgr or chat helper
```

## When one change crosses subsystems

Create a feature page if a change affects three or more rows above, or if it crosses lifecycle, security, threading, configuration, or persistence. The feature page should link the subsystem owners and record the end-to-end control and data flow.
