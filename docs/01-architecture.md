# 01 Architecture

Intune has no history of its configuration and no recycle bin: a changed setting or a deleted policy is gone, and the Intune audit log says only that something changed, not what it was before. Intune Config Backup keeps that history in Git, in Azure DevOps.

## Parts

| Part | Runs in | Identity | Does |
|---|---|---|---|
| Backup | pipeline `icb-backup` on an agent of its own, on the schedule from the Settings page (default every 6 hours), after a restore and with *Back Up Now* | `IntuneConfigBackup.Reader` through the service connection `icb-reader`, read permissions only | reads every object type through Microsoft Graph, normalises it, reads the Intune and Entra audit logs and commits one change per person; more tenants each into their own branch |
| History | Azure Repos `intune-config-backup-data` | the build service, *Contribute* on this repository only | one file per object, one commit per person, the tenant documentation |
| Web UI | Azure Container Apps | `IntuneConfigBackup.Web` (sign-in), a managed identity (Azure DevOps) | shows the history, compares versions, searches, reads the tenant documentation, switches tenants, starts a backup or a restore; no Intune permissions at all |
| Restore | pipeline `icb-restore`, started from the web UI or by hand | dry run: `icb-reader`; writing: `IntuneConfigBackup.Restore` through `icb-restore` | puts one object or a selection back as it was in one version, after an approval in the environment `intune-restore` |
| Replicas | pipeline `icb-replicate` | apps in the replica tenant that trust the same service connections | copies objects into DEV or TEST after a plan and an approval |
| Deploy | pipeline `icb-deploy-web`, on every change of the code | `IntuneConfigBackup.Deploy` through `icb-deploy`, Contributor on one resource group | builds the image on the agent, rolls out the web UI, waits until the new version answers |

## Flow

The picture and the seven steps are on the [start page](../README.md#architecture).

## Design decisions

| Decision | Why |
|---|---|
| Git as the store | history, diff, blame and retention for free; readable without the tool; nothing to run for storage |
| Pipelines on an own agent instead of a server | nothing to patch but one small VM, workload identity federation instead of secrets, the run log is the audit trail, about 12 CHF a month |
| Read-only backup identity | a backup that can change the tenant is a risk of its own; the backup cannot do harm |
| A separate write identity behind an approval | restoring is rare and must be deliberate; two people for a write, and only one job can use that identity |
| Matched by object ID | renaming an object or two objects with the same name never mix anything up |
| Secrets never in Git | passwords and pre-shared keys are written as `<redacted: not backed up>`; restore refuses a version that would write the marker back |
| Web UI without Intune permissions | it reads the Git history; a restore from the page starts the pipeline with an identity that may only start pipelines and save the settings |
| Container Apps scaled to zero | a web UI for a team stays inside the monthly free grant; no Log Analytics workspace |

## Compared with other approaches

Exports by script or tools such as IntuneCD back up the configuration too. Intune Config Backup covers the same types and adds the person behind each change from the audit log, diffs without noise, a strict split between a read-only backup and an approved restore, and a web UI with Entra sign-in and roles for people who never open Git.
