# 08 More tenants

One backup run can back up several tenants, for example DEV and TEST beside PROD. Each tenant gets its own branch `backup-<name>`, its own history with who changed what, its own documentation and Teams cards with its name. The web UI switches between them with the menu at the left of the toolbar.

Restore, approvals and replicas stay with the main tenant (the one of the service connections). Another tenant is read only in the web UI.

## 1. Reader app in the other tenant

In the other tenant, create an app registration like `IntuneConfigBackup.Reader` ([Setup](02-setup.md#intuneconfigbackupreader-reads-intune)):

- the same read permissions, with admin consent ([Permissions](07-permissions.md));
- *Certificates & secrets → Federated credentials → Add credential → Other issuer*: the **Issuer** and **Subject identifier** of the service connection `icb-reader` (*Project settings → Service connections → icb-reader → Edit*), name `ado-icb-reader`.

The app trusts the token of that service connection; no secret is stored.

## 2. Add it on the Settings page

*Settings → Tenants → More Tenants*, the three fields below the list, then *Add* (saved at once; the cross next to a tenant removes it):

| Field | Value |
|---|---|
| Name | short, letters, digits, `-` and `_` (`DEV`); it names the branch `backup-DEV` |
| Tenant ID | Directory (tenant) ID of the other tenant |
| Client ID | Application (client) ID of its reader app |

*This Tenant* names the main tenant in the menu (default *Main*).

## 3. First run

The next run of `icb-backup` has one job per tenant (`main`, `DEV`). A tenant whose job fails does not stop the others. The first run of a new tenant creates its branch and reads its audit log a year back, like the first backup of the main tenant.

The web UI clones the branch of a tenant the first time someone opens it, then pulls it with the main one. Before its first backup, the page says the tenant has no backup yet.

## Limits

- Up to 20 tenants; each job takes about two minutes, one after the other on the one agent.
- The settings (types, names left out, schedule, Teams) are the same for all tenants.
- To remove a tenant: *Remove* on the Settings page. Its branch stays; delete it in *Repos → Branches* when it is no longer needed.
