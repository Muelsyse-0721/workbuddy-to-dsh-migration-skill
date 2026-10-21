---
name: workbuddy-to-dsh-migration
description: "Stable migration path from WorkBuddy to DeepSeek Harness (DSH) using dsh-plugin-migration-workbuddy. Use when: (1) moving WorkBuddy sessions/memories/MCP/automations into DSH, (2) setting up or debugging the migration plugin, (3) patching the plugin for DSH 0.2.x compatibility, (4) running probe/preview/migrate via HTTP API. Covers install, peer-version exemption, config wiring bug, large-file handling, and verification."
---

# WorkBuddy → DSH Migration (stable path)

## TL;DR
- Plugin: `dingguangyi0/dsh-plugin-migration-workbuddy` (GitHub only, NOT on npm)
- Install from **Release tarball**, not `github:` (avoids build-authorization hell)
- DSH 0.2.0-rc.2 requires a peer-version exemption (plugin pins 0.1.2-rc.1)
- **3 upstream bugs must be patched** (see below): config not wired, persistence API drift, retired v3 source kind. With all three applied: 28/28 sessions imported, 0 errors.
- UI settings section may not render; the HTTP API (`/api/migration/workbuddy/*`) is the reliable driver.
- Test patches in a throwaway profile first (`dsh <name> --from-default-profile web`) instead of restarting the live app.

## Install (exact sequence)

```bash
# 1. Exempt peer version (plugin pins 0.1.2-rc.1, DSH is 0.2.0-rc.2)
dsh plugin --profile <name> allow-version dsh-plugin-migration-workbuddy@0.1.0 \
  --dsh-version 0.2.0-rc.2 --accept-risk

# 2. Install from Release tarball (no build authorization needed)
dsh plugin --profile <name> add \
  https://github.com/dingguangyi0/dsh-plugin-migration-workbuddy/releases/download/v0.1.0/dsh-plugin-migration-workbuddy-0.1.0.tgz
```

Restart DSH after install.


## Required local patches (3 upstream bugs on DSH 0.2.x)

Plugin v0.1.0 targets DSH 0.1.2-rc.1. On DSH 0.2.x it needs three patches.
All three live in `~/.dsh/profiles/<name>/node_modules/dsh-plugin-migration-workbuddy/lib/`.

### Patch 1 - index.js: config never reaches the service

`maxFileBytes`/`maxFiles` are parsed by `config.js` and honored by `service.js`,
but `index.js` never passes them into `MigrationService`. Without this the scan
always uses the 10 MB default and silently skips large main sessions (a 37 MB
session produced `file-too-large` and was never migrated).

```js
const service = new MigrationService(ctx.sessionPersistence, ctx.get('workspaceRegistry'), {
    toolMode: config.toolMode,
    maxFileBytes: config.maxFileBytes,   // add
    maxFiles: config.maxFiles,           // add
    automationModelId: config.automationModelId,
    automationModelAllowlist: config.automationModelAllowlist,
});
```

### Patch 2 - service.js: persistence API drift

DSH 0.1.x: `create(header, inheritedEventCount)` + `append(id, events)`.
DSH 0.2.x: `create(header, { inheritedEventCount })` returns a write HANDLE
that owns `append`/`flush`/`close`. The old call throws
`seeded session header requires an inherited event count`, and EVERY session
fails to import (0 imported, N errors).

```js
                        const handle = await this.persistence.create(header, { inheritedEventCount: Number(converted.events.length) });
try {
    await handle.append(converted.events);
    await handle.flush();
}
finally {
    await handle.close();
}
```

For a version-robust variant that also works on DSH 0.1.x, catch the
`inherited event count` error and retry with the positional argument, then
branch on whether the returned value has an `append` method.

### Patch 3 - converter.js: retired v3 plugin source kind

DSH session format v4 refuses `{ kind: 'plugin', plugin: ... }` message sources
(`format v4 message requires a producer-owned source kind`). The WorkBuddy
context-envelope message uses exactly that retired shape, so the sessions that
carry it still fail after Patch 2.

```js
// before
source: { kind: 'plugin', plugin: 'WorkBuddy 上下文', form: 'snapshot', sections: input.contextSections },
// after
source: { kind: 'plugin:workbuddy-context', form: 'snapshot', sections: input.contextSections },
```

`plugin:<name>` is the producer-owned kind shape the v4 codec accepts.

### Profile config (required with Patch 1)

