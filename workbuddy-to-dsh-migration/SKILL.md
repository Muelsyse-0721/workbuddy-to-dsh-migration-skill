---
name: workbuddy-to-dsh-migration
description: "Stable migration path from WorkBuddy to DeepSeek Harness (DSH) using dsh-plugin-migration-workbuddy. Use when: (1) moving WorkBuddy sessions/memories/MCP/automations into DSH, (2) setting up or debugging the migration plugin, (3) patching the plugin for DSH 0.2.x compatibility, (4) running probe/preview/migrate via HTTP API. Covers install, peer-version exemption, config wiring bug, large-file handling, and verification."
---

# WorkBuddy → DSH Migration (stable path)

## TL;DR
- Plugin: `dingguangyi0/dsh-plugin-migration-workbuddy` (GitHub only, NOT on npm)
- Install from **Release tarball**, not `github:` (avoids build-authorization hell)
- DSH 0.2.0-rc.2 requires a peer-version exemption (plugin pins 0.1.2-rc.1)
- **Upstream bug**: `lib/index.js` never passes `maxFileBytes`/`maxFiles` to the service — main WorkBuddy sessions are ~37 MB and get silently skipped by the 10 MB default. Patch locally.
- UI settings section may not render; the HTTP API (`/api/migration/workbuddy/*`) is the reliable driver.

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

## Required local patch (upstream v0.1.0 bug)

File: `~/.dsh/profiles/<name>/node_modules/dsh-plugin-migration-workbuddy/lib/index.js`

```js
// before
const service = new MigrationService(ctx.sessionPersistence, ctx.get('workspaceRegistry'), {
    toolMode: config.toolMode,
    automationModelId: config.automationModelId,
    automationModelAllowlist: config.automationModelAllowlist,
    ...
});

// after
const service = new MigrationService(ctx.sessionPersistence, ctx.get('workspaceRegistry'), {
    toolMode: config.toolMode,
    maxFileBytes: config.maxFileBytes,
    maxFiles: config.maxFiles,
    automationModelId: config.automationModelId,
    automationModelAllowlist: config.automationModelAllowlist,
    ...
});
```

Keep `lib/index.js.orig` for rollback. Also raise the limit in profile patch:

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
4. Run returns `failedCount: 0`; inspect `lossyFields` for dropped tool-result payloads.
5. DSH side: imported sessions appear under the target workspace; memories in `migration/memory/records.jsonl`.

## Known gotchas

- Plugin is **GitHub-only** (not on npm); use Release tarball to avoid pnpm build-authorization.
- Peer deps pin `0.1.2-rc.1` vs DSH `0.2.0-rc.2` → exemption required or install aborts.
- `toolMode: 'transcript'` keeps raw tool calls; `'narrative'` summarizes (lossy).
- Automations need `automationModelId`/`allowlist` configured or all report ineligible.
- MCP secrets are never copied; re-enter credentials post-migration.
- UI "Migration & backup" section may not render on 0.2.0-rc.2; HTTP API is authoritative.
- Source is read-only; WorkBuddy data untouched.

## Rollback

```bash
dsh plugin --profile <name> remove dsh-plugin-migration-workbuddy
# restore profile from backup if needed:
# ~/.dsh/profiles/<name>.bak-<ts>
```
