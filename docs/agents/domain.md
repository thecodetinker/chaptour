# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

This repo's domain model borrows Eric Evans' Domain-Driven Design vocabulary: `CONTEXT.md` holds the **ubiquitous language** (the precise, agreed vocabulary for the domain — see "Use the glossary's vocabulary" below), and `CONTEXT-MAP.md` + per-context `CONTEXT.md` files (see "File structure") model **bounded contexts** — the boundaries within which a given vocabulary holds. This "Context" is a DDD term, not a C4 one: it's about where a *vocabulary* applies, not about what a *system* talks to. If this repo ever adopts C4 diagrams (system/container/component views — see `docs/agents/architecture.md` if it exists), note that C4's "System Context diagram" uses the same word for something different: the system and its external actors/dependencies, not a vocabulary boundary. Don't conflate the two — a domain's bounded Context and a C4 System Context diagram can disagree about where their boundaries are drawn, and that's fine; they're answering different questions.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root, or
- **`CONTEXT-MAP.md`** at the repo root if it exists: it points at one `CONTEXT.md` per context. Read each one relevant to the topic.
- **`docs/adr/`**: read ADRs that touch the area you're about to work in. In multi-context repos, also check `src/<context>/docs/adr/` for context-scoped decisions.

If any of these files don't exist, **proceed silently**. Don't flag their absence; don't suggest creating them upfront. The `/domain-modeling` skill (reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates them lazily when terms or decisions actually get resolved.

## File structure

Single-context repo (most repos):

```
/
├── CONTEXT.md
├── docs/adr/
│   ├── 0001-event-sourced-orders.md
│   └── 0002-postgres-for-write-model.md
└── src/
```

Multi-context repo (presence of `CONTEXT-MAP.md` at the root):

```
/
├── CONTEXT-MAP.md
├── docs/adr/                          ← system-wide decisions
└── src/
    ├── ordering/
    │   ├── CONTEXT.md
    │   └── docs/adr/                  ← context-specific decisions
    └── billing/
        ├── CONTEXT.md
        └── docs/adr/
```

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in `CONTEXT.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal: either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding:

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because…_
