# 09 Replicas (tenant to tenant)

Makes other tenants (DEV, TEST, a training tenant, as many as you like) a replica of this one (PROD), from the newest backup: what PROD has and the replica lacks is created, what differs is updated. Nothing in a replica is deleted; what only the replica has stays. Started on demand, per replica, from the web UI (*Replicas*, role `Backup.Admin`) or in Azure DevOps (*icb-replicate → Run pipeline*, with the name of the replica).

Always two steps, like a restore:

1. **Plan.** Reads the replica with a read-only identity and compares it with the backup, object by object. The summary of the run lists what would be created and updated, with the differences; nothing is written.
2. **Replicate.** Only with `apply`: waits for an approval of the environment `intune-replica-<name>`, then writes with its own identity in the replica. One object that fails does not stop the others; the result names each.

## How objects are matched

| What | In the replica found by |
|---|---|
| an object | its type and its name (IDs differ between tenants). Two objects with the same name in the replica: that one is not copied until one is renamed |
| an assigned group | its name. A group the replica does not have: the assignment is left out, with a warning in the plan. Groups are never created |
| an assignment filter | its name; filters are copied first, so a new one is there for the policies that use it |
| a scope tag | its name; scope tags are copied first. One the replica does not have: left out, the object keeps *Default* |
| a named location | its name; named locations are copied first |
| groups and named locations of a Conditional Access policy | their names; single users cannot be found in another tenant and are left out. A policy that is on in PROD is on in the replica too: the plan says so |
| the groups of a custom role | their names |

Not copied: values the backup did not keep (passwords, encrypted OMA-URI values without `--include-oma-values`); *App Configuration (devices)*, which points to the apps of PROD by their IDs; apps (their files are not in the backup); Apple enrollment profiles (they belong to an ADE token of PROD); and what is set once per tenant: the tenant settings and the Entra policies. The connectors and tokens cannot be written anywhere. The plan lists each with the reason.

## Set up a replica

Once per replica tenant; the example is `DEV`. Both apps sit in the **replica** tenant and trust a service connection of the project, so no secret is stored there either.

### 1. Two app registrations in the replica tenant

Both single tenant.

1. **`IntuneConfigBackup.CopyReader`**: for the plan.
   - *API permissions → Microsoft Graph → Application permissions*: the same as `IntuneConfigBackup.Reader` ([Setup](02-setup.md#intuneconfigbackupreader-reads-intune)), *Grant admin consent*.
   - *Certificates & secrets → Federated credentials → Add credential → Other issuer*: **Issuer** and **Subject identifier** of the service connection `icb-reader` (*Project settings → Service connections → icb-reader → Edit*), name `ado-icb-reader`.
2. **`IntuneConfigBackup.Copy`**: for writing.
   - *Application permissions*: the `…ReadWrite.All` permissions of `IntuneConfigBackup.Restore` ([Setup](02-setup.md#intuneconfigbackuprestore-writes-to-intune)) and `Group.Read.All`, *Grant admin consent*.
   - *Federated credentials*: Issuer and Subject identifier of `icb-restore`, name `ado-icb-restore`.

### 2. The replica in the variable group

*Pipelines → Library → intune-config-backup*: the variable **`REPLICA_TARGETS`**, the list of replicas as JSON, one entry per replica:

```json
[
  { "name": "DEV", "tenantId": "<Directory ID of DEV>", "readerClientId": "<client ID of IntuneConfigBackup.CopyReader>", "writerClientId": "<client ID of IntuneConfigBackup.Copy>" },
  { "name": "TEST", "tenantId": "<…>", "readerClientId": "<…>", "writerClientId": "<…>" }
]
```

A name has letters, digits, `-` and `_` only. Not a secret: these are IDs.

### 3. An environment per replica

*Pipelines → Environments → New environment* **`intune-replica-DEV`**, description *Copies into the tenant DEV wait here for an approval*, *Approvals and checks → Approvals*: who may write into DEV (each replica can have its own). On the first run, permit the pipeline to use it.

The web UI shows *Replicas* to people with `Backup.Admin` as soon as `REPLICA_TARGETS` has an entry, with a switch between several replicas; nothing to deploy. Another replica later: its two app registrations, one entry in the list, one environment.

## In Azure DevOps

*Pipelines → IntuneConfigBackup → icb-replicate → Run pipeline*: the name of the replica, *Write into the replica* off for the plan only; *Only these types* to copy some types (comma-separated keys, e.g. `settingsCatalog,assignmentFilters`; `-`: all).

On the command line, signed in to DEV:

```bash
az login --tenant <replica tenant>
npx tsx src/cli.ts replicate --repo backup                    # plan
npx tsx src/cli.ts replicate --repo backup --types scripts --apply
```

## Good to know

- The source is the newest backup of PROD: a change made in PROD a minute ago is copied after the next backup (*Back Up Now*).
- Tested against a simulated Graph with two tenants; the first real copy should go into an empty or throwaway tenant.
- A replica's own backup, if it has one, records the copy under `IntuneConfigBackup.Copy (app)`.
