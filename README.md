<h1><img src="docs/images/app-icon.svg" alt="" width="36" align="absmiddle">&nbsp;Intune Config Backup</h1>

> **Portfolio showcase – documentation, screenshots and a live demo.** The source code is private. This repository shows what the product does and how it is built and operated. No license is granted (see [LICENSE](LICENSE)); all IDs, host names and domains are placeholders (`contoso.com`, `00000000-…`).

Intune keeps no history of its configuration and has no recycle bin. **Intune Config Backup** saves every policy, profile, script, app assignment, Conditional Access and Entra policy into Git on a schedule (all that IntuneCD covers), names **who changed what** from the Intune audit log, shows it all in a **web UI**, writes a **tenant documentation** you read in the web UI like a wiki, backs up **several tenants**, puts a single object back when needed, **after an approval** (for critical types by a second person, like Multi Admin Approval), and keeps other tenants (DEV, TEST) as **replicas** of this one.

> [!TIP]
> ### [Try the web UI live in your browser](https://marc-baumgartner.github.io/intune-config-backup/)
> The real web UI with a made-up tenant and two weeks of history, no sign-in needed. Start with **Windows - BitLocker** to see who changed what, or open **Changes**. A restore runs as a simulation; nothing is sent anywhere.

<p>
  <a href="docs/videos/web-ui-tour.mp4"><img src="docs/images/web-ui-tour.gif" alt="Web UI tour: the cards and the changes per day, notifications, the changes with what changed in each and the Actions menu, a BitLocker policy with its history and settings, a restore that checks first, the objects sorted by last change and the deleted ones, a search, the documentation with export, the settings by area and dark mode"></a>
  <br>
  <sub><b>Video:</b> a tour of the web UI in demo mode at double speed: the cards and the changes per day, notifications, the changes with what changed in each and the Actions menu, a policy with its history and settings, a restore that shows what would change first, the objects sorted by last change and the deleted ones, a search, the documentation with export, the settings by area and dark mode. <a href="docs/videos/web-ui-tour.mp4">Full resolution (MP4)</a></sub>
</p>

## At a glance

| Part | Technology |
|---|---|
| Backup | Node.js 24 LTS (TypeScript), Microsoft Graph, in Azure Pipelines on an own agent, as the Settings page says (every 6 hours to weekly) or at once with *Back Up Now* ([02](docs/02-setup.md)) |
| Sign-in | workload identity federation for the pipelines (service connections), a managed identity for the web UI: no client secret, no certificate, no PAT |
| Storage | Azure Repos, one JSON file per object, one commit per person from the Intune audit log |
| Web UI | React with Tailwind CSS in the look of shadcn/ui (tweakcn theme *Modern Minimal*) and Base UI, on a Fastify server, Entra sign-in with app roles, Azure Container Apps (scales to zero) |
| Restore | one object or a selection, with a reason; dry run, approval, own identity with write permissions; most types |
| Restores | like Multi Admin Approval: every restore is written only after a second person approves in the web UI; the pipeline checks that approval again |
| Replicas | other tenants as a copy of this one: plan, approval per replica, matched by name, nothing deleted |
| Documentation | Markdown pages in the history with every run, read in the web UI like a wiki: every object's settings in plain words and its assignments; print or download |
| Tenants | one run backs up more tenants, each to its own branch; the web UI switches between them ([08](docs/08-tenants.md)) |
| Activity | four cards and the changes per day as a chart (3 months, 30 or 7 days) on the start page, every change below as a sortable table, filtered by period, person and type; ticked rows undone, exported or linked through one *Actions* menu; notifications for what needs someone |
| Docs | the [docs](docs) folder of this repository |

**51 object types**, the same coverage as IntuneCD: Settings Catalog and endpoint security, configuration profiles, compliance, scripts, remediations, filters, apps, enrollment and Apple tokens, roles and scope tags, Conditional Access and Entra policies. Most are restorable ([06](docs/06-backup-format.md)).

## Architecture

The backup reads Intune with read permissions only and writes into Git. The web UI reads the Git backup, never Intune. A restore is a pipeline of its own: a dry run, an approval, then a separate identity that may write.

![Intune Config Backup on Azure DevOps in seven steps: admins change Intune, icb-backup reads it through Microsoft Graph and commits per person into intune-config-backup-data, the web UI in Azure reads the history, people sign in, a restore starts from the web UI and writes back after an approval](docs/images/architecture.png)

