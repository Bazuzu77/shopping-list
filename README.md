# Family Grocery App

An offline-first household shopping app planned for iOS and Android. This repository currently contains approved behavior, architecture and implementation planning; application implementation has not started.

## Start here

Start with [how the app fits together](docs/architecture/system-overview.md), or use the [design reading guide](docs/architecture/README.md) to find an answer. The design pages use everyday language and examples; exact implementation details are in expandable engineering sections. Design work is tracked by [epic #33](https://github.com/Bazuzu77/shopping-list/issues/33).

Open the [project plan](https://github.com/users/Bazuzu77/projects/1/views/2), [work board](https://github.com/users/Bazuzu77/projects/1/views/3), or [dependency diagram](docs/backlog/dependencies.md). The project is private; repository documentation remains public.

1. [Product behavior](docs/specs/product-behavior.md) — current approved requirements.
2. [How offline changes are combined](docs/architecture/sync-data-design.md) — what happens when people edit or buy without internet.
3. [Backlog](docs/backlog/README.md) — ordered work and dependencies, independent of conversation context.
4. [The first working version](docs/superpowers/plans/2026-11-25-foundation-offline-core.md) — what we will build first and how we will check it.
5. [Remaining decisions](docs/decisions/remaining-decisions.md) — provider and engineering work.

[PLAN.md](PLAN.md) is the historical decision ledger. Its older open questions can be superseded by newer approvals. The original handover in `docs/reference` is preserved as provenance.

Planning baseline: Expo/React Native/TypeScript/SQLite mobile client; Go/PostgreSQL modular API; DigitalOcean hosting subject to later deployment approval. No provider, credentials or deployment is configured here.

Use GitHub issues for execution status once published. Keep durable specifications in the repository and link acceptance evidence from each completed issue.
