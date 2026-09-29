## Repo map

Chaptour has no application code yet. The repo holds decisions, not source, and the .NET/Azure stack is set by ADRs.

- `CONTEXT.md`: what Chaptour is, plus the domain glossary. Read it first and use its terms exactly.
- `docs/adr/`: decisions already made, one per file. Before proposing an approach, find the ADRs that touch it.
- `docs/research/`: cited research notes that feed decisions. They are background, not decisions.

The trunk is `main` (ADR-0014). Work lands through small PRs, and every merge must be shippable.

## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues (uses the `gh` CLI). See `docs/agents/issue-tracker.md`.

### Domain docs

Glossary in root `CONTEXT.md`, decisions in `docs/adr/`. See `docs/agents/domain.md` for how skills use them.
