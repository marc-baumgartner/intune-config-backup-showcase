# 05 Maintenance

What needs attention now and then, and what tells you so.

## What tells you

| What | When | Tells you |
|---|---|---|
| Teams card | after a failed run of `icb-backup`, `icb-restore` or `icb-replicate`, when Teams messages are on | what failed, with a button to the run |
| web UI, *Notifications* in the sidebar | always | a failed last backup, restores that wait, and the Apple push certificate, ADE and VPP tokens that expire within 60 days |
| e-mail of Azure DevOps | after a failed run, once you subscribe (below) | the pipeline and the run |
| `icb-deploy-web` | every change of the code on `main` | fails when the new version does not start, with Azure's revisions and logs in the run |

To get an e-mail for every failed run: *Project settings → Notifications → New subscription → Build → A build fails*, *Deliver to* the team or a group, filter *Definition name* contains `icb-`. Approvers of `intune-restore` get an e-mail for every restore that waits anyway.

## What expires

| What | Where | Expires | Renew |
|---|---|---|---|
| Federated credentials of the app registrations | Entra | never | nothing to do |
| Managed identity of `ca-icb-web` | Azure | never | nothing to do; if the Container App is deleted and created again, add the new identity to the organization and as Administrator of the library again ([Setup](02-setup.md), step 10) |
| Agent | VM `vm-icb-agent` | never | the token for its registration was only needed once; to register again, a new one-day token and `deploy/agent/install.sh` |
| `NOTIFY_WEBHOOK_URL` (Teams workflow) | variable group `intune-config-backup-secrets` | when the workflow in Teams is deleted or its owner leaves | new workflow in the channel ([Web UI](03-web-ui.md#teams-message)), replace the value, run `icb-deploy-web` |
| Apple push certificate, ADE and VPP tokens | Intune, *Tenant administration → Connectors and tokens* | yearly | renew in Intune with the same Apple ID; the web UI shows them from 60 days before, Teams gets a card from 30 days |
| Role assignments with PIM (e.g. time-bound Owner) | Azure | as assigned | extend in PIM |

Nothing else has a password, key or token that runs out.

## Updates

- **Ubuntu** on the agent installs security updates by itself.
- **Agent**: *Organization settings → Agent pools → IntuneConfigBackupPool → Update all agents* when Azure DevOps offers a new version.
- **npm packages**: `npm outdated`, update, `npm test`, push to `main`; `icb-deploy-web` rolls out the web UI.

## When something fails

- **Backup failed with "no permission (needs …)"**: a new type or a removed permission; add the named permission to `IntuneConfigBackup.Reader` and grant admin consent ([Permissions](07-permissions.md)).
- **A run waits with "no agent"**: the VM is stopped or the agent service is down. Start the VM; *Agent pools → IntuneConfigBackupPool → Agents* shows *Online* again within a minute.
- **The sign-in of a service connection fails** (`AADSTS700213` or similar): Issuer or Subject identifier of the federated credential do not match the connection. Compare them in *Service connections → Edit* and the app registration.
- **Web UI does not answer**: Azure portal → `ca-icb-web` → *Revisions and replicas* (is the newest revision healthy?) and *Monitoring → Log stream*. `icb-deploy-web` prints both when a deployment fails.
- **Web UI says it cannot start a run or save the settings**: the message starts with *Azure DevOps:* and says what Azure DevOps refused; mostly its identity lost its access level in the organization or its role Administrator of the library ([Setup](02-setup.md), step 10).
- **Restore ends with a warning at *Back up now*** (`TF215106 … needs Queue builds permissions`): the restore was written; only the backup after it could not start. Give the build service *Queue builds* on `icb-backup` ([Setup](02-setup.md), step 8).
- **Restore waits forever**: the request waits under *Restores* in the web UI for a second person. If the run itself waits with *Review*, the environment `intune-restore` still has an *Approvals* check from before: remove it ([Setup](02-setup.md), step 7). A run that still waits for a check removed meanwhile has no buttons any more: *Cancel* it. The web UI forgets a run that was cancelled or deleted, and its restore starts over.
- **Restore stops at *Second person approved***: the run's log says why (no approval, approved by the one who asked, other objects, older than a day). Ask for the restore again in the web UI.
