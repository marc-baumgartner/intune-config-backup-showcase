# 02 Set up in Azure DevOps

Intune Config Backup runs in Azure DevOps: the backup, restore and replicas as Azure Pipelines on an agent of your own, the history in Azure Repos, the web UI in Azure Container Apps ([how it fits together](../README.md#architecture)). No secret is stored anywhere: the pipelines sign in with workload identity federation, the web UI with its managed identity.

| Part | Name | What it is |
|---|---|---|
| App registrations | `IntuneConfigBackup.Reader`, `.Restore`, `.Web`, `.Deploy` | who reads, who writes, the sign-in to the web UI, who deploys it |
| Entra groups | `G_ICB_Readers`, `G_ICB_Restore`, `G_ICB_Admins` | who may do what in the web UI |
| Agent pool | `IntuneConfigBackupPool` | one small Ubuntu VM that runs all pipelines |
| Code | Azure Repos `intune-config-backup` | this repository, branch `main` |
| History | Azure Repos `intune-config-backup-data` | the backup: `main` for the main tenant, `backup-<name>` for [more tenants](08-tenants.md) |
| Settings | variable groups `intune-config-backup`, `intune-config-backup-secrets` | IDs and the settings of the web UI; the Teams workflow URL as a secret |
| Sign-in | service connections `icb-reader`, `icb-restore`, `icb-deploy` | one per app registration, workload identity federation |
| Approval | environment `intune-restore` | a second person approves every restore |
| Pipelines | `icb-backup`, `icb-restore`, `icb-replicate`, `icb-deploy-web` | YAML in `azure-pipelines/` |
| Web UI | Container App `ca-icb-web`, container registry | rolled out by `icb-deploy-web` on every change of `main` |

## What it costs

| | Per month |
|---|---|
| VM for the agent (B2ats v2, standard SSD) | about 7 CHF |
| Container registry (Basic) | about 5 CHF |
| Container App | nothing: it scales to zero and stays in the monthly free grant |
| Azure DevOps | nothing: one self-hosted parallel job and five Basic users are free |

Set a budget with an alert on the resource group, e.g. 20 CHF.

## Before you start

| Where | What you need |
|---|---|
| Entra ID | Application Administrator (app registrations), Privileged Role Administrator or Global Administrator (admin consent), Groups Administrator |
| Azure | a subscription and a resource group, e.g. `rg-intune-config-backup` in *Switzerland North*; Owner on the resource group, or Contributor and Role Based Access Control Administrator |
| Azure DevOps | an organization connected to the same Entra ID (*Organization settings → Microsoft Entra*), a project, and Project Collection Administrator for the agent pool and users |

Names in `code` below are the ones the pipelines and the web UI look for; keep them. The steps are in the order they depend on each other.

## 1. App registrations

*Entra admin center → App registrations → New registration*, each **single tenant**. Copy the *Application (client) ID* of each from *Overview*; the *Directory (tenant) ID* is the same for all.

### `IntuneConfigBackup.Reader`: reads Intune

The identity of the backup, of every plan before a restore and of the web UI's check. Read permissions only.

1. Name `IntuneConfigBackup.Reader`, no redirect URI → *Register*.
2. *API permissions → Add a permission → Microsoft Graph → Application permissions*:

   | Permission | For |
   |---|---|
   | `DeviceManagementConfiguration.Read.All` | Settings Catalog, device configurations, compliance policies (also of the Settings Catalog), reusable settings, assignment filters, administrative templates, updates |
   | `DeviceManagementScripts.Read.All` | PowerShell and shell scripts, remediations, custom attributes |
   | `DeviceManagementApps.Read.All` | apps, app protection and app configuration, VPP tokens, Managed Google Play; the Intune audit log, for who changed what (Microsoft puts the audit log under this permission although it is not about apps) |
   | `DeviceManagementServiceConfig.Read.All` | Autopilot profiles, enrollment configurations (ESP, restrictions, Windows Hello for Business), Apple enrollment profiles, ADE tokens and push certificate, partners, tenant settings, terms and conditions, notification templates |
   | `DeviceManagementRBAC.Read.All` | custom Intune roles with their assignments, scope tags |
   | `Policy.Read.All` | Conditional Access policies, named locations and the Entra policies |
   | `Group.Read.All` | names of the groups in assignments (optional: without it they show as "(group not found)") |
   | `AuditLog.Read.All` | the Entra audit log, for who changed Conditional Access and named locations (optional) |
   | `DeviceManagementManagedDevices.Read.All` | device categories (optional) |
   | `Domain.Read.All`, `Directory.Read.All`, `Policy.Read.DeviceConfiguration` | Entra domains, group settings, device registration policy (optional) |
   | `DeviceManagementManagedDevices.PrivilegedOperations.All` | Activation Lock bypass codes, only if you turn that type on |

   → *Grant admin consent for <tenant>*. A type whose permission is missing is named in the run with the permission it needs; the others go on ([Permissions](07-permissions.md)).
3. *Branding & properties → Internal notes*: *Reads the Intune configuration for Intune Config Backup. Read only. Signs in from the Azure DevOps service connection icb-reader.*

### `IntuneConfigBackup.Restore`: writes to Intune

Used only by the job *apply* of `icb-restore`, after the approval. Kept apart from the reader, so the backup can never change the tenant.

1. Name `IntuneConfigBackup.Restore` → *Register*.
2. *API permissions → Microsoft Graph → Application permissions*, then *Grant admin consent*:

   | Permission | For |
   |---|---|
   | `DeviceManagementConfiguration.ReadWrite.All` | Settings Catalog, device configurations, compliance policies (also of the Settings Catalog), reusable settings, assignment filters, administrative templates, updates |
   | `DeviceManagementScripts.ReadWrite.All` | scripts, shell scripts, remediations, custom attributes |
   | `DeviceManagementServiceConfig.ReadWrite.All` | Autopilot profiles, enrollment configurations, Apple enrollment profiles, terms and conditions, notification templates, tenant settings |
   | `DeviceManagementApps.ReadWrite.All` | apps and their assignments, app protection, app configuration |
   | `DeviceManagementRBAC.ReadWrite.All` | custom roles with who has them, scope tags |
   | `DeviceManagementManagedDevices.ReadWrite.All` | device categories |
   | `Policy.ReadWrite.ConditionalAccess`, `Policy.Read.All` | Conditional Access policies, named locations |
   | `Policy.ReadWrite.AuthenticationMethod`, `…AuthenticationFlows`, `…Authorization`, `…DeviceConfiguration`, `…ExternalIdentities`, `…SecurityDefaults`, `Directory.ReadWrite.All` | the Entra policies (protected by default) |
   | `Group.Read.All` | check that the assigned groups still exist, find the groups of a role by name |

   Only the types you want to restore need their permission; a missing one is named when Intune refuses.
3. *Internal notes*: *Writes to Intune for a restore of Intune Config Backup, only after a second person approved it in the web UI. Signs in from the service connection icb-restore.*

### `IntuneConfigBackup.Web`: sign-in to the web UI

One registration is the page and its API at the same time.

1. Name `IntuneConfigBackup.Web`, redirect URI platform **Single-page application**; the address follows in step 10, leave it empty for now.
2. *Expose an API → Add* the Application ID URI `api://<client ID>`, then *Add a scope*: `access_as_user`, *Admins and users*, display name *Use Intune Config Backup*, enabled.
3. *App roles → Create app role*, three times, allowed member types *Users/Groups*:

   | Display name and value | Description |
   |---|---|
   | `Backup.Read` | Can read the Intune configuration backup and start a backup. |
   | `Backup.Restore` | Can start a restore and approve the restore of someone else. Includes reading. |
   | `Backup.Admin` | Can change the backup settings, more tenants and replicas. Includes reading. |

4. *API permissions → Add a permission → My APIs → IntuneConfigBackup.Web → Delegated → access_as_user*, then *Grant admin consent*.
5. *Enterprise applications → IntuneConfigBackup.Web → Properties*: *Assignment required* **Yes**, so only assigned people get a token.
6. *Internal notes*: *Sign-in and roles for the web UI of Intune Config Backup (Container App ca-icb-web). Roles through the groups G_ICB_*.*

The web UI gets no Graph permission: it reads the backup in Azure Repos, never Intune.

### `IntuneConfigBackup.Deploy`: rolls out the web UI

1. Name `IntuneConfigBackup.Deploy`, no API permissions.
2. *Internal notes*: *Deploys the web UI of Intune Config Backup into the resource group. Signs in from the service connection icb-deploy.*

## 2. Entra groups

*Entra admin center → Groups → New group*, type **Security**, membership **Assigned**:

| Group | Description | Role in `IntuneConfigBackup.Web` |
|---|---|---|
| `G_ICB_Readers` | *Read the Intune configuration backup* | `Backup.Read` |
| `G_ICB_Restore` | *Restore Intune objects from the backup and approve restores* | `Backup.Restore` |
| `G_ICB_Admins` | *Settings of the Intune configuration backup* | `Backup.Admin` |

Then *Enterprise applications → IntuneConfigBackup.Web → Users and groups → Add user/group*: each group with its role.

- `G_ICB_Restore` needs at least two people: one asks, another approves.
- Only *direct* members get the role. Assigning groups needs Entra ID P1; without it, assign people one by one.
- The roles are in the token: after a change, sign out and in again.

## 3. Agent

Microsoft-hosted agents need a parallel job that a new organization does not have; an agent of your own costs less and is ready at once.

1. *Organization settings → Agent pools → Add pool*: *Self-hosted*, name **`IntuneConfigBackupPool`**, description *Agent for the Intune Config Backup pipelines*, *Grant access permission to all pipelines* on.
2. Azure: *Create a virtual machine* in the resource group: name `vm-icb-agent`, Ubuntu Server 24.04 LTS, size **B2ats v2**, authentication with an SSH key, *Public inbound ports: None*, OS disk *Standard SSD*. The agent only connects out to Azure DevOps; nobody needs to reach the VM.
3. *User settings → Personal access tokens → New token*: name `icb-agent-registration`, expiry one day, scope *Agent Pools: Read & manage*.
4. VM → *Operations → Run command → RunShellScript*: the content of `deploy/agent/install.sh`, with this as the first line:

   ```bash
   export ADO_URL=https://dev.azure.com/<organization> POOL=IntuneConfigBackupPool PAT=<token>
   ```

   It installs Git, jq, Node.js 24, the Azure CLI and Docker, downloads the agent Azure DevOps offers and registers it as a service. Messages about `libicu` are harmless.
5. The agent shows *Online* under the pool's *Agents*. **Revoke the token**: the agent keeps its own credential.

## 4. Repositories

1. *Repos → Import repository*: this code as **`intune-config-backup`**.
2. *New repository* **`intune-config-backup-data`**, *Add a README* on (the backup needs a first commit on `main`).
3. *Project settings → Repositories → intune-config-backup-data → Security* → **`<project> Build Service (<organization>)`**: **Contribute** and **Create branch** on *Allow*. The backup commits and pushes with the identity of the pipeline.
4. Same page, *Policies* of `main` or *Security* for *Project Contributors*: **Force push** and **Delete or disable repository** on *Deny*. The history is the backup; nobody should rewrite it.

## 5. Variable groups

*Pipelines → Library → + Variable group*, *Allow access to all pipelines* off (each pipeline asks once on its first run).

**`intune-config-backup`**, description *Settings and IDs for the pipelines and the web UI*:

| Variable | Value |
|---|---|
| `ICB_SETTINGS` | `{}`; the Settings page of the web UI writes it from then on |
| `WEB_TENANT_ID` | Directory (tenant) ID |
| `WEB_CLIENT_ID` | Application (client) ID of `IntuneConfigBackup.Web` |
| `RESOURCE_GROUP` | the resource group, e.g. `rg-intune-config-backup` |
| `REGISTRY_NAME` | name of the container registry, globally unique, lowercase letters and digits (e.g. `acricbcontoso`) |
| `BACKUP_INCLUDE_OMA_VALUES` | optional, `true` for encrypted OMA-URI values in plain text ([Backup Format](06-backup-format.md)) |
| `REPLICA_TARGETS` | optional, the [replicas](09-replicas.md) |
| `ICB_APPROVALS` | not by hand: the web UI keeps the requests under *Restores* here, and `icb-restore` checks them |
| `ICB_SELF_APPROVAL` | optional, **for a test only**: `true` lets the one who asked approve their own restore, in the web UI and in `icb-restore`. Not on the Settings page on purpose; delete it, or set `false`, after the test. Takes up to a minute in the web UI |

No secret may be in this group: the web UI writes it as a whole, and Azure DevOps refuses that while a secret is in it.

**`intune-config-backup-secrets`**, description *Secrets for the pipelines*: `NOTIFY_WEBHOOK_URL`, the URL of the Teams workflow ([Web UI](03-web-ui.md#teams-message)), with the lock on. Leave it empty until you want Teams messages.

## 6. Service connections

*Project settings → Service connections → New service connection → Azure Resource Manager*, three times:

| Name | Description | App registration | Role on the resource group |
|---|---|---|---|
| **`icb-reader`** | *Reads Intune for the backup* | `IntuneConfigBackup.Reader` | Reader |
| **`icb-restore`** | *Writes to Intune for a restore after approval* | `IntuneConfigBackup.Restore` | Reader |
| **`icb-deploy`** | *Deploys the web UI* | `IntuneConfigBackup.Deploy` | Contributor, and Role Based Access Control Administrator limited to assigning **AcrPull** to service principals |

For each:

1. *Identity type*: **App registration or managed identity (manual)**, *Credential*: **Workload identity federation**. Name, description and Directory (tenant) ID → *Next*.
2. The form shows an **Issuer** and a **Subject identifier**. In Entra, at the app registration: *Certificates & secrets → Federated credentials → Add credential → Other issuer*: paste both, name `ado-<connection>` (e.g. `ado-icb-reader`), description *Azure DevOps service connection <connection>*, audience as offered.
3. Azure: the resource group → *Access control (IAM) → Add role assignment*: the role from the table for the app registration. Reader is only there so that the Azure CLI can sign in; the Intune permissions come from Graph.
4. Back in Azure DevOps: scope *Subscription*, subscription ID and name, the Application (client) ID, *Grant access permission to all pipelines* off.
5. **Verify and save**. Verify reads the whole subscription, which a role on the resource group does not allow: then *Save without verification* (arrow next to the button). The first pipeline run shows whether the sign-in works.

## 7. Environment

*Pipelines → Environments → New environment*: **`intune-restore`**, resource *None*, description *Restores of Intune objects*. It needs **no** *Approvals* check: the second person approves in the web UI (*Restores*), and the job *apply* checks that approval itself before it writes ([Web UI](03-web-ui.md#approvals)). An *Approvals* check here would ask for a second, separate yes in Azure DevOps; remove it if it is there (*⋯ → Approvals and checks*).

Under *⋯ → Security* only the identity of the web UI and the administrators need the role *User*.

## 8. Pipelines

*Pipelines → New pipeline → Azure Repos Git → intune-config-backup → Existing Azure Pipelines YAML file*, branch `main`, then *Save* (arrow next to *Run*), *⋯ → Rename/move*: the name, folder `\IntuneConfigBackup`. Descriptions are not part of a pipeline; the first lines of each YAML file say what it does.

| File | Name | Does |
|---|---|---|
| `/azure-pipelines/backup.yml` | **`icb-backup`** | reads Intune and commits the changes, on the schedule of the Settings page or with *Back Up Now* |
| `/azure-pipelines/restore.yml` | **`icb-restore`** | plan with the reader, then after the approval writes with the restore identity |
| `/azure-pipelines/replicate.yml` | **`icb-replicate`** | copies objects into a replica tenant, after a plan and an approval |
| `/azure-pipelines/deploy-web.yml` | **`icb-deploy-web`** | builds and rolls out the web UI on every change of `main` |

After a restore, `icb-restore` starts a backup so the history shows it at once. For that, on **`icb-backup`**: *⋯ → Manage security* → **`<project> Build Service (<organization>)`** → **Queue builds** on *Allow*. Without it the restore still writes; the run ends with a warning and the next scheduled backup shows it.

## 9. First backup

`icb-backup` → *Run pipeline* → *Run*. On the first run Azure DevOps asks to permit the variable groups, the service connection, the agent pool and the data repository: *View → Permit* for each. The first run backs up everything and reads the audit log a year back (about 10 minutes); from then on every change is a commit of the person who made it.

The history: *Repos → intune-config-backup-data*, branch `main`. *History* of a file shows every change of that object.

## 10. Web UI

1. The subscription needs the resource providers **`Microsoft.App`** and **`Microsoft.ContainerRegistry`** (*Subscription → Settings → Resource providers → Register*). It takes a role on the subscription, once.
2. `icb-deploy-web` → *Run*. With `deploy/azure/ado.bicep` it creates the container registry, the Container Apps environment and the Container App `ca-icb-web` with a system-assigned managed identity, builds the image on the agent and waits for the new version. On the very first run the last step fails: the identity has no access to Azure DevOps yet.
3. *Organization settings → Users → Add users*: the **Object (principal) ID** of the identity (Azure: `ca-icb-web` → *Identity*), access level **Basic**, project with the group **Project Contributors**, *Send email invites* off. That lets it read the history and start the pipelines.
4. *Pipelines → Library → Security* (the button of the library, above the list of groups, not the one inside a group) → *Add*: the identity as **Administrator**, so that the Settings page and approvals can save. Administrator on the group alone is not enough: Azure DevOps answers *403, you do not have permissions to perform this operation on the variable group*.
5. Run `icb-deploy-web` again. The address is in the log (*Live: …*) and on the Container App's *Overview*.
6. `IntuneConfigBackup.Web` → *Authentication → Single-page application*: the address with a trailing slash as redirect URI.

Open the address and sign in. From then on every change of `main` that touches the code rolls the web UI out by itself.

How it fits together:

- **Sign-in**: the API accepts only tokens of `IntuneConfigBackup.Web` from your tenant, for a person, with one of the three roles.
- **History**: the container clones `intune-config-backup-data` at start and pulls it every 5 minutes, signed in with its managed identity (no PAT, no key). Branches of more tenants are cloned when first opened.
- **Pipelines**: *Back Up Now*, a restore, a replica and the settings go through the Azure DevOps REST API, as the same identity. It has no Intune permission; writing stays with `icb-restore` after the approval.
- **Logs**: `ca-icb-web` → *Monitoring → Log stream*. No Log Analytics workspace, so nothing is kept and nothing costs.

## 11. Project page and wiki

- *Overview → Wiki → Publish code as wiki*: repository `intune-config-backup`, branch `main`, folder `/docs`, name *Intune Config Backup*. The pages keep the order of `docs/.order`; pictures are in `docs/.attachments`, which the wiki does not list.
- *Overview → Summary → About this project*: the description *Intune backup in Git: who changed what, compare versions, restore after approval* and *Readme file* `intune-config-backup` `/README.md`.
- *Project settings → Overview → Azure DevOps services*: turn off *Boards*, *Test Plans* and *Artifacts*; the project needs only Repos and Pipelines.

## Day to day

| | How |
|---|---|
| Backup | on the schedule of the Settings page, or *Back Up Now* in the web UI. A slot the schedule leaves out shows as a short run named *skipped* |
| Restore | from the web UI ([Restore](04-restore.md)), or *icb-restore → Run pipeline* |
| Replica | from the web UI, or *icb-replicate → Run pipeline* ([Replicas](09-replicas.md)) |
| Approve a restore | the e-mail of Azure DevOps, *Pipelines → Environments → intune-restore*, or the run itself (*Review*) |
| Agent | Ubuntu updates itself; the agent updates itself when Azure DevOps asks (*Agent pools → IntuneConfigBackupPool → Update all agents*) |
| A failed run | *Project settings → Notifications → New subscription → Build → A build fails*, filter on the folder `\IntuneConfigBackup`: an e-mail for every failed run ([Maintenance](05-maintenance.md)) |

## More tenants and replicas

The apps in another tenant trust the same service connections: give them a federated credential with the **Issuer** and **Subject identifier** of `icb-reader` (reader apps) or `icb-restore` (writer apps of a replica), from *Project settings → Service connections → <connection> → Edit*. The pipelines sign in to the other tenant with the token of that connection; no secret there either.

- More tenants: on the Settings page ([More Tenants](08-tenants.md)); one job per tenant, each into its branch.
- Replicas: `REPLICA_TARGETS` in the variable group and per replica an environment `intune-replica-<name>` with its approval ([Replicas](09-replicas.md)).
