# Archyl sync for `docs/adr/` (issue #2)

Research note for [thecodetinker/chaptour#2](https://github.com/thecodetinker/chaptour/issues/2), "Set up Archyl sync for docs/adr". It checks the issue's proposed `archyl.yaml` and `archyl-com/actions/sync@v1` workflow against Archyl's own docs, the action's source code, Archyl's published JSON Schema and OpenAPI spec, and GitHub's Actions documentation. This is research output only. It does not add `archyl.yaml`, a workflow, or any other file, and it does not change `docs/adr/`.

Sources are primary throughout: `archyl.com/docs` pages, the `archyl-com/actions` repo (read at tag `v1`), the DSL JSON Schema served by `api.archyl.com`, Archyl's OpenAPI document linked from its RFC 9727 API catalog, and `docs.github.com` / `github.blog/changelog`. `docs.archyl.com` does not resolve (DNS `ENOTFOUND` on 2026-09-29). Archyl's docs live under `https://www.archyl.com/docs/...`.

## The short answer to the central question

**Nothing primary says whether `adrs: folder: docs/adr` imports anything when the file is pushed through `archyl-com/actions/sync`. The action cannot make it work by itself.** The action uploads the text of `archyl.yaml` and nothing else. If ADRs appear, it is because Archyl's server fetches `docs/adr/` from a repository connected to the project. Archyl documents repository connection (OAuth) as a separate setup. It never states that the `/dsl/ingest` endpoint uses that connection to resolve `adrs.folder`. This has to be tested before it can be relied on. A cheap test and a fully documented fallback are described below.

## What the sync action actually does

The action is a single short script. `sync/src/index.js` in [`archyl-com/actions`](https://github.com/archyl-com/actions/blob/v1/sync/src/index.js) (read via `gh api repos/archyl-com/actions/contents/sync/src/index.js`):

1. Resolves `file` (default `archyl.yaml`) against `GITHUB_WORKSPACE` and fails if the file is missing or empty.
2. Reads that one file as UTF-8.
3. `POST`s `{ "content": <file text> }` to `${api-url}/api/v1/projects/${project-id}/dsl/ingest` with header `X-API-Key`.
4. Reads `response.result.import` and prints a summary built only from `*Created` counters (`systemsCreated`, …, `adrsCreated`, `docsCreated`, …). It sets outputs only for systems, containers, components and relationships, plus `summary`.

It never reads `docs/adr/`, never follows `include:`, and never uploads anything besides the YAML string. The compiled bundle that actually runs (`sync/dist/index.js`, per `runs.main` in [`sync/action.yml`](https://github.com/archyl-com/actions/blob/v1/sync/action.yml)) contains the same `readFileSync(filePath…)` and `/dsl/ingest` URL (line ~27577–27586 of the bundle). `action.yml` confirms the four inputs: `api-key` (required), `project-id` (required), `api-url` (default `https://api.archyl.com`), and `file` (default `archyl.yaml`). It also confirms `runs.using: 'node20'`.

On versions and SHAs, `gh api repos/archyl-com/actions/tags` shows three tags: `v1`, `v1.0.1` and `v1.0.0`. `v1` is an annotated tag object (`fe56bf09…`) that dereferences to commit **`338b8b818dc50bbede23e52746391653a08d82cb`**, the same commit as `v1.0.1` and the current head of `main` (committed 2026-08-15, "Fix API endpoints in generate-context, auto-cr and conformance-check"). `v1.0.0` is `a3a97007b7d1c4a51c31aed2534991f97eceb591`. The sync action itself was added in `904005c` (2026-03-11) and was not changed by the v1.0.1 commit. There is also a standalone Marketplace mirror, [`archyl-com/sync-action`](https://github.com/archyl-com/sync-action) (`v1` = `fd598ed07a7ccaf487e75b292e1f088bcdb962b4`), whose `src/index.js` is byte-identical to `archyl-com/actions/sync/src/index.js` (checked with `diff`). The issue names the monorepo path, so this note uses it.

Archyl's own [Architecture as Code](https://www.archyl.com/docs/features/export) page describes the same behaviour. It says the action "reads your `archyl.yaml`, pushes it to the Archyl API, and reports what was created or updated." It also gives a `curl` equivalent that posts only `{"content": $(cat archyl.yaml | jq -Rs .)}` to `/dsl/ingest`.

## What Archyl documents about `adrs.folder`

**Schema.** Archyl serves a JSON Schema for the DSL at `https://api.archyl.com/api/v1/dsl/schema` (fetched 2026-09-29, HTTP 200, no auth). Its relevant parts:

- `required: ["version"]`, and `version` is `enum: ["1.0"]`. So `version: "1.0"` is correct and must be a string.
- `project` is an object with `name`, `description` and `tags`, none of them schema-required.
- `adrs` has two properties:
  - `folder`, a string described as "Path to ADR folder in repo".
  - `records`, an array of objects with `title` (required), `number`, `status` (`proposed|accepted|deprecated|superseded`), `date` (format `date`), `context`, `decision`, `consequences`, `tags` and `links`.
- **ADR records have no `file` property.** `docs.records` has one ("Path to markdown file in repo") and so does `content`. You can therefore point at an individual Markdown file for docs, but not for ADRs. For ADRs, the only way to reference repo files is `folder`.

**Architecture as Code page.** [archyl.com/docs/features/export](https://www.archyl.com/docs/features/export) shows `adrs: folder: docs/adrs # optional: path to ADR folder in repo` alongside inline `records`. It does not say how the folder is read, which file formats or headings it expects, or whether it works through `/dsl/ingest`. The same page says "Only `version` is required" and, in the import-name table, that for Archyl YAML "`project.name` — required". The context is *creating a new project* from an import. When importing into an existing project, the page says (for Structurizr) the name "is ignored there, because the project already has one". So `project.name: Chaptour` is harmless and correct, but not load-bearing for sync into an existing project.

The page also describes a second, UI-driven path, "Syncing from a Repository": "Go to **Project Settings > Architecture as Code** … Click **Sync Now**. Archyl fetches the file from your repository's default branch (or the branch configured in DSL settings) and imports it." That path requires a connected repository. It says Archyl fetches "the file", not the folder. It also notes that "The `import_dsl` MCP tool and repository sync read a single file and do not resolve `!include`". That remark is about Structurizr `!include`, but it shows repository sync is file-oriented.

**Documentation & ADRs page.** [archyl.com/docs/features/documentation](https://www.archyl.com/docs/features/documentation) documents **ADR Discovery**, a separate feature: "Go to **Project Settings > ADR Discovery**. Configure the path to your ADRs (e.g., `docs/adr/`). Click **Discover ADRs**. Review and approve discovered records". The OpenAPI spec describes the matching endpoint, `POST /adrs/discovery`, as "Initiates AI-powered ADR discovery from a repository". Its request body (`StartADRDiscoveryRequest`) needs `owner`, `repository` and `projectId`, and takes an optional `branch`, `folderPath`, `provider` and `accessToken` ("empty for public repos"). It is asynchronous (`202`, poll `GET /adrs/discovery/{jobId}`). This is the only documented mechanism that reads Markdown ADRs out of a folder. It is AI extraction with a human review/approve step, not a deterministic sync.

**Git integration page.** [archyl.com/docs/git-integration/overview](https://www.archyl.com/docs/git-integration/overview) says repository connection is set up under Project Settings > Repository > "Connect Repository", with GitHub OAuth. It lists "Webhooks (Coming Soon)" for automatic sync on push. So no push-triggered server-side sync exists today. Anything automatic has to come from CI.

**OpenAPI.** The spec at `https://api.archyl.com/docs/openapi.json` (linked as `service-desc` from `https://www.archyl.com/.well-known/api-catalog`; Swagger 2.0, `basePath: /api/v1`, 178 paths) documents `/projects/{id}/dsl/import`, `/dsl/validate`, `/dsl/export` and the archive variants. It does **not** list `/projects/{id}/dsl/ingest`, the endpoint the action calls. That endpoint is documented only in prose on the Architecture as Code page. `ImportDSLRequest` is `{ content (required), format: "archyl"|"structurizr"|"likec4" }`, which contains no field for attached files. `ADRResponse` has a `filePath` field, which suggests ADRs *can* be tied to a repo file, most likely by Discovery or folder import. That is an inference; I could not confirm it.

**Conclusion.** There are two possibilities, and I could not rule out either from primary sources:

- (a) The server, on ingest, sees `adrs.folder`, fetches `docs/adr/*` from the project's **connected repository**, and parses those files. This would require the human to connect the GitHub repo in Archyl (OAuth) in addition to the API key.
- (b) Via `/dsl/ingest`, `folder` is stored as metadata or silently ignored, and only inline `records` create ADRs.

Archyl's docs never say that `/dsl/ingest` reads repository files for anything. Every documented folder-reading feature (ADR Discovery, Documentation Discovery, UI "Sync Now") goes through a connected repository. So if (a) is true, a repo connection is almost certainly a prerequisite. This is inference, **unverified**.

I found no Archyl CLI and no other official action that uploads folder contents. The `archyl-com` GitHub org contains only `actions`, the per-action Marketplace mirrors (`sync-action`, `drift-score`, `release-action`, `conformance-check`, `generate-context`, `auto-cr`) and `agent-skills` (`gh repo list archyl-com`). The ADR reference in `agent-skills` (`plugins/archyl-developer/skills/archyl-developer/references/documentation/adrs.md`) only describes creating ADRs one at a time through MCP tools (`create_adr` with `title/status/context/decision/consequences`). Archyl's [API authentication](https://www.archyl.com/docs/api/authentication) page and the [SDK page](https://www.archyl.com/docs/api/sdk) mention Node/Python SDKs, not a CLI.

## ADR file format

Nothing primary documents the Markdown format Archyl expects from `adrs.folder`, whether it wants front-matter, or whether it needs particular headings. Archyl's ADR model is title, status, context, decision, consequences, plus optional number, date and tags (from the schema and from `CreateADRRequest` in OpenAPI). Chaptour's ADRs (e.g. `docs/adr/0003-archive-not-delete.md`) are an H1 plus one paragraph, with no `Status:`/`Date:` lines and no Context/Decision/Consequences headings. If folder import is a structured parser (MADR/Nygard-style), it may pick up only the title, or it may put the whole paragraph into one field. If it reuses the "AI-powered" discovery pipeline, it may split the paragraph by meaning. Neither is documented. Also, chaptour ADRs have no status, so Archyl will apply its default status. The `create_adr` reference in `archyl-com/agent-skills` gives that default as `"proposed"`, which would mislabel all 15 accepted decisions. That default comes from the MCP tool; I have not confirmed it applies to DSL ingest.

## Idempotency, updates, deletes

- **Re-runs.** The Architecture as Code page says, for both "Sync Now" and "Import into Existing Projects": "Elements that already exist are updated; new elements are created." It does not say that for `/dsl/ingest` specifically, and it does not say what key identifies an existing ADR (title? number? file path?). The ingest response documented on that page contains only `*Created` counters (`"adrsCreated": 0`, …), with no `updated` or `deleted` counters, and the action prints only "Created: …" or "No new elements created". A second run should show `No new elements created` if matching works. That is the observable test for duplicates.
- **Deletes and renames.** Nothing in Archyl's docs says ingest removes elements, ADRs included, that are missing from the file. The docs describe only "updated" and "created". Assume **deleting or renaming an ADR file will not remove the old ADR in Archyl** (unverified). If ADRs are matched by title, retitling an ADR's H1 would probably create a duplicate. ADR-0003's archive-not-delete ethos aside, chaptour ADRs are append-mostly, so this is a small risk for now. A later "superseded" change, though, would need to show up as a status change, which chaptour's format does not currently carry.

## Auth model

API keys are **per user**, not per project: "Go to **Profile > API Keys**". Permissions are "Read-only or Read-Write", and "Write" means "Create, update, and delete projects, elements, and documentation" ([archyl.com/docs/api/authentication](https://www.archyl.com/docs/api/authentication)). Expiration is optional: "Never", 30 days, 90 days, or custom. Archyl's [GitHub Actions page](https://www.archyl.com/docs/features/github-actions) and `sync/action.yml` both require a key "with write scope". So the `ARCHYL_API_KEY` secret is a standing, user-wide, read-write credential with no project scoping. I found no OIDC or workload-identity option in Archyl's API docs. The issue's note is accurate: this is Archyl's model. The key is broader than "a key for this project", though, so give it an expiry (the 90-day option) and name it for this purpose.

The same GitHub Actions page recommends storing the project UUID as an Actions **variable** (`vars.ARCHYL_PROJECT_ID`), with only the key as a secret. That fits better than hard-coding the UUID in the workflow. It is not a secret, but a variable keeps it out of the file.

## Chaptour repo facts and ADR tensions

- **No `.github/` directory exists** (`ls -a` at repo root: `.git`, `.gitignore`, `AGENTS.md`, `CONTEXT.md`, `docs`). There is no "CI pipeline" to add a step to. The sync would be the repo's **first** workflow file, so it would also be the first thing to exercise ADR-0009 (GitHub Actions for CI/CD).
- **The default branch is `master`** (`gh repo view --json defaultBranchRef` → `master`), not `main` as the issue and all Archyl examples use. A copy-pasted `branches: [main]` would never fire. The repo is **public**, which matters for the trigger choice below.
- **ADR-0013 (OIDC, no stored secrets).** This is a direct tension. ADR-0013 rejects long-lived stored credentials "a real liability if it ever leaks", and `ARCHYL_API_KEY` is exactly that kind of credential, user-wide and read-write. The ADR's wording scopes it to *Azure* auth ("GitHub Actions authenticates to Azure via OIDC"), so this is not a literal violation. It is worth a line in the eventual PR, or a short ADR, that records the exception and its mitigations: key expiry, a dedicated key, and the secret exposed only to one job and one step.
- **ADR-0014 (trunk-based, every commit on main shippable)** makes `on: push` to the trunk the natural trigger, and there are no long-lived branches to sync from. The sync has no bearing on shippability. It should not be made a required status check, so an Archyl outage cannot block the trunk.
- **ADR-0015 (automated dependency scanning).** GitHub says "Pinning an action to a full-length commit SHA is currently the only way to use an action as an immutable release" ([Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)). A third-party action that receives a write-scoped key is exactly the case pinning is for, because a moved `v1` tag could exfiltrate the key. Pinning by SHA does stop automatic pickup of fixes, so pair it with Dependabot's `github-actions` ecosystem, which "checks for new versions of your actions" weekly and raises PRs ([Dependabot for actions](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/auto-update-actions)). ADR-0015 already commits to Dependabot, but there is no `.github/dependabot.yml` yet. I did not confirm from that page that Dependabot rewrites SHA pins, as opposed to tag pins. GitHub's docs are widely understood to support it when a `# vX.Y.Z` comment follows the SHA, but I'm flagging it as not verified here.
- **Node 20.** `sync/action.yml` declares `using: 'node20'`. GitHub switched runners to Node 24 by default on 2026-06-16, and Node 20 "is no longer available" as of 2026-09-23 ([deprecation notice](https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/), [removal notice](https://github.blog/changelog/2026-09-23-node-20-is-no-longer-available-in-github-actions/)). The removal notice says runners "now use Node 24 for JavaScript actions" and asks maintainers to update `runs.using`. It does not state explicitly that a `node20` action is force-run on Node 24 rather than rejected. The action's code is plain CommonJS on `@actions/core` and `@actions/http-client`, so it will very likely run fine on Node 24, but this is **unverified until the first run**. It is also a sign that the action is lightly maintained: the sync code is unchanged since March 2026. The `curl` form Archyl documents is a zero-dependency fallback if the action breaks.

## GitHub Actions practice for this workflow

- **`permissions: contents: read`** at workflow level. The job only checks out and reads a file, and GitHub recommends setting `GITHUB_TOKEN` to "read access only for repository contents" by default ([Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)).
- **Pin both actions by SHA.** `actions/checkout` latest is `v7.0.1` = `3d3c42e5aac5ba805825da76410c181273ba90b1` (`gh release view --repo actions/checkout`; its `action.yml` uses `node24`). Use `persist-credentials: false`, since nothing pushes.
- **`paths:` filter** on `archyl.yaml`, `docs/adr/**`, and the workflow file itself, so edits to the workflow get exercised.
- **`workflow_dispatch`** for the first verification run, and for re-syncing after Archyl-side changes without a dummy commit.
- **No `pull_request` trigger.** The repo is public, fork PRs do not get secrets, and syncing unmerged ADRs would misrepresent decisions anyway.
- **`concurrency`**, one sync at a time and not cancelled mid-flight, so two quick pushes cannot race.

## Recommended implementation

This has two stages, because the central question can only be settled empirically.

**Stage 1: try the issue's approach and read the result.** Proposed `archyl.yaml`. Verified against the schema: the `version` value, the `project.name` type, and that `adrs.folder` is a string. **Unverified:** whether `folder` does anything through ingest.

```yaml
# yaml-language-server: $schema=https://api.archyl.com/api/v1/dsl/schema
version: "1.0"
project:
  name: Chaptour
adrs:
  folder: docs/adr   # UNVERIFIED: whether /dsl/ingest resolves this (may need a connected repo)
```

Proposed `.github/workflows/archyl-sync.yml`. The SHAs were verified on 2026-09-29. Node 24 compatibility of the `node20` action is **unverified**.

```yaml
name: Archyl sync

on:
  push:
    branches: [master]          # issue says main; repo default is master
    paths:
      - archyl.yaml
      - docs/adr/**
      - .github/workflows/archyl-sync.yml
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: archyl-sync
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - id: sync
        uses: archyl-com/actions/sync@338b8b818dc50bbede23e52746391653a08d82cb # v1 (= v1.0.1)
        with:
          api-key: ${{ secrets.ARCHYL_API_KEY }}
          project-id: ${{ vars.ARCHYL_PROJECT_ID }}
      - run: echo "$SUMMARY" >> "$GITHUB_STEP_SUMMARY"
        env:
          SUMMARY: ${{ steps.sync.outputs.summary }}
```

To verify, trigger with `workflow_dispatch` and read the log line. `Created: 15 ADRs` means folder resolution works, so check the Decisions tab for title, status and body mapping. `No new elements created` on the first run means the folder was ignored. A second run producing `No new elements created` again means re-sync does not duplicate.

**Stage 2 (only if Stage 1 imports 0 ADRs): generate inline `records` at CI time.** This path is fully schema-documented and deterministic. A small script step parses `docs/adr/NNNN-*.md` into `adrs.records` (`number` from the filename, `title` from the H1, `decision` from the paragraph, `status: accepted`), writes the result to a temp file, and passes it to the action via `file:`. Nothing is duplicated in git. This follows the spirit of the issue ("point at the folder, don't duplicate") while not depending on undocumented server behaviour. The ADR matching key for updates and the delete behaviour are still unverified either way. The other alternative is ADR Discovery (UI, or `POST /adrs/discovery`). It is documented as AI-powered and needs a human to review and approve, so it suits a one-off bootstrap better than a push-triggered sync.

Optionally, add `.github/dependabot.yml` with `package-ecosystem: github-actions` so the SHA pins stay current, as ADR-0015 already intends.

## Open questions / blockers for the human

1. **Does ingest resolve `adrs.folder`?** Only a real run (Stage 1) or Archyl support can answer this. If it does, find out whether it needs the GitHub repo **connected in Archyl** (Project Settings > Repository, OAuth). That would be a human step the issue does not list.
2. **How does Archyl map a chaptour ADR (H1 plus one paragraph, no status) into title/context/decision/consequences, and what default status does it assign?** Check the first imported ADR in the UI.
3. **What key does Archyl use to match an existing ADR on re-sync, and are removed or renamed ADRs ever deleted?** Undocumented. Test with a second run, and consider asking Archyl.
4. **API key scope.** Keys are user-wide read-write. Create a dedicated key with a 90-day expiry, and decide whether ADR-0013 needs an explicit note or ADR recording this exception.
5. **Store the project UUID as the Actions variable `ARCHYL_PROJECT_ID`** (Archyl's own convention), in addition to the `ARCHYL_API_KEY` secret.
6. **Run the Node 24 check** on the first run. If the `node20` action misbehaves, switch to the documented `curl` call.

## Mismatches between issue #2 and reality

- The issue says `main`. The repo's default branch is **`master`**.
- The issue says "Add … to the CI pipeline". **There is no CI pipeline**, no `.github/` at all. This would be the first workflow.
- The issue says "`adrs` section can point at an existing ADR folder rather than duplicating records inline". The schema allows `folder`, but **Archyl does not document that it works through `/dsl/ingest`**, and the sync action demonstrably uploads only the YAML text.
- The issue says "evaluated in `docs/research/architecture-documentation-options.md`". **That file never mentions Archyl.** It evaluates arc42, C4, Google design docs and Nygard ADRs, and recommends C4 diagrams informally (e.g. Mermaid) rather than a platform. The Archyl decision has no research note or ADR behind it yet.
- The issue calls this "the official `archyl-com/actions/sync@v1`". That is correct, but `@v1` is a mutable tag. Given ADR-0015 and the write-scoped key, pin it to `338b8b818dc50bbede23e52746391653a08d82cb`.
- Step 2 says to generate a key "with write scope". Archyl's keys are per-user and cover all of that user's projects, which is broader than the issue implies.