1. **Admins change Intune.** People change policies, profiles and apps in the Intune admin center as always; Intune records each change in its audit log.
2. **The backup reads, read only.** `icb-backup` runs on the schedule of the Settings page or with *Back Up Now*. It signs in through the service connection `icb-reader` without a secret and reads every type and the audit logs through Microsoft Graph.
3. **One commit per person.** Each changed object is one file in `intune-config-backup-data`, committed under the person who changed it, at the time they changed it. The run also writes the tenant documentation and sends a Teams card for important changes.
4. **The web UI reads the history.** The Container App clones the repository with its managed identity and pulls it every 5 minutes. It has no Intune permission.
5. **People sign in with Entra ID.** Their group decides what they may do: read, restore or change the settings.
6. **A restore starts from the web UI.** Someone picks a version and gives a reason; the web UI starts `icb-restore`, which first shows what would change.
7. **Writing waits for a second person.** Once someone else approved it under *Restores* in the web UI, and the job has checked that approval, it signs in as `IntuneConfigBackup.Restore`, writes the version, reads it back and starts a backup, so the history shows the restore under the person who asked.

The agent pool, the service connections and the variable groups are shared by every pipeline; `icb-deploy-web` builds the web UI on the agent and rolls it out through the container registry on every change of `main`.

## Who changed what

The Intune audit log names the person behind each change. Each person gets their own commit, at the time of their change, so the history of a file is the history of that policy. The first backup reads the audit log a year back, so every object starts with who changed it last and who created it:

```
anna.meier@contoso.com: 1 changed

changed  Settings Catalog/Windows_-_BitLocker__d0000001-….json

Intune audit log:
  Patch DeviceManagementConfigurationPolicy 2026-11-02 09:46 UTC Windows - BitLocker
```

Reads, and the backup's own app, are left out. What the audit log does not explain goes into a commit of the backup itself ([06](docs/06-backup-format.md)).

## Permission model

Four app registrations, each with only what it needs, all signed in through service connections with workload identity federation. People get their roles in the web UI through Entra groups; an account without a role or from another organization does not get in. Writing to Intune needs a second key: an approver of the environment `intune-restore` in Azure DevOps.

| Entra group | Role | May |
|---|---|---|
| `G_ICB_Readers` | `Backup.Read` | read everything, *Back Up Now* |
| `G_ICB_Restore` | `Backup.Restore` | restore, approve the restore of someone else |
| `G_ICB_Admins` | `Backup.Admin` | settings, more tenants, replicas |

## Web UI

What changed and who changed it, every version of an object, the comparison of any two versions in the words of the Intune portal, deleted objects and a search through names, settings, groups and scripts. Built in the look of shadcn/ui with the tweakcn theme *Modern Minimal*: a sidebar, cards, sortable tables with one *Actions* menu for ticked rows, light and dark, works on phone width. (Screenshots from the built-in demo mode with sample data.)

### Changes

The start page: four cards (objects, changes of the week against the one before, restores waiting for approval, the last backup), the changed objects per day as a chart, then every change as a table grouped by day, one row per object with type, kind, person and time. A click on a column title sorts; the arrow at the end opens what changed; ticked rows can be undone together, exported as CSV or linked. *Back Up Now* sits at the top of the sidebar; *Notifications* at the bottom count what needs someone (a failed backup, restores that wait, Apple tokens that run out), each dismissed until the next day.

| | |
|---|---|
| ![Changes: four cards, the changes per day as a chart, then every change as a table](docs/images/web-ui-overview.png)<br><sub>The start page: four cards, the chart, then every change</sub> | ![A change opened and two rows ticked, with the Actions menu](docs/images/web-ui-changes.png)<br><sub>A change opened, two rows ticked, the Actions menu</sub> |
| ![Notifications in the sidebar: a restore waiting, Apple tokens that run out](docs/images/web-ui-notifications.png)<br><sub>Notifications: what needs someone, each can be dismissed</sub> | ![The start page in dark](docs/images/web-ui-overview-dark.png)<br><sub>The same in dark</sub> |

### One object, every version

The history on the left, the change in plain words on the right. Every version can be compared with the one before or any older one, read as settings or as JSON.

