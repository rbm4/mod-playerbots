# mod-playerbots engineering documentation

Last source review: 2026-09-16

## Purpose

This directory preserves implementation context that is otherwise spread across C++ creator maps, AzerothCore hooks, configuration, and SQL. It is written for agents and maintainers who must change the module without rediscovering its architecture each time.

This is not a replacement for the public player and installation wiki. The wiki explains how to operate the module. These documents explain how the implementation fits together and how to evolve it safely.

## Reading paths

### First change in this repository

1. Read [`../AGENTS.md`](../AGENTS.md).
2. Read [`architecture/overview.md`](architecture/overview.md).
3. Choose the relevant architecture or subsystem page below.
4. Check [`features/README.md`](features/README.md) for feature-specific constraints.

### AI behavior change

1. [`architecture/ai-engine.md`](architecture/ai-engine.md)
2. [`development/extending-ai.md`](development/extending-ai.md)
3. [`subsystems/catalog.md`](subsystems/catalog.md)

### Lifecycle, login, logout, command, packet, or threading change

1. [`architecture/runtime-lifecycle.md`](architecture/runtime-lifecycle.md)
2. [`architecture/overview.md`](architecture/overview.md)

### Configuration, SQL, cache, or localization change

1. [`architecture/data-and-configuration.md`](architecture/data-and-configuration.md)
2. The relevant subsystem in [`subsystems/catalog.md`](subsystems/catalog.md)

## Document map

| Document | Question answered |
|---|---|
| [`architecture/overview.md`](architecture/overview.md) | What are the major components and dependency directions? |
| [`architecture/runtime-lifecycle.md`](architecture/runtime-lifecycle.md) | How do bots start, log in, update, receive commands, and log out? |
| [`architecture/ai-engine.md`](architecture/ai-engine.md) | How does the named-object decision engine choose behavior? |
| [`architecture/data-and-configuration.md`](architecture/data-and-configuration.md) | Where do settings and persistent or cached data live? |
| [`subsystems/catalog.md`](subsystems/catalog.md) | Which subsystem owns a gameplay concern and what does it depend on? |
| [`development/extending-ai.md`](development/extending-ai.md) | What registrations and checks are required for new AI behavior? |
| [`features/README.md`](features/README.md) | Which cross-cutting features have dedicated documentation? |
| [`templates/feature.md`](templates/feature.md) | How should a new feature be documented? |
| [`templates/subsystem.md`](templates/subsystem.md) | How should an existing subsystem be documented in depth? |

## Source-of-truth order

When sources disagree, use this order and resolve the drift:

1. Executed code in the matching custom AzerothCore and module branches
2. SQL schema or update files and `conf/playerbots.conf.dist`
3. This versioned engineering documentation
4. Repository README and CI workflows
5. External wiki, issue, or chat guidance

Line numbers are deliberately avoided in most documentation because this codebase changes frequently. References use symbols and repository-relative paths. Search the named symbol before editing.

## Maintenance rules

### Update an existing page when

- A documented call flow or owner changes.
- A subsystem gains or loses a dependency.
- A named context, config key, SQL table, cache, command, or thread boundary changes.
- A caution or invariant is no longer accurate.

### Add a feature page when

The work spans more than one of these surfaces:

- AzerothCore hooks or packets
- Bot lifecycle or ownership
- AI registrations
- Configuration
- SQL or caches
- Commands or security
- More than one gameplay subsystem

Copy [`templates/feature.md`](templates/feature.md) to `features/<lowercase-kebab-name>.md`, complete every applicable section, and add it to the feature index.

### Add a subsystem page when

A subsystem has enough internal state or interactions that the catalog entry cannot explain it safely. Copy [`templates/subsystem.md`](templates/subsystem.md) to `subsystems/<lowercase-kebab-name>.md`, then link it from the catalog.

### Quality standard

A useful document answers:

- Why does this component exist?
- Who owns it and how long does it live?
- Which thread executes it?
- How do control and data enter and leave it?
- What is persistent, cached, or derived?
- Which exact registrations make it reachable?
- What can break when it changes?
- How can a maintainer verify it?

Do not copy large code blocks. Prefer symbols, exact string keys, short call chains, tables, and diagrams that reveal relationships.

## Review checklist

- [ ] Links resolve within the repository.
- [ ] Every named source path exists.
- [ ] Thread and ownership statements were checked against current code.
- [ ] New config keys and SQL objects are named exactly.
- [ ] Feature and subsystem indexes include new pages.
- [ ] The document states verification limits rather than implying unrun tests passed.
- [ ] Historical notes are labeled as such and do not override current behavior.
