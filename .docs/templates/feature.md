# {Feature name}

Status: Proposed | Active | Deprecated | Retired

Owners: `{primary source paths or maintainers}`

Last validated: YYYY-MM-DD against module `{branch or commit}` and core `{branch or commit}`

## Summary

What player, operator, or developer problem does this feature solve? Describe current behavior in a few sentences.

## Scope

### Included

- {Behavior included}

### Excluded

- {Related behavior intentionally outside this feature}

## User and operator experience

Describe commands, configuration, visible bot behavior, defaults, and error messages. Include exact command and config keys.

## Bot applicability

| Dimension | Applies to | Does not apply to |
|---|---|---|
| Bot category | {master-owned, random, addclass, self} | {categories} |
| AI state | {combat, non-combat, dead} | {states} |
| Class or role | {classes, specs, roles} | {exceptions} |
| Map or encounter | {maps} | {exceptions} |

## Entry points

| Trigger or entry | Source symbol | Execution thread | Preconditions |
|---|---|---|---|
| {chat, hook, packet, trigger, world update} | `{path and symbol}` | {map, world, callback, command server} | {conditions} |

## End-to-end control flow

```text
{entry}
  -> {owner}
  -> {decision or queue}
  -> {mutation}
  -> {response or cleanup}
```

Explain alternate, failure, retry, and cleanup paths.

## Ownership and lifetime

- Owning object or singleton:
- Per-player or per-bot state:
- Creation point:
- Destruction or reset point:
- Delayed work and validity strategy:
- Duplicate or reentrancy protection:

## AI integration

| Kind | Exact key | Registration | Consumer |
|---|---|---|---|
| Strategy | `{key}` | `{path}` | `{activation path}` |
| Trigger | `{key}` | `{path}` | `{strategy}` |
| Action | `{key}` | `{path}` | `{trigger, command, or packet}` |
| Value | `{key or key::qualifier}` | `{path}` | `{readers and writers}` |
| Multiplier | `{class or key}` | `{path}` | `{strategy}` |

State why each relevance and check interval was chosen.

## Threading and concurrency

- Thread for each stage:
- Shared structures touched:
- Queued `PlayerbotOperation` types:
- Locks or guards:
- Behavior if bot, master, group, map, or session disappears:
- Queue overflow or cancellation behavior:

## Configuration

| Key | Type | Default | Range or values | Reloadable | Effect |
|---|---|---:|---|---|---|
| `{exact key}` | `{type}` | `{default}` | `{constraints}` | Yes or No | {behavior} |

Describe interactions among keys and backward compatibility for default changes.

## Data model and persistence

| Database | Table or source | Authoritative or derived | Read path | Write path | Migration |
|---|---|---|---|---|---|
| {database} | `{table}` | {kind} | `{symbol}` | `{symbol}` | `{SQL file}` |

Document serialization, restart behavior, cache invalidation, clean rebuild, and data migration.

## Commands, packets, and security

- Commands and permission level:
- `PlayerbotSecurity` path:
- Account ownership or linking behavior:
- Packet opcodes and direction:
- Remote command exposure:
- Untrusted input validation:

## Dependencies and subsystem interactions

Link relevant pages and explain the relationship:

- Lifecycle:
- AI engine:
- Travel or movement:
- Items or economy:
- Groups, guilds, LFG, PvP:
- Text and localization:
- Custom AzerothCore APIs:

## Failure modes and recovery

| Failure | Detection | User-visible result | Cleanup or retry | Diagnostic |
|---|---|---|---|---|
| {failure} | {condition} | {result} | {behavior} | {log, metric, command} |

## Performance model

- Frequency per bot:
- Expected population multiplier:
- Expensive scans, allocations, queries, or logs:
- Cached values and intervals:
- Startup cost:
- `PerfMonitor` or benchmark plan:

## Observability

List log categories, metrics, debug actions, commands, and useful database queries. Do not include secrets or production credentials.

## Verification

### Automated

- [ ] Formatting
- [ ] Codestyle or static analysis
- [ ] Custom-core compile on relevant platforms
- [ ] Focused automated check if available

### Manual scenarios

| Scenario | Setup | Steps | Expected result | Result and date |
|---|---|---|---|---|
| {scenario} | {bot type, class, map, load} | {steps} | {result} | {not run or result} |

Cover happy path, denied path, invalid state, logout or reset, restart persistence, and representative population.

## Compatibility and rollout

- Required module and custom-core branches or commits:
- Existing configuration behavior:
- Existing database migration:
- Downgrade limitations:
- Operator action required:

## Decisions and rejected alternatives

Record important choices and why alternatives were not used. Keep this focused on facts that prevent future agents from repeating the same investigation.

## Open questions

- {Question with owner or condition for resolution}

## Change history

| Date | Change | Code reference | Documentation impact |
|---|---|---|---|
| YYYY-MM-DD | Initial documentation | `{commit or PR}` | Created |
