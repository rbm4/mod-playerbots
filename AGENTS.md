# AGENTS.md

## Scope

These instructions apply to the entire `mod-playerbots` repository.

## Project identity

`mod-playerbots` adds player-controlled alt bots and autonomous random bots to a World of Warcraft 3.3.5a AzerothCore server. It is not standalone. It requires the `Playerbot` branch of `mod-playerbots/azerothcore-wotlk`; stock AzerothCore is not a compatible build target.

The module combines four concerns:

1. AzerothCore integration through scripts, hooks, sessions, packets, and a dedicated database.
2. Bot lifecycle management for player-owned, random, addclass, and self bots.
3. A named-object AI framework composed of strategies, triggers, actions, values, and multipliers.
4. World simulation services such as travel, equipment, quests, guilds, battlegrounds, raids, dungeons, speech, and RPG behavior.

## Start here

Before changing code, read the documents relevant to the task:

- Documentation index and maintenance contract: [`.docs/README.md`](.docs/README.md)
- Architectural overview and dependency map: [`.docs/architecture/overview.md`](.docs/architecture/overview.md)
- Login, update, command, logout, and threading flows: [`.docs/architecture/runtime-lifecycle.md`](.docs/architecture/runtime-lifecycle.md)
- Strategy, trigger, action, value, and engine model: [`.docs/architecture/ai-engine.md`](.docs/architecture/ai-engine.md)
- Configuration, databases, SQL, caches, and localization: [`.docs/architecture/data-and-configuration.md`](.docs/architecture/data-and-configuration.md)
- Subsystem ownership and interaction catalog: [`.docs/subsystems/catalog.md`](.docs/subsystems/catalog.md)
- Safe AI extension workflow: [`.docs/development/extending-ai.md`](.docs/development/extending-ai.md)
- Feature documentation index: [`.docs/features/README.md`](.docs/features/README.md)

Use code as the final source of truth. When code and documentation disagree, verify the runtime path, fix the code or documentation as appropriate, and record the result in the same change.

## Source map

| Path | Responsibility |
|---|---|
| `src/Script/` | Module loader, AzerothCore hooks, command registration, secure login, world-thread operations |
| `src/Bot/` | `PlayerbotAI`, bot holders and managers, random bot lifecycle, command server, factories, AI engine |
| `src/Bot/Engine/` | Named contexts, strategy engine, queues, events, actions, triggers, values, multipliers |
| `src/Ai/Base/` | Shared gameplay behavior and global named-object registrations |
| `src/Ai/Class/` | Class and specialization behavior and class-specific context factories |
| `src/Ai/Raid/` | Raid encounter strategies, actions, triggers, and multipliers |
| `src/Ai/Dungeon/` | Dungeon encounter strategies, actions, and triggers |
| `src/Ai/World/` | Open-world and RPG behavior |
| `src/Mgr/` | Travel, movement, item, guild, talent, text, and security services |
| `src/Db/` | Playerbot persistence and read-through repositories |
| `src/PlayerbotAIConfig.*` | Runtime configuration loading and initialization orchestration |
| `conf/playerbots.conf.dist` | User-facing configuration contract and defaults |
| `data/sql/` | Dedicated playerbots schema, core database additions, base data, and updates |
| `.github/workflows/` | Supported build and static validation paths |

## Runtime model in one page

### Startup

`Addmod_playerbotsScripts()` in `src/Script/playerbots_loader.cpp` registers scripts through `AddPlayerbotsScripts()` in `src/Script/Playerbots.cpp`. Database loading registers `PlayerbotsDatabase`. Before world initialization, `PlayerbotAIConfig::Initialize()` creates or classifies random bot accounts, initializes managers and caches, builds all shared AI contexts, loads text and travel data, and initializes the spell repository.

### Ownership

- `PlayerbotsMgr` is the global registry that associates a player GUID with its `PlayerbotAI` and, for real players, its `PlayerbotMgr`.
- `PlayerbotMgr` belongs to one real master and owns that master's active alt bots.
- `RandomPlayerbotMgr` is a global `PlayerbotHolder` for autonomous bots and their population schedule.
- `PlayerbotHolder::botLoading` prevents duplicate asynchronous login attempts.
- A bot receives one `PlayerbotAI`, which owns combat, non-combat, and dead `Engine` instances.

### Update paths

- Map-thread path: `PlayerbotsPlayerScript::OnPlayerAfterUpdate()` calls `PlayerbotAI::UpdateAI()` and the master's `PlayerbotMgr::UpdateAI()`.
- World-thread path: `PlayerbotsWorldScript::OnUpdate()` drains `PlayerbotWorldThreadProcessor` and updates `RandomPlayerbotMgr`.
- Session path: custom playerbot hooks update random and master-owned bot sessions.

Do not move behavior between these paths without proving the target AzerothCore API is safe on that thread.

### AI decision path

`PlayerbotAI::DoNextAction()` selects the engine for combat, non-combat, or dead state. `Engine::DoNextAction()` checks active triggers, queues their handlers plus default actions, orders candidates by relevance, evaluates `isUseful()`, applies active multipliers, then evaluates `isPossible()`, runs prerequisites, executes one action, and queues continuers or alternatives.

Names are API keys. Lower-case space-separated strings connect strategies, triggers, actions, and values through creator maps. Parameterized objects use `name::qualifier`. Renaming a key requires searching all registrations, strategy bindings, chat commands, persisted values, and SQL data.

## Non-negotiable invariants

