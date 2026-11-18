# 04 Restore

Puts one object (or a selection) back into Intune as it was in a backup version: an update in place, or, for a deleted object, a new object with the same content (Intune gives it a new ID). Restore never deletes anything. It works in the main tenant (the one of the service connections); more tenants are backed up only ([More Tenants](08-tenants.md)).

Always two steps:

1. **Dry run** (in the web UI: the check after *Restore…*). Compares Intune now with the backup version and shows the difference (`-` now, `+` after the restore). Read permissions are enough; `IntuneConfigBackup.Reader` does it.
2. **Restore.** Writes the version, then reads the object back and shows what is still different (usually fields Intune fills in itself). Needs `IntuneConfigBackup.Restore`, the identity that may write, kept apart from the backup, and the approval of a second person.

| Restorable | How |
|---|---|
| Settings Catalog | the whole policy with all settings in one `PUT` (as the Intune portal does), or `POST` for a new one |
| Device Configurations | `PATCH`, or `POST` |
| Compliance Policies | `PATCH`, then the actions with `scheduleActionsForRules`; a new one with its actions in one `POST` |
| Scripts, Shell Scripts, Remediations | `PATCH` or `POST` with the script files of the version |
| Assignment Filters, Compliance Scripts, Device Categories | `PATCH`, or `POST` |
| Administrative Templates | the policy with `PATCH`/`POST`, then its settings with `updateDefinitionValues`: the ones there now removed, the version's added, each bound to its ADMX setting and by label to its text boxes and lists |
| Imported ADMX | uploaded again with its ADML files when it was removed; Intune cannot change an uploaded file in place, so a changed one must be removed in Intune first |
| Endpoint Security (old format) | `PATCH` and `updateSettings`; a new one from its template with `createInstance` |
| Feature, Quality and Driver Updates | `PATCH`, or `POST` |
| Autopilot Profiles, Terms and Conditions | `PATCH`, or `POST`; the assignments are objects of their own: the ones there now removed, the version's added |
| Enrollment Configurations | `PATCH`, or `POST`, then the order with `setPriority` (the defaults keep theirs) |
| App Protection iOS, Android, Windows; App Configuration (apps) | `PATCH`, or `POST`, then the apps with `targetApps` |
| App Configuration (devices) | `PATCH`, or `POST` |
| Roles | the role with `PATCH`/`POST`, then who has it: its role assignments as in the version, the groups found again by name (the backup keeps names, not IDs) |
| Scope Tags | `PATCH`, or `POST` |
| Notification Templates | `PATCH`, or `POST`, then the text per language, matched by locale |
| Named Locations | `PATCH`, or `POST` |
| Conditional Access | `PATCH`, or `POST`, as in the version, state included: look at the plan before you approve, a policy can lock people out |
| Applications | `PATCH` of the properties, then the assignments with `assign`; a deleted app is uploaded again in Intune first (its file is not in the backup) |
| Custom Attributes | `PATCH` or `POST` with the script of the version |
| Compliance (Settings Catalog) | the whole policy in one `PUT`, or `POST` with its actions |
| Reusable Settings | `PATCH`, or `POST` |
| Apple Enrollment Profiles | `PATCH`, or `POST` below the ADE token the backup names |
| Tenant Settings | `PATCH` of the Intune settings and branding |
| Entra authentication methods, authentication flows, authorization, device registration, external identities, security defaults | `PATCH` of the policy (authentication methods: each method too) |
| Entra group settings | `PATCH` of the values, or `POST` |

Every type the backup holds can be restored, except the connectors and tokens (Apple push certificate, ADE and VPP tokens, Managed Google Play, partners), Entra domains and the inventories (Autopilot devices, Activation Lock): they are there to read and to warn before they expire. An app is restored with its properties and assignments; a deleted app is uploaded again in Intune first, its file is not in the backup. The dry run already says so (*Cannot restore*), so nobody is asked to approve it. Assignments are set to those of the version (`assign`, which replaces the list), unless `--no-assignments`. Tested in the lab tenant: device configurations; the other types follow Graph's documentation and are tested against a simulated Graph, so try a dry run and a first restore of each in a test tenant.

