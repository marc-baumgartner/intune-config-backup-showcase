# 06 The backup

What the backup holds, how a file looks, and how a change is put under the person who made it.

## Types

| What | Folder in the backup |
|---|---|
| Settings Catalog (incl. endpoint security in the new format) | `Settings Catalog` |
| Device configuration profiles (templates, custom OMA-URI, update rings) | `Device Configurations` |
| Compliance policies with their actions | `Compliance Policies` |
| PowerShell scripts, with the script as `.ps1` | `Scripts` |
| Shell scripts (macOS), with the script as `.sh` | `Shell Scripts` |
| Remediations, with detection and remediation script | `Remediations` |
| Assignment filters | `Assignment Filters` |
| Administrative templates with each configured ADMX setting by name | `Administrative Templates` |
| Imported ADMX: file name, namespace, prefix, languages (Graph does not hand out the ADMX/ADML content itself, so keep the original files) | `Imported ADMX` |
| Endpoint security policies of the old format | `Endpoint Security (old format)` |
| Compliance scripts (custom compliance), with the script as `.ps1` | `Compliance Scripts` |
| Windows feature, quality and driver update profiles | `Feature Updates`, `Quality Updates`, `Driver Updates` |
| Autopilot deployment profiles | `Autopilot Profiles` |
| Enrollment configurations: enrollment status page, restrictions, Windows Hello for Business | `Enrollment Configurations` |
| App protection policies (iOS, Android, Windows) with their apps | `App Protection iOS`, `… Android`, `… Windows` |
| App configuration for managed devices and for managed apps | `App Configuration (devices)`, `App Configuration (apps)` |
| Custom Intune roles with their assignments, custom scope tags | `Roles`, `Scope Tags` |
| Terms and conditions, notification message templates, device categories | `Terms and Conditions`, `Notification Templates`, `Device Categories` |
| Conditional Access policies, with the names of the groups and named locations they name (`idNames`) | `Conditional Access` |
| Named locations (IP ranges, countries) | `Named Locations` |
| Apps (Win32, store, line-of-business, web links, VPP) with their assignments; the app files stay in Intune | `Applications` |
| macOS custom attributes, with the script as `.sh` | `Custom Attributes` |
| Compliance policies of the Settings Catalog (Linux, newer macOS and Windows) | `Compliance (Settings Catalog)` |
| Reusable settings of the Settings Catalog | `Reusable Settings` |
| Apple Automated Device Enrollment profiles | `Apple Enrollment Profiles` |
| Tenant-wide Intune settings and branding | `Tenant Settings` |
| Apple push certificate, ADE and VPP tokens, Managed Google Play, compliance, management and remote assistance partners: what they are and when they expire, no secrets | `Apple Push Certificate`, `Apple ADE Tokens`, `Apple VPP Tokens`, `Managed Google Play`, `Compliance Partners`, `Management Partners`, `Remote Assistance Partners` |
| Entra ID: authentication methods, authentication flows, authorization policy, device registration, external identities, security defaults, group settings, domains | `Entra …` |
| Off until turned on in the settings: Autopilot devices (serial number, group tag) and Activation Lock bypass codes (only with secrets in plain text) | `Autopilot Devices`, `Activation Lock` |

The built-in roles and scope tags Microsoft maintains are left out. The Entra settings IntuneCD reads through the internal Azure portal API (external collaboration, roaming settings, self-service password reset) have no Graph endpoint for an app and are left out.

## A file

Each object is `<Folder>/<Name>__<Object ID>.json`:

```json
{
  "id": "…",
  "name": "Windows - BitLocker",
  "type": "Settings Catalog",
  "object": { "…": "the object as Graph returns it, without IDs and timestamps" },
  "settings": [ "…Settings Catalog only, sorted by setting…" ],
  "settingNames": { "…setting ID…": { "name": "Encryption method", "options": { "…": "XTS-AES 256-bit" } } },
  "assignments": [
    { "target": { "kind": "group", "groupId": "…", "groupDisplayName": "Clients Bern", "filterDisplayName": "Windows 11 only", "…": "…" } }
  ]
}
```

