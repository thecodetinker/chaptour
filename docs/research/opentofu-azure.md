# OpenTofu on Azure facts (issue #10)

Research note for [thecodetinker/chaptour#10](https://github.com/thecodetinker/chaptour/issues/10), a child of map issue #7. It gathers facts for a later IaC decision. It does not decide anything, create Azure resources, or add any code. Read on 2026-10-08.

Sources are primary: `opentofu.org` docs, GitHub repos (`opentofu/opentofu`, `hashicorp/terraform-provider-azurerm`, `Azure/terraform-provider-azapi`, `Azure/azure-dev`), `learn.microsoft.com`, `docs.github.com`, `aspire.dev` and `cncf.io`. `learn.microsoft.com/dotnet/aspire/...` pages 301-redirect to `aspire.dev`, so the Aspire docs are cited there.

Context from the ADRs: Aspire is local-dev orchestration only (0006); production is Azure App Service with a staging slot and swap (0008); merge to `main` deploys, smoke-checks and swaps with no gate (0010); EF migrations run as a pipeline step against Azure SQL (0011); GitHub Actions authenticates to Azure with OIDC and no stored secret (0013).

## Short answers

1. **State and auth.** Yes. The `azurerm` backend supports GitHub Actions OIDC and Entra ID auth to the blob container, so no storage key or client secret is needed. Locking uses native blob capabilities. Server-side encryption is always on, and OpenTofu also has its own client-side state encryption with an Azure Key Vault key provider.
2. **Resource coverage.** `azurerm` has first-class resources for everything required, including `.NET 10` on Linux App Service, slots, and an explicit slot-swap resource. `azapi` is a thin ARM layer for anything `azurerm` lags on.
3. **Aspire and azd.** There is no documented OpenTofu integration in Aspire. `azd` supports Terraform only in beta and hard-codes the `terraform` binary. Aspire now recommends `aspire deploy` over `azd`. Because ADR-0006 limits Aspire to local dev, the two coexist trivially as long as the AppHost never provisions or publishes Azure resources.
4. **Versions and governance.** OpenTofu 1.13.1 is current (2026-10-01). It is MPL-2.0, Linux Foundation hosted, and a CNCF Sandbox project. Terraform is under BUSL 1.1 with IBM as licensor.

## 1. State backend, OIDC, locking, encryption

### Backend configuration and OIDC

Source: [opentofu.org azurerm backend](https://opentofu.org/docs/language/settings/backends/azurerm/).

- The backend documents these OIDC arguments: `use_oidc`, `oidc_token`, `oidc_token_file_path`, `oidc_request_url`, `oidc_request_token`, plus `client_id`, `tenant_id` and `subscription_id`.
- On GitHub Actions the backend picks up `ACTIONS_ID_TOKEN_REQUEST_URL` and `ACTIONS_ID_TOKEN_REQUEST_TOKEN` automatically. The workflow needs `permissions: id-token: write` ([GitHub docs, OIDC in Azure](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-azure)).
- Its documented OIDC example uses `resource_group_name`, `storage_account_name`, `container_name`, `key`, `subscription_id`, `tenant_id`, `client_id` and `use_oidc = true`. It stores no secret.
- `use_azuread_auth = true` makes the backend talk to Blob Storage with Entra ID instead of the storage account key. It needs the **Storage Blob Data Contributor** role on the account or container. The docs warn that `Owner` does not grant data access through Entra ID. `resource_group_name` is then not required.
- The two settings are separate: `use_oidc` is how the identity logs in, `use_azuread_auth` is how it reaches blob data. The OIDC example in the docs does not itself show `use_azuread_auth`, so the combination `use_oidc = true` plus `use_azuread_auth = true` (no account key anywhere) is the expected setup for ADR-0013 but this note did not find a single doc example of it. Verify when specifying.
- The `azurerm` provider (not the backend) also supports OIDC, using `ARM_USE_OIDC`, `ARM_CLIENT_ID`, `ARM_TENANT_ID` and `ARM_SUBSCRIPTION_ID` ([provider OIDC guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/service_principal_oidc.html.markdown)). Added in provider 3.7.0.
- Federated credential: the app registration needs a federated credential whose subject matches the workflow. The provider guide's example is `repo:OWNER/REPO:refs:refs/heads/main` (copied from the guide as written; GitHub's own subject formats for environment, branch and PR are in its OIDC reference, which the page above only points to). GitHub says to define at least one condition, and notes repositories created after 2026-07-15 get an immutable default `sub` claim containing owner and repository IDs, so check which format applies before creating the credential. Recommended audience: `api://AzureADTokenExchange`.
- Bootstrap: the state storage account must exist before the first `tofu init`. Microsoft's guide creates it with `az` CLI or a separate Terraform config ([Store Terraform state in Azure Storage](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)), and its example uses an access key, which it says to replace with a more secure option in production. That one-off creation is a human step and falls outside ADR-0013's pipeline scope. It also advises restricting network access to the account.

### Locking

- The OpenTofu docs say only that the backend "supports state locking and consistency checking with Azure Blob Storage native capabilities". They give no mechanism detail.
- Microsoft Learn: "Azure Storage blobs are automatically locked before any operation that writes state", and the lock is visible on the blob in the portal (same Learn page, section 4).
- Optional `snapshot` argument (default `false`) snapshots the state blob before use. OpenTofu's docs instead recommend enabling versioning and soft delete on the container.

### Encryption

Two independent layers:

1. **Storage-side.** Azure encrypts blob data at rest automatically ([Learn](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage), section 5). The backend also accepts `customer_provided_key` (base64 AES-256) or `encryption_scope`, mutually exclusive. The docs warn that a blob encrypted with a customer-provided key cannot be recovered if the key is lost.
2. **OpenTofu state and plan encryption** ([docs](https://opentofu.org/docs/language/state/encryption/)). Configured in `terraform { encryption { ... } }`. Key providers: PBKDF2 (passphrase, minimum 16 characters), AWS KMS, GCP KMS, **Azure Vault**, OpenBao, plus external. Methods: AES-GCM, external, unencrypted (for migration). Caveats from the page:
   - Once enabled, OpenTofu refuses to read plain-text state, so migrating needs a fallback block using the `unencrypted` method.
   - Lost keys mean unrecoverable state.
   - Renaming a key provider or method breaks existing encrypted data unless `encrypted_metadata_alias` is used.
   - The `azure_vault` provider takes `vault_uri`, `vault_key_name`, `key_length`, `symmetric` and `symmetric_key_size`. It uses RSA-OAEP-256 for asymmetric keys, and "exclusively uses Entra ID for authentication" ([azure_vault docs](https://github.com/opentofu/opentofu/blob/main/website/docs/language/state/encryption.mdx)). The doc does not say whether that includes the GitHub OIDC flow; unverified.
   - The page does not state which OpenTofu version introduced the feature; it is documented for the current 1.13.x line.
   - Implication: a Key Vault-backed key provider makes state decryption depend on Key Vault, which itself would be managed by this same state, so the bootstrap ordering needs care.

State is plain text and may hold secrets unless encrypted. For example, `azurerm_mssql_server` notes credentials land in raw state unless the write-only password attribute is used ([mssql_server docs](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/r/mssql_server.html.markdown)).

## 2. Resource coverage

Provider docs read from `hashicorp/terraform-provider-azurerm` `main` (v5.9.0 at time of reading) and `Azure/terraform-provider-azapi` `main`.

| Need | azurerm resource | Facts from the provider docs |
|---|---|---|
| App Service (Linux) | `azurerm_service_plan`, `azurerm_linux_web_app` | `site_config.application_stack.dotnet_version` accepts `10.0` (also 3.1 to 9.0). Supports `identity` (system/user assigned), `app_settings`, `sticky_settings` (settings that do not swap), `key_vault_reference_identity_id`, `virtual_network_subnet_id`. |
| Deployment slot | `azurerm_linux_web_app_slot` | Requires `app_service_id` and `site_config`. `site_config.auto_swap_slot_name` exists. |
| Slot swap | `azurerm_web_app_active_slot` | Takes `slot_id` and swaps it with Production. `overwrite_network_config` defaults to `true`. Destroying it does **not** swap back. |
| Azure SQL | `azurerm_mssql_server`, `azurerm_mssql_database` | Server supports `azuread_administrator` with `azuread_authentication_only`, `identity`, TLS 1.2 only. Database supports serverless SKUs (e.g. `GP_S_Gen5_1`) with `auto_pause_delay_in_minutes` and `min_capacity`, retention policies, TDE on by default, and docs recommend `prevent_destroy`. The docs mention no free-offer attributes. |
| Blob Storage | `azurerm_storage_account`, `azurerm_storage_container` | `shared_access_key_enabled`, `default_to_oauth_authentication`, `allow_nested_items_to_be_public` (default false), `blob_properties` for versioning and delete retention. |
| Application Insights | `azurerm_application_insights` | `workspace_id` links a Log Analytics workspace; the docs say it cannot be removed once set and that Microsoft may auto-migrate classic resources, causing diffs. `application_type` is case-sensitive. |
| Key Vault | `azurerm_key_vault` | `rbac_authorization_enabled` is required. Inline `access_policy` and `azurerm_key_vault_access_policy` conflict. `soft_delete_retention_days` is set once only. Purge protection cannot be disabled once on. Switching authorization model needs Owner or User Access Administrator. |

**azapi.** The provider is "a very thin layer on top of the Azure ARM REST APIs" for resources "not yet or may never be supported in the AzureRM provider" ([README](https://github.com/Azure/terraform-provider-azapi)). Its `docs/resources` directory lists generated per-ARM-type pages plus `resource`, `update_resource`, `resource_action` and `data_plane_resource`. `azapi_resource_action` runs an action against an existing resource; its default method is POST, so a POST action such as a swap is expressible. This note did not find a documented slot-swap example for it and did not test it (inference). Because `azurerm` already has `azurerm_web_app_active_slot`, azapi is only a fallback. None of the five needed resource areas appear to require azapi.

### Chaptour-relevant tension (flagged, not decided)

ADR-0010 swaps staging to production inside the deploy pipeline on every merge. `azurerm_web_app_active_slot` models swap as a declarative resource and does not swap back on destroy. Putting the per-deploy swap in IaC would mean a `tofu apply` per deployment with a changing `slot_id` input, and a second swap source of truth next to a CI swap step. The slot, plan and app resources themselves fit IaC cleanly. Where the swap lives is a design question for the decision, not answered by the provider docs.

## 3. OpenTofu alongside Aspire and azd

### What Aspire does with Azure

- Per [Local Azure provisioning](https://aspire.dev/integrations/cloud/azure/local-provisioning/): with Azure integrations (`AddAzure*`) in the AppHost, `aspire run` creates **real Azure resources** in the developer's subscription from generated Bicep, using the developer's credentials, and they incur charges. ADR-0006 as written (local SQL Server and Azurite containers) does not trigger this. It would if the AppHost used `AddAzureSqlServer` and similar without `RunAsContainer`-style local emulation. This note did not verify the exact local-emulation APIs.
- `aspire publish` emits target-specific artifacts for external tooling. `aspire deploy` generates output, resolves parameters and applies it directly. App Service, Container Apps and AKS are listed targets ([deployment overview](https://aspire.dev/deployment/deploy-with-aspire/)).
- The Aspire App Service integration is marked **Preview**. It deploys project resources and Dockerfile-backed containers, supports named deployment slots, and generates an App Service Plan, a **Container Registry** and a managed identity via Bicep ([get started](https://aspire.dev/integrations/cloud/azure/azure-app-service/azure-app-service-get-started/)). The container registry sits awkwardly with ADR-0008's reason for choosing App Service ("no Dockerfile or container registry to maintain").
- Customisation hooks: `ConfigureInfrastructure`, `AddBicepTemplate`, and `AsExisting` / `RunAsExisting` / `PublishAsExisting` for pre-provisioned resources ([customise resources](https://aspire.dev/integrations/cloud/azure/customize-resources/)). `PublishAsExisting` is the documented way to point Aspire at Azure resources created elsewhere, which is the natural seam for resources owned by OpenTofu. The page makes no mention of Terraform or OpenTofu.

### What azd does

- `azd` supports Bicep (default) and Terraform as IaC providers via `infra.provider: terraform` in `azure.yaml`, but **Terraform support is "still in beta"** ([Learn](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/use-terraform-for-azd)).
- The `azd` Terraform tool wrapper runs the binary named `terraform` (`ToolInPath("terraform")` in [`cli/azd/pkg/tools/terraform/terraform.go`](https://github.com/Azure/azure-dev/blob/main/cli/azd/pkg/tools/terraform/terraform.go)). A repo search for OpenTofu issues in `Azure/azure-dev` returned nothing. So an OpenTofu-only machine is not a supported `azd` configuration; it is possible only by putting a `tofu` shim named `terraform` on PATH, which is unsupported (inference).
- The same Learn page says the `azurerm` provider authenticates through Azure CLI by default, so `azd up` needs `az login` too. It also says remote state must be set up **before** `azd pipeline config`, via `provider.conf.json` and `RS_*` env vars.
- Aspire's own position: `azd` "is still supported with Aspire for existing workflows, but it is no longer the recommended default deployment path"; `aspire deploy` is recommended. The deployment manifest behind the azd integration is "deprecated and no longer being evolved" ([Aspire and azd](https://aspire.dev/deployment/azure/azure-developer-cli/)). It notes `WithAzdResourceNaming()` for migrating naming from `azd` to `aspire deploy`.

### Is there a supported or common pattern?

- No primary source documents OpenTofu working with Aspire or `azd`. The Aspire pages checked say nothing on Terraform or OpenTofu, and `azd` documents Terraform (beta) but not OpenTofu.
- The documented seams are: (a) `azd` hooks (`preprovision`, `postprovision`, etc.) in `azure.yaml` can run arbitrary scripts, so a script could call `tofu` ([hooks docs](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-extensibility)); outputs are exchanged via `azd env set`; and (b) Aspire's `AsExisting` family for consuming pre-created resources.
- Neither is a documented OpenTofu pattern. Treat any such combination as bespoke.

### Pitfalls

1. Dual ownership. If Aspire (via `aspire deploy` or `azd`) and OpenTofu both create the same Azure resources, there are two sources of truth and no shared state. Pick one owner per resource.
2. ADR-0006 scopes Aspire to local dev. The coexistence question disappears only if the AppHost never provisions or publishes to Azure. The Azure integrations can silently start provisioning real resources on `aspire run`, so this needs a guard in the solution layout decision.
3. The App Service Aspire integration is Preview and generates a container registry, which conflicts with ADR-0008's rationale; it should not be adopted for production as a side effect.
4. `azd` + Terraform is beta and expects the `terraform` binary plus an `az login`, which clashes with an OIDC-only, OpenTofu-only pipeline (ADR-0013).
5. Slot swap ownership (see section 2) and EF migrations as a pipeline step (ADR-0011) both sit between infrastructure apply and app deploy, so the pipeline order needs defining.

## 4. Versions, stability, licence and governance

| Item | Fact | Source |
|---|---|---|
| OpenTofu latest | v1.13.1 (2026-10-01), stable. v1.13.0 released 2026-09-30. v1.12.7 patched 2026-10-01. | [GitHub releases](https://github.com/opentofu/opentofu/releases) |
| 1.13 notes | Windows ARM64 officially supported. Final release series supporting 32-bit CPUs. Experimental `-lint` and symbol libraries. AWS KMS, GCP KMS and OpenBao encryption providers gained extra parameters. | releases page |
| azurerm provider | v5.9.0 (2026-10-08). Prior: 5.8.0 (2026-10-01), 5.7.0 (2026-09-24). Roughly weekly releases. | [GitHub releases](https://github.com/hashicorp/terraform-provider-azurerm/releases) |
| azapi provider | v2.13.0 (2026-09-28). Prior: 2.12.0 (2026-08-10). | [GitHub releases](https://github.com/Azure/terraform-provider-azapi/releases) |
| Provider source | Default registry `registry.opentofu.org`. `required_providers` addresses like `hashicorp/azurerm`. Commit `.terraform.lock.hcl`. | [docs](https://opentofu.org/docs/language/providers/requirements/) |
| OpenTofu licence | MPL-2.0 (GitHub repo metadata: `MPL-2.0`). | [repo](https://github.com/opentofu/opentofu), [manifesto](https://opentofu.org/manifesto/) |
| Governance | Community-driven, stewarded by the Linux Foundation. Accepted to the CNCF at **Sandbox** maturity on 2025-04-23. | manifesto, [CNCF](https://www.cncf.io/projects/opentofu/) |
| Terraform licence | Business Source License 1.1, licensor IBM Corporation, applies to 1.6.0 and later, converting to MPL 2.0 four years after each publication. | [LICENSE](https://github.com/hashicorp/terraform/blob/main/LICENSE) |

Stability notes:

- The OpenTofu repo was created 2023-08-16. Sandbox is the earliest CNCF tier, which CNCF describes as not yet extensively production tested.
- The `azurerm` provider moved to a 5.x major, and the backend docs mark `endpoint` and `msi_endpoint` as deprecated, so pin provider versions and read the 5.0 upgrade guide before adopting. The upgrade guide was not read for this note.
- Whether the HashiCorp-owned `azurerm` provider's licence and registry availability under OpenTofu are unaffected was not separately checked; the OpenTofu registry serves `hashicorp/azurerm` by default per the requirements doc, but this note did not independently confirm the registry listing for the latest version.
- The issue also asked about compatibility with Terraform. The migration page on `opentofu.org` returned 404, so no primary statement about Terraform-version compatibility was captured beyond the manifesto's backward-compatibility pledge.

## Factory vs Chaptour-specific

Per map #7, each fact is marked for reuse.

**Factory (reusable on another project):**
- State backend pattern: `azurerm` backend with `use_oidc` and `use_azuread_auth`, Storage Blob Data Contributor role, one-off bootstrap of the state account, versioning and soft delete on the container, optional OpenTofu state encryption.
- GitHub OIDC federation setup (ADR-0013) and the workflow permission `id-token: write`.
- Coverage findings for the `azurerm` resources and the azapi fallback.
- The rule that Aspire and OpenTofu must not own the same resource, and that `azd` plus OpenTofu is unsupported.
- Version and governance facts (re-verify per project).

**Chaptour-specific:**
- The mapping to ADR-0008, 0010, 0011: Linux App Service on .NET 10 with a staging slot, Azure SQL (serverless is available), one Blob Storage account, Application Insights, Key Vault.
- Where the staging-to-production swap lives (pipeline step vs `azurerm_web_app_active_slot`).
- Keeping the AppHost free of Azure provisioning (ADR-0006) and avoiding the Preview App Service integration's container registry (ADR-0008).

## Open points not resolved by primary sources

- A documented end-to-end example of `use_oidc` plus `use_azuread_auth` together on GitHub Actions.
- Whether the `azure_vault` encryption key provider can authenticate with GitHub OIDC.
- The OpenTofu version that introduced state encryption.
- Any Aspire or `azd` statement on OpenTofu (none found).
- Whether `azapi_resource_action` successfully performs a slot swap (not tested).
