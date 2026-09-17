# Feature documentation index

## Purpose

Feature pages preserve the end-to-end context for behavior that crosses subsystem boundaries. They explain the current implementation, not only the change that originally introduced it.

The architecture pages currently document the foundational existing features:

| Existing feature area | Current document |
|---|---|
| Master-owned bot login and control | [`../architecture/runtime-lifecycle.md`](../architecture/runtime-lifecycle.md) |
| Random bot population and lifecycle | [`../architecture/runtime-lifecycle.md`](../architecture/runtime-lifecycle.md) |
| Strategy, trigger, action, and value engine | [`../architecture/ai-engine.md`](../architecture/ai-engine.md) |
| Configuration, persistence, caches, and localization | [`../architecture/data-and-configuration.md`](../architecture/data-and-configuration.md) |
| Gameplay subsystem interactions | [`../subsystems/catalog.md`](../subsystems/catalog.md) |

Create dedicated pages here as a feature receives substantial work or when the architecture pages cannot hold its operational details safely.

## Naming

Use lower-case kebab-case file names:

```text
features/random-bot-level-brackets.md
features/account-linking.md
features/remote-command-server.md
features/icc-encounter-support.md
```

Name the behavior, not the ticket or contributor.

## Lifecycle

1. Copy [`../templates/feature.md`](../templates/feature.md).
2. Complete all applicable sections before implementation is considered complete.
3. Add the page to the active index below.
4. Keep the page synchronized as later changes modify the behavior.
5. Mark removed features as retired with replacement or migration notes. Do not silently delete historical operational context.

## Active feature pages

No dedicated feature pages have been added yet. When the first page is added, replace this sentence with an alphabetical table containing feature name, owner paths, status, and last validation date.

## Documentation triggers

A dedicated feature page is expected when a change includes any of these:

- New bot category, owner, login, logout, or session behavior
- New world-thread operation family
- New command or permission model
- New process-wide manager or cache
- New schema with runtime behavior
- New configuration family
- New remote or network-facing behavior
- A raid or dungeon implementation with several mechanics
- Behavior spanning three or more catalog subsystems
- A performance-sensitive feature designed for large bot populations

## Review expectation

A reviewer should be able to use the page to answer:

- Who starts the feature?
- Which bot categories and engine states does it affect?
- Which thread executes each stage?
- Which exact named-object keys connect it?
- What data survives restart?
- Which settings and permissions control it?
- What are failure and cleanup behaviors?
- Which scenarios prove it works?