| | |
|---|---|
| ![BitLocker policy: encryption method changed from XTS-AES 128 to 256 bit by anna.meier](docs/images/web-ui-object-changes.png)<br><sub>A Settings Catalog change: the setting by its name, before and after</sub> | ![The whole version as settings: basics, settings, assignments](docs/images/web-ui-object-settings.png)<br><sub>The whole version: basics, settings and assignments in plain words</sub> |
| ![JSON of the version with the changed lines](docs/images/web-ui-object-json.png)<br><sub>JSON view: the changed lines of the file in Git</sub> | ![Remediation: detection script changed line by line](docs/images/web-ui-script-changes.png)<br><sub>Scripts and remediations: the script changed line by line</sub> |

### Objects, search and deleted objects

| | |
|---|---|
| ![All objects in one table: name, type, platform, assigned to, last change](docs/images/web-ui-objects.png)<br><sub>All objects in one table, sorted by any column</sub> | ![Search: what has the text now, and in the history when it came or went](docs/images/web-ui-search.png)<br><sub>Search from the toolbar (Ctrl K): now, and in the history with who and when</sub> |
| ![Deleted objects, with who deleted them](docs/images/web-ui-deleted.png)<br><sub>Deleted objects stay in the history, with who deleted them</sub> | ![Wi-Fi profile restored: the history marks the restore and the version it put back](docs/images/web-ui-restored-object.png)<br><sub>A restore in the history: marked with the version it put back</sub> |

### Restore

Only for people with the role `Backup.Restore`. The page starts the pipeline `icb-restore` and follows it live, step by step; it never writes to Intune itself. When the Teams message is on, the approvers get a card with *Open in Azure DevOps* as soon as a restore waits for them. A Conditional Access policy that could lock everyone out is named in the plan before anyone approves it.

Several objects go together: *Objects → Select*, tick them, one check, one reason, one approval. Every restore needs a reason; it goes to the run and into the history.

