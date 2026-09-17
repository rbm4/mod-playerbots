# Data and configuration architecture

## Configuration contract

The user-facing source of truth is `conf/playerbots.conf.dist`. Most runtime fields are declared in `src/PlayerbotAIConfig.h` and loaded in `PlayerbotAIConfig::Initialize()` or a focused reload helper in `src/PlayerbotAIConfig.cpp`. Narrow integration settings can be consumed directly through `sConfigMgr` in a script, as `Playerbots.Updates.EnableDatabases` is in `PlayerbotsDatabaseScript`.

The config object is process-wide and controls much more than decision tuning. It covers:

- Module enablement and bot counts
- Random-bot accounts, maps, level distribution, login cadence, and behavior
- AI timing, activity budgets, strategies, cheats, and performance logging
- Ownership and command permissions
- Travel, RPG, quests, economy, guilds, LFG, battlegrounds, and arenas
- Gear, consumables, talents, and maintenance
- Remote command server
- Playerbots database connection and updater behavior

### Adding or changing a setting

A normal AI config change includes:

1. A documented key and default in `conf/playerbots.conf.dist`.
2. A correctly typed field in `src/PlayerbotAIConfig.h`.
3. Loading with the same key and default in `src/PlayerbotAIConfig.cpp`.
4. Validation or normalization near loading when invalid input can harm runtime behavior.
5. Reload integration if the setting is expected to respond to `.reload config`.
6. Documentation of affected owners, threads, and behavior.
7. Manual verification for default, intended override, and invalid boundary values.

A setting used only by a narrow module script can omit the `PlayerbotAIConfig` field and be read directly with `sConfigMgr`. Keep that exception local rather than scattering repeated config reads through AI behavior.

Do not silently change the code default without changing the distributed config default. Search the exact key because old aliases or related settings may already exist.

### Initialization is orchestration

Late in `PlayerbotAIConfig::Initialize()`, the module performs this approximate order:

```text
create or update random bot characters
  -> assign account types
  -> initialize random-bot manager if enabled
  -> initialize guild manager
  -> initialize item caches and BIS data
  -> load localized text and text chances
  -> initialize PlayerbotFactory
  -> build all shared AI contexts
  -> optionally load dungeon suggestions
  -> initialize TravelMgr
```

Adding a manager to this sequence creates an ordering dependency. Document its prerequisites and shutdown behavior.

## Database boundaries

The module reads or writes four logical databases.

| Database | Typical ownership | Examples |
|---|---|---|
| `LoginDatabase` | Accounts and authentication | Account lookup, bot account creation or classification inputs |
| `CharacterDatabase` | Character and social state | Character login query holders, character names, guild and arena data |
| `WorldDatabase` | Static game-world data | Vendor items, creatures, quests, DBC-derived tables |
| `PlayerbotsDatabase` | Module-owned persistent and derived data | Bot state, text, account links, custom strategies, item caches, travel graph |

The dedicated connection is registered by `PlayerbotsDatabaseScript` in `src/Script/Playerbots.cpp`. Core declarations for the connection and prepared statements live in the required custom AzerothCore fork, not in this repository.

Before adding a query, verify:

- Which component owns the data.
- Whether a prepared statement already exists in the custom core.
- Whether the call can occur on a hot map-thread path.
- Whether the result should be loaded once, cached, or queried on demand.
- Whether the write needs a schema migration and rollback-independent forward behavior.

## Playerbots schema groups

Base definitions live under `data/sql/playerbots/base/`.

### Runtime state and identity

| Table | Purpose |
|---|---|
| `playerbots_db_store` | Generic per-bot strategy and value persistence |
| `playerbots_random_bots` | Random-bot event and scheduling state |
| `playerbots_account_type` | Unassigned, random-bot, and addclass account classification |
| `playerbots_account_keys` | Hashed linking keys |
| `playerbots_account_links` | Reciprocal trusted-account links |
| `playerbots_custom_strategy` | Runtime custom strategy definitions |

### Text and behavior data

| Table | Purpose |
|---|---|
| `ai_playerbot_texts` | Named text entries with locale columns |
| `ai_playerbot_texts_chance` | Probability controls for named text |
| `playerbots_speech` and probability tables | Speech content and selection behavior |
| `playerbots_dungeon_suggestion_*` | Dungeon suggestion definitions, abbreviations, and strategies. The existing abbreviation table is misspelled `playerbots_dungeon_suggestion_abbrevation`; treat that exact name as the current schema contract. |
| `playerbots_preferred_mounts` | Preferred mount data |
| `playerbots_guild_tasks` | Guild task definitions or state |

### Item and character-generation data

| Table | Purpose |
|---|---|
| `playerbots_enchants` | Enchantment definitions |
| `playerbots_weightscales` and `playerbots_weightscale_data` | Stat-weight definitions |
| `playerbots_equip_cache` | Derived equipment candidates |
| `playerbots_item_info_cache` | Derived item information and weight-scale fields |
| `playerbots_rnditem_cache` | Derived random-item selections |
| `playerbots_rarity_cache` | Derived item rarity information |

### Travel data

