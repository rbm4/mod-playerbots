# {Subsystem name}

Status: Active | Deprecated | Retired

Primary owners: `{source paths}`

Last validated: YYYY-MM-DD against module `{branch or commit}` and core `{branch or commit}`

## Responsibility

What does this subsystem own? State what it deliberately does not own.

## Public surface

| Surface | Symbol or key | Caller | Result |
|---|---|---|---|
| {hook, method, command, config, table} | `{exact name}` | `{caller}` | {result} |

## Ownership and lifetime

| Object or state | Cardinality | Created | Reset or destroyed | Owner |
|---|---:|---|---|---|
| {object} | {one process, per player, per bot} | {point} | {point} | {owner} |

Document raw-pointer relationships and delayed-work validity rules.

## Execution contexts

| Flow | Entry point | Thread | May block | Shared state touched |
|---|---|---|---|---|
| {flow} | `{symbol}` | {map, world, callback, session, remote} | Yes or No | {state} |

## Control flow

```text
{input}
  -> {owner}
  -> {processing}
  -> {output}
```

Include startup, normal operation, error, reset, and shutdown flows.

## Data flow and state

| Data | Source of truth | Cache | Readers | Writers | Invalidation |
|---|---|---|---|---|---|
| {data} | {config, DB, core object} | {cache} | {readers} | {writers} | {rule} |

## AI framework integration

List exact strategy, trigger, action, value, qualifier, and multiplier keys. Identify registration contexts and active engine states.

## Configuration

| Exact key | Default | Consumer | Reload semantics | Interactions |
|---|---:|---|---|---|
| `{key}` | {value} | `{symbol}` | {restart or reload} | {related keys} |

## Persistence and SQL

| Database | Table | Purpose | Migration path | Authoritative or derived |
|---|---|---|---|---|
| {database} | `{table}` | {purpose} | `{path}` | {kind} |

## Dependencies

### Calls into

- {Subsystem and reason}

### Called by

- {Subsystem and reason}

### Custom AzerothCore contracts

- {Hook, class, prepared statement, or behavior not provided by stock core}

## Invariants

1. {Invariant that must remain true}

## Failure modes

| Failure | Effect | Detection | Recovery |
|---|---|---|---|
| {failure} | {effect} | {diagnostic} | {recovery} |

## Performance characteristics

Document frequency, scaling factor, expensive operations, cache intervals, queue limits, and realistic load assumptions.

## Security boundaries

Document authorization, ownership, untrusted input, network exposure, and sensitive data handling.

## Extension guide

Give the smallest complete steps to add behavior without breaking registration, threading, persistence, or cleanup.

## Verification matrix

| Scenario | Setup | Expected result | Diagnostic |
|---|---|---|---|
| {scenario} | {setup} | {result} | {log, metric, query} |

## Known constraints and technical debt

Record current limitations as facts. Link an issue when one exists. Do not present speculation as established behavior.

## Related documentation

- [`../architecture/overview.md`](../architecture/overview.md)
- {Other relevant page}

## Change history

| Date | Change | Code reference |
|---|---|---|
| YYYY-MM-DD | Initial documentation | `{commit or PR}` |