Every restore needs a second person, like Multi Admin Approval in Intune: the yes after *Restore…* is *Ask for Approval*, and someone else with `Backup.Restore` approves on the *Restores* page after reading the check. The one who asked cannot approve it; the server checks that ([03](docs/03-web-ui.md#approvals)).

| | |
|---|---|
| ![Restore: what would change in Intune](docs/images/web-ui-restore-check.png)<br><sub>Restore… shows first what would change, nothing written yet</sub> | ![Waiting for an approval in Azure DevOps](docs/images/web-ui-restore-approval.png)<br><sub>After the yes: writing waits for the approval</sub> |
| ![Restored and read back](docs/images/web-ui-restore-done.png)<br><sub>Written, read back, and a backup started so the history shows it</sub> | ![Approval with the check as a diff and the reason for the decision](docs/images/web-ui-approval.png)<br><sub>Critical types: a second person reads it, then approves</sub> |

### Groups

Every group something is assigned to, as a table with the number of objects; one group shows what it gets now in a sortable table: type, included or excluded, the filter and the last change. Assignments on an object page link here.

| | |
|---|---|
| ![Groups with the number of objects assigned](docs/images/web-ui-groups.png)<br><sub>Every group, all devices and all users</sub> | ![What one group gets: type, assignment and last change](docs/images/web-ui-group.png)<br><sub>One group: what it gets and how</sub> |

### Replicas

Other tenants (DEV, TEST, a training tenant) as a copy of this one, on demand: a plan first, then the copy after an approval in Azure DevOps, per replica with its own approvers and identity. Objects matched by name, groups, filters and scope tags too; nothing in a replica is deleted ([09](docs/09-replicas.md)).

![Replicas: plan a copy to DEV or TEST](docs/images/web-ui-replicas.png)

### Settings

For people with the role `Backup.Admin`: the schedule (every 6 hours, twice a day, daily, weekly or off), more tenants, object types on or off, objects left out by name, OMA-URI values in plain text, and a Teams message after a run (off, only important changes, or all) with links into the web UI. Saved in the variable group of the pipelines and used from the next run.

![Settings: the sections on the left, the object types as cards by area with a switch each](docs/images/web-ui-settings.png)

### Documentation

Every backup writes the tenant documentation as Markdown; the web UI reads it like a wiki: the types on the left, an overview with the counts and who gets what, and one page per type that lists every object on one line and opens its settings, assignments and scripts on a click. Filter, export as CSV or Markdown, or print to PDF.

| | |
|---|---|
| ![Documentation overview: counts and what is assigned to which group](docs/images/web-ui-documentation-overview.png)<br><sub>The overview: counts and who gets what</sub> | ![Documentation: the Settings Catalog page, one line per policy](docs/images/web-ui-documentation.png)<br><sub>One type: one line per policy</sub> |

### More tenants

One run backs up more tenants (DEV, TEST, other customers), each to its own branch with its own history and documentation. The menu at the top of the sidebar switches between them ([08](docs/08-tenants.md)).

### Account and dark mode

Light, dark or automatic, switched in the account menu at the bottom of the sidebar. Under *Roles and Session* the menu shows when the person signed in, how long the token is valid, the tenant and every app role, with the assigned ones ticked.

| | |
|---|---|
| ![Account menu with roles and session opened](docs/images/web-ui-account-menu.png)<br><sub>Account menu: appearance, roles, session details</sub> | ![Account menu, dark](docs/images/web-ui-account-menu-dark.png)<br><sub>Account menu in dark mode</sub> |
| ![Object, dark](docs/images/web-ui-object-changes-dark.png)<br><sub>An object in dark mode</sub> | ![Changes, dark](docs/images/web-ui-changes-dark.png)<br><sub>A change opened, dark</sub> |

### Phone

| | | | |
|---|---|---|---|
| ![The start page on a phone](docs/images/web-ui-phone-overview.png)<br><sub>Start page</sub> | ![A change on a phone](docs/images/web-ui-phone-object.png)<br><sub>A change</sub> | ![Changes on a phone](docs/images/web-ui-phone-changes.png)<br><sub>Changes</sub> | ![Objects on a phone](docs/images/web-ui-phone-objects.png)<br><sub>Objects</sub> |

## Permissions at a glance

| App registration | Who | Gets |
|---|---|---|
| `IntuneConfigBackup.Reader` | backup, dry run, through `icb-reader` | Graph application permissions `…Read.All` only ([07](docs/07-permissions.md)) |
| `IntuneConfigBackup.Restore` | the apply job of a restore, only after the approval in the environment `intune-restore` | the `…ReadWrite.All` of the types it restores, Entra policies only if wanted ([04](docs/04-restore.md)) |
| `IntuneConfigBackup.Web` | the web UI and its API | app roles `Backup.Read`, `Backup.Restore` and `Backup.Admin`, given to people through Entra groups; nobody without a role gets in |
| `IntuneConfigBackup.Deploy` | the pipeline that deploys the web UI | Contributor on one resource group |

## How it is built

| Area | Approach |
|---|---|
| Code | TypeScript throughout: one reader per object type, a normaliser that leaves out IDs, timestamps and secrets, Git through the `git` CLI, a Fastify server and a React web UI |
| Clean diffs | sorted keys, settings and assignments in a fixed order, names next to IDs, matched by object ID; a run without changes makes no commit |
| Security | read-only identity for the backup, a separate write identity behind an approval, workload identity federation and a managed identity instead of secrets, secrets never in Git, the web UI without Intune permissions, Content Security Policy |
| Operations | Azure Pipelines on one small Ubuntu agent for backup, restore, replicas and deploy, history in Azure Repos, Azure Container Apps scaled to zero, Teams cards for failed runs and tokens that run out |
| Quality | tests for normalising, Graph errors, Git, audit attribution, restore plans, readable names and the web server; the built server is started with production dependencies only before every deployment |
| Web UI | React 19, Tailwind CSS 4 in the look of shadcn/ui with the tweakcn theme *Modern Minimal*, Base UI for menus and lists, Recharts, MSAL (PKCE), light and dark appearance |

## Documentation

| | |
|---|---|
| [01 Architecture](docs/01-architecture.md) | parts, flow, design decisions |
| [02 Set up in Azure DevOps](docs/02-setup.md) | step by step: app registrations, groups, agent, repositories, variable groups, service connections, environment, pipelines, web UI |
| [03 Web UI](docs/03-web-ui.md) | pages, **roles through Entra groups**, settings, Teams message, approvals |
| [04 Restore](docs/04-restore.md) | dry run, approval, write identity, what is refused |
| [05 Maintenance](docs/05-maintenance.md) | what tells you, what expires, when something fails |
| [06 Backup format](docs/06-backup-format.md) | the types, a backup file, who changed what, the tenant documentation |
| [07 Permissions](docs/07-permissions.md) | every Graph call and the permission it needs |
| [08 More tenants](docs/08-tenants.md) | more tenants in the same backup and web UI |
| [09 Replicas](docs/09-replicas.md) | tenant to tenant: DEV, TEST and others as a copy of PROD |
| [10 Command line](docs/10-command-line.md) | backup, restore and web UI on a workstation |
