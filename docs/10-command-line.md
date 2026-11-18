# 10 Command line

Everything the pipelines do also runs on a workstation. Node.js 24 and Git are needed.

```bash
npm ci
az login --tenant <tenant>   # or any other sign-in DefaultAzureCredential knows
```

## Backup

```bash
npm run backup -- --out backup   # writes ./backup
```

| Option | |
|---|---|
| `--out <folder>` | backup folder, default `./backup` |
| `--commit` | commit the changes in that folder (it must be a Git repository) |
| `--push` | push the commits |
| `--no-audit` | one commit for all changes, without reading the Intune audit log |
| `--include-oma-values` | also back up encrypted OMA-URI values in plain text ([Backup Format](06-backup-format.md)) |
| `--types <list>` | only some types: see `--help` |

Exit code 0: everything backed up; 1: at least one type could not be read (the report says which permission is missing); 2: the run could not start.


## Restore

```bash
npx tsx src/cli.ts restore <object ID> --repo backup                     # dry run: what would change
npx tsx src/cli.ts restore <object ID> --repo backup --version 1a2b3c4d  # dry run of that version
npx tsx src/cli.ts restore <object ID> --repo backup --apply             # write it, with an identity that may write
npx tsx src/cli.ts restore <ID>@<version> <ID>@<version> --repo backup  # several objects in one go
```

Details, and what restore refuses: [Restore](04-restore.md).

## Web UI on your own computer

```powershell
npm run build
node dist/serve.js --repo ..\backup   # a clone of intune-config-backup-data; http://127.0.0.1:4300/
```

Without sign-in the server answers only on this computer. Options and sign-in: [Web UI](03-web-ui.md).

## Development

```bash
npm run typecheck
npm test
npm run build        # server and web UI into dist/
npm run build:demo   # the public demo with sample data into dist/demo/
```

Node.js 24, TypeScript, Fastify and React, `@azure/identity` for the sign-in, plain `fetch` for Microsoft Graph, the `git` CLI for the history.
