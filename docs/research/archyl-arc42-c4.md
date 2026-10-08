# Archyl for arc42-style docs and C4 diagrams (issue #9)

Research note for [thecodetinker/chaptour#9](https://github.com/thecodetinker/chaptour/issues/9), a child of map [#7](https://github.com/thecodetinker/chaptour/issues/7). It extends [`archyl-adr-sync.md`](archyl-adr-sync.md) and does not repeat it. That note covers the `adrs.folder` question, the sync Action's internals, ADR mapping, API-key scope and the Node 24 caveat. This note covers everything else: documentation features, C4 modelling, DSL import/export, doc sync from a repo, the free plan, and API/CI options.

Method and sources: primary only. `archyl.com/docs` pages (read via their `/index.md` Markdown twins where available), the DSL JSON Schema (`https://api.archyl.com/api/v1/dsl/schema`), the OpenAPI document (`https://api.archyl.com/docs/openapi.json`, Swagger 2.0), Archyl's own blog post on arc42 and the `llms.txt` index, and the plan constants compiled into Archyl's public web bundle. Everything was fetched unauthenticated on 2026-10-08. No account was created, no credential was entered and no authenticated API was called. Statements marked **inference** or **unverified** are not in a primary source.

## Short answers

1. **Archyl has no arc42 support.** Its own blog says "archyl holds the C4 model, not an arc42 document," and that "the text sections stay in your arc42 document, linking to the model" ([arc42 vs C4 Model](https://www.archyl.com/blog/arc42-vs-c4-model/index.md)). Nothing in the docs, the DSL schema or the OpenAPI mentions arc42 templates, sections or structure.
2. **Archyl should be a synced view, not the canonical home, for arc42.** Archyl documentation is a flat list of Markdown pages with tags and links to C4 elements, plus folders. The free plan is capped at 100 model objects and has no webhooks or shared links. The repo stays the source of truth, and Archyl can mirror the C4 model, ADRs and chosen Markdown pages from it.
3. **The C4 model is a strong fit and Archyl's real strength.** All four C4 levels, relationships, overlays, a drawn diagram per level, export to PNG/SVG/PDF, and round-trip to YAML. Archyl YAML has no `views` section, so diagram layout and which elements appear at which level are not controlled from the repo.
4. **Whether `docs:` in `archyl.yaml` is resolved server-side through `/dsl/ingest` is as unverified as `adrs.folder` was.** The same two-possibility analysis from the ADR note applies. The DSL schema offers `docs.records[].file` and `docs.folder`. The CI Action still uploads only the YAML text.

## Factory vs Chaptour-specific

Issue #9 asks each resolution to say which parts are reusable factory and which are Chaptour-specific.

| Finding | Factory (reusable on another project) | Chaptour-specific |
|---|---|---|
| Archyl is C4 + ADRs + docs-as-pages, not arc42. Pair it with arc42 kept in the repo. | Yes | |
| `archyl.yaml` is the repo-held contract for the C4 model. Sync via `archyl-com/actions/sync` (pinned SHA). | Yes | |
| Free-plan constraints (100 model objects, 3 AI discoveries, no webhooks/shared links) and the BYO-AI-provider escape hatch. | Yes (limits apply to any project on the free plan) | |
| Generating `archyl.yaml` `docs.records` and `adrs.records` from repo files at CI time (script), if server-side folder resolution is absent. | Yes (a script/skill that is project-agnostic) | |
| Which C4 elements Chaptour actually has (the model contents), the project name, the Azure/.NET technologies tagged. | | Yes |
| Where arc42 section files live (`docs/arc42/...`) and how they are linked to `CONTEXT.md` and `docs/adr/`. | Convention is factory, paths are decided in a sibling ticket | Chaptour's content |
| The `ARCHYL_API_KEY` exception to ADR-0013. | The exception pattern is factory | The key and project UUID are Chaptour-specific |

## Documentation features

Source: [Documentation & ADRs](https://www.archyl.com/docs/features/documentation), [Attachments](https://www.archyl.com/docs/features/documentation-attachments), [Concepts](https://www.archyl.com/docs/getting-started/concepts), OpenAPI.

- **Content type.** Documentation pages are Markdown: "headings and formatting," "code blocks with syntax highlighting," "tables and lists," "images and links." Pages are created in the Docs tab, or imported from a Git repo with **Documentation Discovery** (Project Settings, then "Discover Docs", then review and approve). The docs page does not mention Mermaid or other diagram-in-Markdown support, so assume none. Embedding Archyl diagrams into *other* tools is documented (below); embedding diagrams into an Archyl page is not.
- **Structure.** There is no template mechanism and no arc42 or any other section scheme. The API has doc **folders** (`/projects/{projectId}/doc-folders`, `/doc-folders/{id}/move`, MCP `*_documentation_folder` tools) and pages that can be moved between folders. The `DocumentationResponse` schema is `id, projectId, title, slug, content, tags, filePath, createdAt, updatedAt`, so a page can record the repo file it came from (`filePath`). The user-facing docs page does not describe folders, so the UI behaviour is **unverified**.
- **Links to C4 elements.** Documentation can be linked to elements: the docs page gives "Link API documentation to Service containers" as a use case, the API has `/documentation/by-element` and `/documentation/{id}/links`, and the DSL has `docs.records[].links` (element paths in dot notation, e.g. `Payment Service.API Gateway`). This is the one genuine arc42-to-C4 bridge: a section page (say "Building Block View") can be linked to the containers it describes.
- **ADRs.** Title, status (`proposed|accepted|deprecated|superseded`), context, decision, consequences, tags, and links to elements (`link_adr_to_element` MCP tool). Details of repo import are in the ADR note.
- **Attachments.** Drag and drop, paste, or Attach in the editor. Default maximum "10 MB per file," any file type. Images (PNG, JPEG, GIF, WebP, SVG, AVIF) render inline. Storage is a private bucket with short-lived signed links.
- **Export of documentation.** The docs page links to "Export documentation with diagrams" and the sharing page says PDF export gives "multi-page documents with: Title page, Diagram, Element descriptions, Relationship details." Neither says that Markdown *pages* are included in an export. The YAML export schema (`C4ExportData`) does include `documentation` and `documentationFolders`, so the API can return the page content (**inference** that this is the full Markdown; field-level content not read).
- **Search/versioning.** Not documented for pages. Nothing primary describes page history, so do not rely on it. The map ticket should treat git as the only version history.
- **Comparison with what arc42 needs.** An arc42 document is 12 fixed sections with cross-references (e.g. section 5 building blocks referring to section 6 runtime scenarios, section 9 pointing at ADRs). Archyl could hold 12 pages (or a folder of them), but it enforces none of that, has no template, and shows no cross-page link features beyond Markdown links.

## C4 modelling and diagrams

Source: [C4 Model Overview](https://www.archyl.com/docs/c4-model/overview), [Concepts](https://www.archyl.com/docs/getting-started/concepts), [Flows](https://www.archyl.com/docs/features/flows), DSL schema.

- **Levels.** All four: System Context, Container, Component, and Code (optional). Navigation is by double-click to zoom a level and breadcrumbs up.
- **Authoring model.** Diagrams are drawn from the model. You edit by dragging between elements to create relationships. Colours, custom icons and "grouping with overlays" are supported. The docs do not say whether layout is auto-generated or manual, nor whether manual layout persists. The DSL has no coordinates or layout, so layout cannot come from the repo.
- **Element schema (DSL).** Hierarchical `systems > containers > components > code`, each with tags and technologies; `relationships` use dot-notation endpoints, a `type` enum (`uses`, `depends_on`, `calls`, `reads_from`, `writes_to`, `sends_to`, `receives_from`, `publishes_to`, `consumes_from`, `deployed_on`, and others), technologies and style; `overlays` are named groups with a level (1 to 4); plus `environments`, `releases`, `technologies`, `api_contracts`, `events`. The schema has **no `views`/`diagrams` section**, so which elements appear on which view is not expressible.
- **Dynamic and deployment diagrams.** Archyl's arc42 blog maps a C4 *Dynamic* diagram to arc42 section 6 and a *Deployment* diagram to section 7, but the product docs only document **Flows** (not "formal C4 dynamic diagrams": the Flows page describes sequences of steps through the architecture, created in the UI or by an AI generator, with "public flow links" and embedding) and, in the DSL, `environments` and the `deployed_on` relationship type. There is no documented deployment diagram feature. Flows are not in the DSL schema, so they cannot live in the repo. **Unverified:** whether a deployment view exists in the UI.
- **Diagram export.** PNG (standard/high/print), SVG, PDF; background, quality and viewport options; "Export all levels" gives separate files per level, delivered as a ZIP ([Architecture as Code and export](https://www.archyl.com/docs/features/export), [Sharing](https://www.archyl.com/docs/features/sharing)). This is manual in the UI. No documented API endpoint renders a diagram image (the API overview says it can "export diagrams" without paths, and the OpenAPI lists `/projects/{id}/export` which returns the model as JSON, not images). So committing generated PNG/SVG from CI is **not documented as possible**.
- **Sharing/embedding.** Shareable links (view level, navigation toggle, expiry from 1 hour to permanent, optional password), plus embeds for Confluence, Notion, GitBook (Markdown), and "any platform supporting iframes or oEmbed." Shared links are a paid feature on this plan structure (see below), so a public-repo README cannot rely on a free-plan embed.

## DSL import and export

Source: [Architecture as Code](https://www.archyl.com/docs/features/export), DSL schema, OpenAPI.

Five import formats, selectable under Import Project (new project) or into an existing project:

| Format | Status | Notes |
|---|---|---|
| Archyl YAML (`archyl.yaml`) | Native, full | All C4 levels, technologies, environments, releases, ADRs, docs, API contracts, event channels. `version: "1.0"` required, `include:` for monorepo splitting. |
| Structurizr DSL (`.dsl`) | Supported | Extracts persons, systems, containers, components, relationships, tags, technology. **"Views, styles, themes, and deployment nodes are skipped (Archyl has its own visual layer)."** Multi-file `!include` workspaces must be uploaded as a `.zip` (limits: 10 MiB, 500 files, 50 MiB unpacked, 5 MiB per file). `import_dsl` MCP tool and repo sync read one file and do not resolve `!include`. |
| LikeC4 (`.c4`, `.likec4`) | Supported; Archyl says it is "the first tool to import LikeC4 files" | Handles `specification` custom kinds, nesting, technology, hashtag tags, `#external`, merged `model` blocks. Views are not mentioned as imported. **Unverified** for LikeC4 view blocks. |
| IcePanel JSON, Backstage JSON | Supported | Not relevant to Chaptour. |

Export: **Archyl YAML only** (`GET /projects/{id}/dsl/export`, or the Export button), described as "a complete `archyl.yaml` that you can check into your repository." There is **no Structurizr or LikeC4 export** in the docs, schema or OpenAPI (the OpenAPI `dsl` paths are `import`, `import-archive`, `validate`, `validate-archive`, `export`, `/dsl/import-project-archive` and `/dsl/schema`; `ImportDSLRequest.format` is `archyl|structurizr|likec4`). The JSON `GET /projects/{id}/export` returns `C4ExportData` (systems, relationships, overlays, ADRs, ADR folders, documentation, documentation folders, API contracts, event channels, releases, environments, technologies).

Consequences for Chaptour:

- Import is one-way into Archyl for Structurizr/LikeC4, and the visual layer (views, styles) is discarded. If the team ever keeps C4 as Structurizr DSL or LikeC4 in the repo as the "real" source (which would also give repo-rendered diagrams), Archyl would see only the structure and not the curated views.
- Archyl YAML is the only format Archyl can both read and write, so it is the only option that round-trips. Choosing `archyl.yaml` as the repo-held C4 source is lock-in to Archyl's schema (it is a public JSON Schema, which softens that). Choosing Structurizr or LikeC4 as the repo source is lower lock-in but only structure transfers, and Archyl can import but cannot give it back.
- Because the Action takes a single file, a Structurizr or LikeC4 file would be sent through `/dsl/ingest` with its own `format`? **Unverified.** Archyl documents `format` on `/dsl/import` (OpenAPI `ImportDSLRequest`) and on the `import_dsl` MCP tool, but the `sync` Action's source (see the ADR note) sends only `{content}` to `/dsl/ingest`, which is documented as "JSON-formatted YAML content." Assume the CI Action handles Archyl YAML only.

## Doc sync from a repo

Three mechanisms exist, none verified to be a deterministic Markdown mirror:

1. **Documentation Discovery** (UI, and `POST /documentation/discovery`). "AI-powered documentation discovery from a repository." Request: `owner`, `repository`, `projectId` (required), optional `branch`, `folderPath`, `provider`, `baseUrl`, `accessToken`. Asynchronous (202), review and approve. This is the same family as ADR Discovery, which the ADR note observed consuming the free-plan AI quota.
2. **`docs:` in `archyl.yaml`.** `docs.folder` ("Path to docs folder in repo") and `docs.records[].file` ("Path to markdown file in repo") or `content` (inline). Whether `/dsl/ingest` fetches those files (needs the repo connected) or only stores the path is the same open question as `adrs.folder`. **Unverified.** The Architecture as Code page shows the `file:` form as an example without saying how it is read.
3. **Inline `content` generated in CI.** The only fully documented path: a CI step reads `docs/arc42/*.md` and writes them into `docs.records[].content` in a temporary YAML file, which the Action uploads. Deterministic, uses no AI quota, needs no repo connection. Cost: one more script to maintain, and the whole doc set travels inside one request (size limits not documented).

Webhooks for push-triggered discovery are "Coming Soon" on the [Git integration page](https://www.archyl.com/docs/git-integration/overview) (unchanged since the ADR note). Webhooks that exist today are *outbound* from Archyl and gated by plan (see below). The AI Discovery page does say incremental analysis can run "through webhooks or GitHub Actions on push events," and the Actions page lists six official Actions: `sync`, `drift-score`, `release`, `conformance-check`, `generate-context`, `auto-cr`. Only `sync` and `release` need no `organization-id`.

## Canonical home or synced view?

**Recommendation: synced view.** Reasons, each from a primary source:

- Archyl itself positions arc42 text as living outside the tool ("The text sections stay in your arc42 document").
- No arc42 template, section structure or cross-page validation exists.
- Page history, Markdown diagrams and review workflow are not documented. Git gives all three.
- The free plan caps model objects at 100 and gives no shared links or webhooks, so it is not a publishing platform for an open-source-style doc set either.
- Content in Archyl is in a proprietary store with only YAML round-trip export, whereas the repo is the shippable, reviewable artifact under ADR-0014.
- AGENTS.md already says the repo "holds decisions"; agents work against files, not a SaaS UI (though Archyl's MCP server and `generate-context` Action can feed agents the C4 model).

What Archyl *should* own, as a derived view: the C4 diagrams (visual layer), element-linked ADRs, and optionally mirrored arc42 pages linked to elements. If the human later wants Archyl as canonical for the **C4 model only** (edit in Archyl, `GET /dsl/export` committed back as `archyl.yaml`), that is workable but needs a reverse-sync job that is not provided, and merge conflicts between UI edits and file edits are undocumented.

## Free-plan limits (Developer plan)

The pricing page's per-plan numbers render client-side, so the Markdown twin of [archyl.com/pricing](https://www.archyl.com/pricing/index.md) shows only headings. The numbers below are the plan constants in Archyl's own public web bundle (`https://www.archyl.com/assets/index-i7-YuySe.js`, read 2026-10-08), whose field names match the OpenAPI `PlanLimits` schema. Treat them as first-party but not as a published contract. The OpenAPI `PlanType` marks Developer as "Free tier," Business as "20 EUR/month or 180 EUR/year" and Scale as "40 EUR/month or 360 EUR/year" (pricing is "per editor/month").

| Limit | Developer (free) | Business | Scale |
|---|---|---|---|
| Projects | 10000 (bundle shows `maxProjects:1e4`; feature text says "Unlimited projects") | same | same |
| Model objects (systems, containers, components, code elements, relationships per Concepts page) | **100** | 1,000 | 10,000 in `PlanLimits`, "Unlimited" in the feature text (contradiction) |
| ADRs, diagram overlays | Unlimited | Unlimited | Unlimited |
| Documentation pages | No limit field exists in `PlanLimits` | same | same |
| AI discovery | **3** "projects AI discovery" | 10 | 50 |
| AI flows / month | 5 | 20 | 100 |
| AI chat queries | Off (0) | 200 | 2,000 |
| Managed agent runs | Off | Off | On |
| Webhooks | **Off** | 5 | Unlimited |
| Shared links | **Off** | On | On |
| Team invitations | Off | 3 | Unlimited |
| Teams | 1 | 1 | Unlimited |
| Static export, drift score | Yes | Yes | Yes |
| Architecture as Code (DSL) | The pricing headings list "Architecture as Code (DSL)" as a compared row, but per-plan values are not in the page text I could read. The Developer feature list does not mention it. **Unverified for free**; the ADR note's trial and the Action docs do not state a plan gate. | | |

Points that matter:

- **Model objects: 100 on the free plan.** Chaptour at C4 level 1 to 3 is well under that. A Container-and-Component model of a small .NET/Azure system fits comfortably, but Level 4 code elements and relationships also count (**inference** from the Concepts page: "model objects" are systems, containers, components, code elements, relationships; whether *relationships* count is unconfirmed against the quota check).
- **Free "AI discovery" is a count of 3 *projects*, not 3 runs** in `PlanLimits` (field description: "Number of projects with AI discovery"). The ADR note observed "3 Import using AI runs per month" in the UI and used 2. These two descriptions differ, and the monthly reset is real (`aiDiscoveryResetsAt` in the OpenAPI `UsageResponse`). The usage check should be re-read in the UI before relying on either reading.
- **Bring your own AI provider.** The pricing page states that connecting your own OpenAI, Anthropic, Google Gemini, AWS Bedrock or OpenAI-compatible endpoint means "the monthly limits above (AI discovery, chat queries, managed agent runs) no longer apply." Whether this is available on the free plan is not stated. If it is, it removes the AI-quota constraint for Documentation/ADR Discovery. It also means Chaptour would supply its own provider key, another stored secret for ADR-0013 to account for.
- **No webhooks on free** (outbound events from Archyl). Combined with "Coming Soon" repo webhooks, any automation must be CI-initiated. Shared links are also off, so a public diagram link or embed needs Business.
- Rate limits for the REST API are not stated on the API overview or authentication pages.

## API and CI options

- **REST.** Base `https://api.archyl.com/api/v1`, auth via `X-API-Key` (per-user, read-only or read-write, see ADR note). Relevant endpoints in OpenAPI: `/projects/{id}/dsl/import|validate|export|import-archive|validate-archive`, `/projects/{id}/export`, `/projects/{projectId}/documentation` (GET list, POST create), `/documentation/{id}`, `/documentation/{id}/links`, `/documentation/by-element`, `/doc-folders`, `/adrs`, `/adrs/discovery`, `/documentation/discovery`. `/dsl/ingest` is documented only in prose (not in OpenAPI).
- **MCP server** at `https://api.archyl.com/mcp` (Streamable HTTP; API key via `X-API-Key` or OAuth 2.1). 11 documentation tools (create, update, move, delete documentation and folders), 6 ADR tools including `link_adr_to_element`, and `import_dsl` (Archyl YAML, Structurizr, LikeC4, IcePanel, Backstage). This gives a coding agent (including a Chaptour maintenance skill) a way to update pages and ADRs one at a time, a documented alternative to `/dsl/ingest` for both docs and ADRs.
- **SDKs.** Node and Python mentioned on the [SDK page](https://www.archyl.com/docs/api/sdk) (previously noted). No CLI.
- **Official Actions** (`archyl-com/actions`): `sync` (archyl.yaml to Archyl, covered in the ADR note), `release` (deploy tracking, needs only key and project), and four that need an `organization-id` and look like paid-tier features (`drift-score`, `generate-context`, `auto-cr`, `conformance-check`). **No plan gating is stated for any of them** on the [GitHub Actions page](https://www.archyl.com/docs/features/github-actions), but drift score is listed on every plan in the plan features. **Unverified:** whether any Action fails on the free plan. `generate-context` (writes `archyl.txt` for agents, optionally commits) is the only documented Archyl-to-repo direction other than manual `dsl/export`.
- **Archyl to repo.** The only documented way: call `GET /projects/{id}/dsl/export` and commit the result (CI job, needs an API key with read scope), or `generate-context`. No documented PR-creating sync.

## What this means for the map (#7)

For deciding the architecture-documentation layer:

1. **Keep arc42 in the repo** (`docs/arc42/*.md`, one file per section) and treat Archyl as an optional derived view of C4, ADRs and linked pages. This contradicts nothing in ADR-0014 and keeps `main` shippable without Archyl.
2. **Use `archyl.yaml` for the C4 model only** if Archyl is adopted, with element-linked ADRs. The decision between "model lives in `archyl.yaml`" and "model lives in Structurizr/LikeC4 and is imported" is real: Archyl YAML round-trips, the others lose views and cannot be exported back.
3. **Do not plan on generated diagram images in the repo** from Archyl. There is no documented render endpoint. If images in the repo matter (README, PR review), a repo-side renderer (Mermaid, Structurizr, LikeC4) is the only documented-capable route, and Archyl would be a second, parallel view.
4. **Decide the arc42 doc sync path early:** Discovery (AI, quota), `docs.folder`/`file` through ingest (unverified), or CI-generated inline `content` (deterministic). The first ticket to settle it should run one `workflow_dispatch` with a single `docs.records[].file` entry and read the `docsCreated` counter, as the ADR note proposes for `adrs.folder`.

## Open questions

1. Does `/dsl/ingest` fetch `docs.folder` and `docs.records[].file` (needs a connected repo), or only store the path? (Same test as `adrs.folder`, count `docsCreated`.)
2. Is Architecture as Code / `/dsl/ingest` available on the free Developer plan, and does it count against any quota?
3. Do relationships count toward the 100 model objects, and what happens at the limit (read-only, rejected import, or partial)?
4. Is "bring your own AI provider" available on the free plan, and does it remove the 3-discovery limit for Documentation Discovery?
5. Does Archyl have a deployment-view or C4 dynamic-view feature in the UI, and are Flows exportable?
6. Do `sync`, `drift-score`, `generate-context`, `auto-cr` and `conformance-check` work on the free plan?
7. Are doc pages versioned, and can a page be addressed by `filePath` on re-import (idempotent update by path vs duplicate by title)?
8. Does a LikeC4 import carry any of `view` blocks or only the model?