| Table | Purpose |
|---|---|
| `playerbots_tele_cache` | Level-related teleport positions |
| `playerbots_travelnode` | Travel graph nodes |
| `playerbots_travelnode_link` | Directed or linked graph edges |
| `playerbots_travelnode_path` | Stored edge paths |

## Core database additions

`data/sql/characters/base/` includes name pools and related character-side bot data. `data/sql/world/base/` includes static world or DBC-derived data required by bot behavior.

These files target their named core database, not `PlayerbotsDatabase`. If a feature needs both core and module data, document install and migration behavior for each schema.

## SQL updater model

The dedicated database base contains AzerothCore updater metadata:

- `updates.sql`
- `updates_include.sql`
- `version_db_playerbots.sql`

`updates_include` points the updater at these source-relative paths, where `$` is the updater's source-directory token:

- `$/data/sql/playerbots/updates`
- `$/data/sql/playerbots/custom`
- `$/data/sql/playerbots/archive`

### Migration rules

- Place new forward migrations in `data/sql/playerbots/updates/`.
- Use `YYYY_MM_DD_NN[_description].sql` for new updates. A few legacy files omit the sequence number, but they are exceptions rather than the pattern for new migrations.
- Do not edit a released migration to represent a new state. Add a later update.
- Make migration intent clear through deterministic SQL.
- Keep `custom/` SQL re-applicable with patterns such as `CREATE IF NOT EXISTS`, `REPLACE`, or explicit delete and insert.
- Update base SQL when project release practice requires clean installations to include the current schema. Confirm the established repository convention before duplicating update changes.
- Use the correct character or world update path for changes outside the dedicated database.

The updater can be disabled with `Playerbots.Updates.EnableDatabases`, so code should fail clearly when a required schema is absent rather than corrupting unrelated data.

## Generic AI persistence

`PlayerbotRepository` persists bot strategy strings for combat, non-combat, and dead states plus generic context values in `playerbots_db_store`.

`AiObjectContext::Save()` only emits values whose `Save()` result is not `?`. It stores the exact value key and serialized text. `Load()` resolves that key through current creator maps.

Consequences:

- Renaming a persisted value key is a data migration.
- Changing serialization requires backward compatibility or migration.
- A new `Value` is not automatically persistent unless its save and load behavior supports it and the repository flow includes it.
- Strategy names are persistent contracts too.

## Caches and provenance

### Item caches

`RandomItemMgr` initializes equipment, random item, rarity, item-info, ammo, food, potion, trade, enchant, and teleport-related data. Some tables exist to avoid recomputing expensive world-data joins at every startup or bot action.

Before modifying a cache table directly, find the manager's load, build, clear, and save paths. Determine whether the table is:

- Authoritative configuration
- Generated from `WorldDatabase`
- Generated from config and character level
- Safe to rebuild
- Version-sensitive to the custom core's game data

### Spell and vendor cache

`PlayerbotSpellRepository::Initialize()` builds skill-spell data from DBC stores and a vendor-item set from `WorldDatabase`. It is initialized after the main config orchestration in the world startup hook.

### Travel graph

`TravelMgr` owns the world travel model and uses the travel-node tables. Graph generation and persistence can be expensive. Changes to node identity, links, or path formats must account for existing stored data and clean rebuild behavior.

### Guild and text caches

`PlayerbotGuildMgr` caches generated guild names and bot guild state. `PlayerbotTextMgr` loads named text and chance values into memory for runtime selection.

## Localization

`ai_playerbot_texts` contains a default `text` column and locale columns. `PlayerbotTextMgr` selects by client locale and falls back to the default text when the localized entry is empty.

When adding user-visible bot text through this system:

1. Add or update it through SQL in the correct base and update convention.
2. Keep the `name` key stable and unique for its intended use.
3. Preserve placeholders expected by the caller.
4. Supply a default text even when translations are not available.
5. Add chance data if selection is probabilistic.
6. Verify at least default locale fallback and placeholder substitution.

Do not hard-code a new translatable response in an action when neighboring behavior uses `PlayerbotTextMgr`.

## Security-related data

Account linking uses `playerbots_account_keys` and reciprocal rows in `playerbots_account_links`. Ownership checks combine config flags, account identity, guild relation, addclass classification, and account links.

Changes to these tables or commands must consider:

- Existing linked pairs
- Reciprocal insertion and deletion
- Key reset behavior
- Online login ownership checks
- Disclosure in command responses
- Remote command paths that do not share chat authorization

## Legacy assembler path caveat

`include.sh` sources `conf/conf.sh.dist`, whose assembler variables currently point at module-relative `sql/auth`, `sql/characters`, and `sql/world` paths. The repository's files are under `data/sql/`, and there is no matching top-level `sql/` tree in this checkout.

Do not "fix" this in isolation. First inspect how the matching custom core installs, copies, or interprets module SQL paths on each supported platform. Record the verified behavior in this document when resolved.

## Data-change verification

- Start from a clean dedicated database and confirm base plus updates reach the expected revision.
- Upgrade an existing database through the new update.
- Start with updater enabled and disabled and verify failure behavior.
- Verify the exact database connection used by each query.
- Rebuild or reload affected caches and compare counts or representative records.
- Exercise default and non-default locale text.
- Test restart persistence for strategy or value changes.
- Measure startup and runtime cost when adding cache construction or queries.