- **Only real changes in a diff:** IDs, timestamps, versions and OData links are left out, keys are sorted, settings and assignments are in a fixed order. A run without changes writes nothing and makes no commit.
- **Names next to IDs:** assignments carry the name of the group and of the assignment filter.
- **No secrets in Git:** fields such as passwords and pre-shared keys are written as `<redacted: not backed up>`. So are custom OMA-URI values Intune stores encrypted, unless `--include-oma-values` is set: Intune encrypts many of them although they are no secret (an ADMX file, a list of groups), and without them such a profile cannot be restored. With the option the backup holds them in plain text, real secrets included, and the backup repository must be protected like a password safe. Intune only hands out these values to an identity with `DeviceManagementConfiguration.ReadWrite.All`, a write permission ([permissions](07-permissions.md)); without it the values stay redacted and the run warns.
- **Matched by ID:** renaming an object or two objects with the same name do not mix anything up; the name in the file name is only for reading.
- **Deleted objects:** their files are removed in the next run and stay in the Git history. A type Graph refuses (missing permission, outage) keeps its files and fails the run, so a gap never looks like a deletion. When all objects of a type are gone at once, or more than half of ten or more, the type is read again after 30 seconds (Graph sometimes answers with an empty list for a moment); only what is still gone then counts as deleted, and the run log says so.

- **Readable names:** Settings Catalog policies carry the names Intune shows for their settings and choices (`settingNames`), so the web UI shows "Encryption method: XTS-AES 256-bit" instead of an ID. Restore does not send them back.

## Who changed what

With `--commit` the run reads the Intune audit log since the last backup commit and gives every person their own commit: the author is the person (their UPN), the author date is the time of their change, and the message lists the audit log entries behind it. An app or pipeline that changed something gets a commit as `<app name> (app)`. *History* or *Blame* of a file in the data repository then shows who changed that object, not only when.

```
anna@contoso.local: 1 new, 1 changed, 1 deleted

new      Settings Catalog/Windows_-_BitLocker_v2__6d1e….json
changed  Device Configurations/Custom_OMA_Proxy__c3a0….json
deleted  Settings Catalog/Windows_-_BitLocker__6d1e….json

Intune audit log:
  Patch DeviceManagementConfigurationPolicy 2026-11-13 09:14 UTC Windows_-_BitLocker_v2
  Patch DeviceConfiguration 2026-11-13 08:52 UTC Custom_OMA_Proxy (by beat@contoso.local)
```

- Only changes count: Intune also logs some reads (reading an encrypted OMA-URI value is *Get DeviceConfiguration*), and the backup's own app (`BACKUP_OWN_APP_ID`, in the pipeline the client ID of `icb-reader`) is left out.
- A file belongs to the person who changed its object **last**; earlier edits by others are listed in that commit with their name.
- Matched by object ID (scope tags by their number), so a rename (new file, old file deleted) is one person's change.
- Deleting a scope tag takes it off every object that had it, but Intune logs only the deletion: those objects go to the person who deleted the tag, with *(scope tag N taken off)* in the message.
- What the audit log does not explain (a change Intune makes itself, an object the log has no ID for) goes into the usual commit of *Intune Config Backup*. So does everything when the audit log cannot be read (the run warns and needs `DeviceManagementApps.Read.All`, see [Setup](02-setup.md#intuneconfigbackupreader-reads-intune)). `--no-audit` turns it off.
- **The first backup** reads the audit log a year back, as far as Intune keeps it (`FIRST_BACKUP_AUDIT_DAYS`, at most 365). Each object is saved under the person who changed it last, dated at that change, and the commit says `First-Backup: yes`; the web UI shows it as *Last changed by …*. Only objects nobody touched in that year go into the commit of *Intune Config Backup*.
- **Conditional Access and named locations** are logged by Entra, not Intune: with `AuditLog.Read.All` the run reads the Entra audit log too (category *Policy*). Entra keeps it 30 days (7 without Entra ID P1), so a first backup names their people only for that time.
- **Who created it**: a commit names the creator of its objects from the first *Create* entry in the audit log (`Created-By: <id> <time> <person>`); the object page shows *Created by …*. Objects created before the year the audit log keeps have no creator.
- The backup itself stays the committer of every commit, so a person's commit is never mistaken for one they made in Git.

## Tenant documentation

Every run also writes the folder `Documentation` into the data repository: one Markdown page per object type, every object with its settings in the words of the Intune portal (the same as the web UI shows), its assignments and its scripts, and an overview with the number of objects per type and what is assigned to which group. The web UI reads them on its *Documentation* page like a wiki (filter, export, print); Azure Repos shows them as they are too, and the folder can be published as a wiki of its own (*Overview → Wiki → Publish code as wiki*, repository `intune-config-backup-data`, folder `/Documentation`).

- More tenants: each branch `backup-<name>` has its own ([More Tenants](08-tenants.md)).
- Always the state of the last backup: written anew from the backup files, so *Back Up Now* also brings the documentation up to date.
- In the same commit as the change it documents, so the history of a page goes with the history of the objects; a run without changes changes no page.
- Generated: do not edit by hand, the next run writes it again.