```yaml
# ~/.dsh/profiles/<name>/cordis.patch.yml
- id: migration-workbuddy
  name: dsh-plugin-migration-workbuddy
  config:
    enabled: true
    maxFileBytes: 67108864   # 64 MB; schema max 100 MB
    maxFiles: 20000
    toolMode: transcript
```

Keep `.orig` copies of each patched file for rollback.

## HTTP API (reliable driver, bypass UI)

Base: `http://127.0.0.1:<port>` (port from `lsof -nP -iTCP -sTCP:LISTEN -p <dsh_pid>`).

```
GET  /api/migration/workbuddy/probe     # scan report
GET  /api/migration/workbuddy/preview   # full preview (sessions/workspaces/memories/mcp/automations)
GET  /api/migration/workbuddy/manifest  # previous import manifest
POST /api/migration/workbuddy/run        # body: {confirmed:true, sessions, memories, mcp, automations, force,
                                        #         sessionKeys?, workspaceIds?, automationIds?}
POST /api/migration/workbuddy/archive   # {manifestId?}
```

Mutating endpoints require `Origin: http://127.0.0.1:<port>`.

## Migration run (example)

```bash
PORT=19387
# pick workspace ids from preview's workspaces[].id
curl -sS -X POST "http://127.0.0.1:$PORT/api/migration/workbuddy/run" \
  -H "Origin: http://127.0.0.1:$PORT" -H "Content-Type: application/json" \
  -d '{"confirmed":true,"sessions":true,"memories":true,"mcp":true,"automations":true,
       "workspaceIds":["<id>"]}'
```

Returns an audit report: imported/skipped/failed counts, lossy fields, manifest id.

## Verification checklist

1. `probe` returns `available:true`, `sessions > 0`.
2. No `file-too-large` warning for the main session jsonl (size check: `ls -la ~/.workbuddy/projects/.../<sid>.jsonl`).
3. Preview `summary.eligible` matches expected session count.
4. Run returns `importedSessions > 0` and `errors: []`. Partial success means a patch is missing:
   - all errors `seeded session header requires an inherited event count` -> Patch 2 missing
   - errors `format v4 message requires a producer-owned source kind` -> Patch 3 missing
   - `file-too-large` in preview warnings -> Patch 1 missing
5. DSH side: imported sessions appear under the target workspace. Verify via the
   `session/list` RPC (endpoint `session/list`, payload `{ args: { _request: {} } }`
   over `POST /api/session/list` with the browser auth cookie).

### Verification recipe (no restart needed)

```bash
# isolated profile
dsh migtest --from-default-profile web
dsh plugin --profile migtest allow-version dsh-plugin-migration-workbuddy@0.1.0 \
  --dsh-version 0.2.0-rc.2 --accept-risk
dsh plugin --profile migtest add <tarball-url>
# apply the 3 patches to ~/.dsh/profiles/migtest/node_modules/.../lib/
dsh --profile migtest --no-open
curl -sS -X POST http://127.0.0.1:3080/api/migration/workbuddy/run \
  -H 'Origin: http://127.0.0.1:3080' -H 'Content-Type: application/json' \
  -d '{"confirmed":true,"sessions":true,"memories":true,"mcp":true,"automations":true,
       "workspaceIds":["<id>"]}'
```

`importedSessions` must equal the preview's eligible count and `errors` must be empty.

## Known gotchas

- Plugin is **GitHub-only** (not on npm); use Release tarball to avoid pnpm build-authorization.
- Peer deps pin `0.1.2-rc.1` vs DSH `0.2.0-rc.2` → exemption required or install aborts.
- `toolMode: 'transcript'` keeps raw tool calls; `'narrative'` summarizes (lossy).
- Automations need `automationModelId`/`allowlist` configured or all report ineligible.
- MCP secrets are never copied; re-enter credentials post-migration.
- UI "Migration & backup" section may not render on 0.2.0-rc.2; HTTP API is authoritative.
- Source is read-only; WorkBuddy data untouched.
- Automations need an `automationImport` channel registered by a deployment plugin;
  without it, automations are reported ineligible (`automation-channel-unavailable`).
- MCP entries import only names/addresses/transport; credentials must be re-entered.
- Session `cwd` for `is_playground=1` sessions becomes `~/.dsh/workspaces/session-mode`;
  only real project sessions keep their original workspace path.

## Rollback

```bash
dsh plugin --profile <name> remove dsh-plugin-migration-workbuddy
# restore profile from backup if needed:
# ~/.dsh/profiles/<name>.bak-<ts>
```