**Warned**, still possible: a Conditional Access policy that is on for all users and excludes nobody (not even the emergency accounts), or that blocks all users: the plan, the web UI and the Teams message say so before anyone approves.

**Refused**, nothing is written:

- The version has a value the backup did not keep (`<redacted: not backed up>`, e.g. a Wi-Fi password, or an encrypted OMA-URI value without `--include-oma-values`). Restoring it would overwrite the real value with the placeholder.
- An assigned group no longer exists. Create it again, or restore without assignments.
- An assigned filter no longer exists. A filter restored again has a new ID: set it on the assignments in Intune, or restore without assignments.
- A scope tag of the version no longer exists (also without assignments: scope tags decide who sees the object). A scope tag restored again has a new ID: set the scope tags in Intune, or pick another version.
- An imported ADMX that is still there but different, or one whose ADMX file the backup does not have.
- A deleted object when one of the same type and name exists again (restored already, or made by hand): a second create would make a duplicate. Restore the one that exists instead.

## From the web UI

With the role `Backup.Restore` the object page has *Restore…* for the version shown, with or without the assignments. The page starts the pipeline `icb-restore` as a dry run and shows what would change in Intune; *Ask for Approval* is the yes, with a reason that is required: it goes to the run and, after the next backup, into the history. Then the page moves to the request under *Restores*: it waits there (*Waiting*) for the second person, then shows the restore (*Running*) and what Intune looks like afterwards (*Done*). One request, one place, from asking to done.

Several objects at once: tick them on *Objects* or *Changes*, then *Actions* ([Web UI](03-web-ui.md#restore-several-objects)). They run in one pipeline run; one that fails does not stop the others.

Every restore needs a second person with `Backup.Restore` who approves on the *Restores* page, like Multi Admin Approval in Intune ([Web UI](03-web-ui.md#approvals)).

## In Azure DevOps

*Pipelines → IntuneConfigBackup → icb-restore → Run pipeline*:

| Parameter | |
|---|---|
| Object ID | the object's ID; the web UI shows it with the version at the bottom of the object page (*More → Copy Object ID*). Several as `<id>@<version>`, separated by spaces |
| Backup version | the commit; `-` means the newest version with content |
| Write to Intune | off: dry run only. On: restore after the dry run; needs an approval from the web UI (`approvalId`), so a run started by hand with this on stops before it writes |
| Restore the assignments too | default on |

The other parameters (who asked, who approved, the approval, the reason) are set by the web UI; leave them at `-`.

The run has two stages:

1. **Dry run**, job *plan*: signs in with `icb-reader`, compares and writes the plan into the run's summary (*Extensions* tab) and the log. With Teams messages on, the channel gets a card that the restore waits.
2. **Restore**, job *apply*, only with *Write to Intune*: a deployment to the environment `intune-restore`. Its first step, *Second person approved*, reads the approval from `ICB_APPROVALS` of the variable group and writes only when it is approved, by someone other than the one who asked, for exactly the objects and versions of the run, within the last day. Then it signs in with `icb-restore`, writes, reads back and starts `icb-backup`, so the history shows the restore at once.

The write identity can only be used by this job, so only after an approval. Setup: [Setup](02-setup.md), steps 1, 6 and 7.

## On the command line

```bash
az login --tenant <tenant>    # an account or app that may write, for the restore itself
npx tsx src/cli.ts restore <object ID> --repo backup                    # dry run, newest version with content
npx tsx src/cli.ts restore <object ID> --repo backup --version 1a2b3c4d  # dry run of that version
npx tsx src/cli.ts restore <object ID> --repo backup --version 1a2b3c4d --apply
```

`--no-assignments` leaves the assignments as they are, `--summary <file>` also writes the plan and the result as Markdown. Exit code 0: done or nothing to do; 1: refused (the reason is printed); 2: could not start.

## After a restore

The backup that follows saves the change under the person who asked, with who approved it and the reason; the Intune audit log names only `IntuneConfigBackup.Restore (app)`, which wrote it. A re-created object appears under its new ID; the deleted one stays in the history under the old ID.