1. **Use the custom core.** Do not claim stock AzerothCore compatibility or validate only against it.
2. **Respect thread ownership.** Bot decision work runs during map updates. Cross-player and shared world mutations may require `PlayerbotWorldThreadProcessor`. Follow an existing operation pattern instead of calling Group, Guild, LFG, battleground, or login APIs from an arbitrary thread.
3. **Preserve one-owner registries.** A GUID must not acquire duplicate `PlayerbotAI`, `PlayerbotMgr`, holder membership, or pending login entries.
4. **Keep engine states separate.** Combat, non-combat, and dead engines have independent strategy sets and queues. Register or activate behavior in every intended state explicitly.
5. **Register every named object.** Implementing a class is insufficient. Add its creator to the correct global, class, raid, or dungeon context and connect it to a strategy.
6. **Treat names as contracts.** Exact strings are used for factory lookup, qualifiers, commands, strategy persistence, and diagnostics.
7. **Keep hot paths bounded.** Trigger checks, calculated values, target scans, and actions may run for thousands of bots. Reuse cached values and configured intervals. Avoid database queries in AI ticks.
8. **Keep configuration synchronized.** A normal AI option requires a documented entry in `conf/playerbots.conf.dist`, a field in `src/PlayerbotAIConfig.h`, and loading or validation in `src/PlayerbotAIConfig.cpp`. A narrowly scoped script option may instead be read directly through `sConfigMgr`, as `Playerbots.Updates.EnableDatabases` is, but its distributed default and use-site default must still match.
9. **Use the correct database.** Distinguish `PlayerbotsDatabase`, `CharacterDatabase`, `WorldDatabase`, and `LoginDatabase`. Do not place or query data in a convenient but incorrect schema.
10. **Make SQL updates forward-only and updater-compatible.** Put playerbots migrations in `data/sql/playerbots/updates/` using the established date sequence. Keep `custom/` SQL re-applicable.
11. **Preserve localized text behavior.** User-visible bot text belongs in `ai_playerbot_texts` and related update SQL when the existing text manager is used. Preserve default-locale fallback and placeholders.
12. **Do not weaken control boundaries.** Bot ownership and commands pass through `PlayerbotSecurity`, account-link checks, chat filters, and login collision handling.
13. **Document cross-cutting changes.** Any feature that changes ownership, lifecycle, threading, named-object registration, configuration, SQL, packet flow, or subsystem interaction must update `.docs` in the same change.

## Change workflow for agents

1. Read this file and the relevant `.docs` pages.
2. Trace the existing entry point through callers, named registrations, configuration, and persistence before editing.
3. Search exact string keys as well as C++ symbols.
4. Identify the execution thread and object owner for each changed path.
5. Prefer an existing neighboring pattern from the same scope: global, class, raid, dungeon, manager, or world script.
6. Implement the smallest complete vertical slice, including registration, configuration, SQL, and text where applicable.
7. Update an existing feature or subsystem document. For a new feature, copy `.docs/templates/feature.md` into `.docs/features/<feature-name>.md` and add it to `.docs/features/README.md`.
8. Verify formatting and compile against the custom core. Perform focused in-game validation when behavior cannot be covered automatically.
9. Review the diff for lifetime, thread, performance, and migration risks.

## Documentation contract

Documentation is part of the implementation, not a retrospective note.

Update documentation when any of these change:

- Entry points, call order, ownership, lifetime, or logout behavior
- Thread or queue boundaries
- Strategy, trigger, action, value, multiplier, or named context registration
- Configuration keys, defaults, validation, or reload behavior
- Database tables, updater paths, caches, or localization keys
- Commands, permissions, packets, or account-link behavior
- Supported raids, dungeons, maps, classes, or subsystems
- Build, formatting, static analysis, or manual verification steps

Use `.docs/templates/subsystem.md` when replacing undocumented legacy knowledge with a subsystem page. Each page must say what owns the subsystem, how control and data enter it, what it calls, what persists, which thread it runs on, how to extend it, and how to verify it.

## Verification

There is no dedicated C++ unit-test suite in this repository. Use the narrowest relevant checks and then the supported full build when practical:

- C++ formatting: `bash ./code_format.sh` in an environment with the expected `clang-format`
- Codestyle: `python apps/codestyle/codestyle-cpp.py` using the workflow's arguments
- Static analysis: `cppcheck --force --inline-suppr --suppressions-list=./.suppress.cppcheck src/`
- Linux build: place this repository at `modules/mod-playerbots` in the custom core, run CMake from the core root, then build
- Windows build: follow `.github/workflows/windows_build.yml`
- macOS build: follow `.github/workflows/macos_build.yml`

Always report what was and was not run. For behavior changes, record manual scenarios in the feature document and pull request, including bot type, class, map or encounter, command or trigger, expected behavior, observed behavior, and relevant load level.

## Known caution areas

- Bot login combines asynchronous character queries, fake `WorldSession` creation, and a queued world-thread completion operation. Preserve duplicate-login guards and object validity checks.
- Logout touches holder maps, AI persistence, group cleanup, sessions, and player destruction. Treat changes as lifetime-sensitive.
- The remote command server opens a configured TCP listener and is separate from normal chat command authorization. Treat exposure or command additions as security-sensitive.
- Random bot startup can create accounts and characters, classify account types, build large item caches, load travel data, and schedule many logins. Avoid multiplying startup or login database work.
- `conf/conf.sh.dist` references legacy `sql/...` assembler paths while repository SQL currently lives under `data/sql/...`. Verify parent-core installation behavior before changing either convention.
- Several caches are derived data. Identify the authoritative source table or game data before writing directly to cache tables.
- The public wiki is useful operator guidance but may be incomplete. Repository code and these versioned documents are the local engineering source of truth.
