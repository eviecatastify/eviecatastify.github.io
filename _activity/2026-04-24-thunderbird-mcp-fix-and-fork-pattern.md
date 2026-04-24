---
title: "Thunderbird MCP Race Condition Fixed — Downstream Fork Pattern Established"
date: 2026-04-24
category: infrastructure
outcome: "Thunderbird MCP now connects reliably at Claude Code startup. Local patch survives upstream updates automatically via a GitHub Action."
tags: [thunderbird, mcp, git, github-actions, debugging, automation]
status: completed
description: "Diagnosed a startup race condition in the Thunderbird MCP bridge, applied two fixes, then established a downstream fork pattern to protect the fix from being overwritten by future upstream pulls."
---

## What I did

Investigated why the `thunderbird-mail` MCP server was consistently unavailable at Claude Code startup, requiring a manual `/exit` and restart to connect. Diagnosed the root cause, applied two fixes, and set up a sustainable process for maintaining the fix against upstream updates.

## Root cause

The Thunderbird MCP uses two components:

- **Thunderbird extension** — starts an HTTP server on a random port and writes a connection file at `~/Downloads/thunderbird.tmp/thunderbird-mcp/connection.json` containing the port and auth token.
- **`mcp-bridge.cjs`** — a Node.js process Claude Code spawns at startup. It reads the connection file and forwards tool calls to Thunderbird over HTTP.

When Claude Code starts, it calls `tools/list` to discover available tools. This call is forwarded by the bridge to Thunderbird. If the connection file does not exist at that moment — because Thunderbird is still starting, including any time spent on a Proton Bridge reconnection retry — the bridge waited only **5 seconds** before failing.

Claude Code caches the result of `tools/list` and never re-queries it during a session. So any failure at startup meant zero Thunderbird tools for the entire session.

## Fix 1: increase the retry window

In `~/src/work/thunderbird-mcp/mcp-bridge.cjs`, changed:

```js
const CONNECTION_MAX_RETRIES = 5;   // 5 seconds
```
to:
```js
const CONNECTION_MAX_RETRIES = 90;  // 90 seconds
```

This covers the full window of a slow Thunderbird startup including the Proton Bridge retry.

## Fix 2: council digest script waits rather than skips

The existing check in `~/.local/bin/council-digest.sh` immediately skipped if the connection file was absent. Changed to poll every 10 seconds for up to 3 minutes before skipping. The log now records how long it waited, which will help identify if the window ever needs adjusting.

## Downstream fork pattern

`mcp-bridge.cjs` lives in a cloned upstream repository (`TKasperczyk/thunderbird-mcp`). A `git pull` would overwrite the fix. To protect it permanently:

1. Forked the upstream repo to `callenb/thunderbird-mcp` on GitHub.
2. Renamed the local `origin` remote to `upstream`; pointed `origin` at the fork.
3. Created a `local` branch with two commits on top of upstream's `main`:
   - The 90-second fix
   - A GitHub Actions workflow for automated sync
4. Pointed the local `main` branch at `upstream/main` so `git pull` on `main` fetches from upstream, not the fork.
5. Pushed `local` to the fork.

## GitHub Actions sync workflow

A workflow at `.github/workflows/sync-upstream.yml` runs daily at 06:00 UTC and on manual trigger. It:

- Checks whether upstream has new commits; exits silently if not.
- Fetches `upstream/main` and rebases `local` on top.
- Force-pushes the updated `local` branch to the fork (using `--force-with-lease`).
- If the rebase fails due to a conflict, opens a GitHub issue on `callenb/thunderbird-mcp` with exact resolution commands. Does not open a duplicate if one is already open.

Write permissions for the workflow were enabled under **Settings → Actions → General → Workflow permissions**.

## Files changed

| File | Change |
|---|---|
| `~/src/work/thunderbird-mcp/mcp-bridge.cjs` | `CONNECTION_MAX_RETRIES`: 5 → 90 |
| `~/.local/bin/council-digest.sh` | Connection file check: immediate skip → wait up to 180s |
| `~/src/work/thunderbird-mcp/.github/workflows/sync-upstream.yml` | New — daily upstream rebase Action |

## Knowledge captured

A reference note explaining the problem, the reasoning, and the full implementation steps was written to:

```
~/vaults/second-brain/00_Inbox/2026-04-24 - Maintaining local patches on upstream Git repositories.md
```
