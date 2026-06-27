# qa-mode

The Claude Code half of **qa-mode** — visual web commenting → code edits.

You point at any element on a live web page (via the qa-mode browser extension), describe the change you want, and save. This plugin picks up each saved comment and dispatches a sub-agent to implement it in the repo Claude Code is open in — then reviews the diff against your intent and either resolves it or flags it for you.

> This repo is the **plugin** (the code side). It works together with the **qa-mode browser extension**, which is where you click and comment. You need both halves.

## Install (one-time per machine)

```bash
claude plugin marketplace add https://github.com/qa-mode-1/qa-mode-plugin
claude plugin install qa-mode@qa-mode
```

Then **restart Claude Code** so it launches the bundled MCP server. The plugin installs at user scope, so it's available in every repo on this machine.

## Pair your browser (one-time per machine)

The token that authorizes the browser is **machine-scoped and server-owned** — there's nothing to paste. You pair with a short code:

1. In Claude Code, run `/qa-mode:pair`. It prints an **8-digit code** (valid ~2 min, single use).
2. Open the qa-mode extension popup in your browser and type the code into the **Pair browser** field.
3. The popup flips to **paired ✓**. The machine is now trusted — you won't need a code again. `/qa-mode:pair` then **offers to add the current repo** as your first project; you can accept, or decline and add it later.

Under the hood: the broker generates `~/.qa-mode/token` on first boot, and `/qa-mode:pair` hands it to the browser over loopback once you enter the matching code. The browser stores it and bears it on every request thereafter. The broker also stamps `~/.qa-mode/paired-at` on each successful redeem — the trustworthy "a browser paired" signal that `/qa-mode:pair` waits on and `/qa-mode:add-project` checks. If a code lapses, just run `/qa-mode:pair` again.

## Add & switch projects

qa-mode is multi-project. **Pairing trusts the machine once; `/qa-mode:add-project` registers the repo it's run in** as a selectable project — it needs a paired browser, but never a code.

1. Run `/qa-mode:add-project` in any repo — name it with `/qa-mode:add-project "Marketing site"` or let it default to the repo folder. The project appears in the popup. (`/qa-mode:pair` offers this for the first repo right after pairing.)
2. In the extension popup, the **project list** shows everything you've added. Pick the **active project** — one global selection, shared across every browser tab — and saved comments route to it.
3. Comment as usual; the save routes to the active project, and `/qa-mode:feedback` in *that* repo picks it up.

Remove a project with the **×** next to it in the popup (hide-only — its comments stay on disk; re-adding restores it). The active-project selection lives in the browser; the registry of added projects lives at `~/.qa-mode/projects.json`.

## Use it

In Claude Code, in the repo you want edited:

```
/qa-mode:feedback
```

It long-polls for comments, batches near-simultaneous saves, dispatches one `comment-handler` sub-agent per comment (sequentially — they share one working tree), reviews each diff against the stated intent, then `resolve`s on a pass or flags `needs-reply` (leaving the edit in the tree, never auto-reverting) otherwise. It idle-polls and exits after ~10 minutes of quiet.

The server auto-creates `<repo>/.qa-mode/` for its SQLite DB + screenshots and drops a `.qa-mode/.gitignore` (`*`) so your repo never tracks it — zero setup.

## Troubleshooting

**"qa-mode disconnected — reload this page"** (shown in the overlay on save) — the browser extension was reloaded or updated (a Web Store update, or a dev rebuild) while a tab still had the old in-page overlay. Chrome won't re-inject content scripts into already-open tabs, so reload the page (⌘/Ctrl-R) to reconnect. If the popup badge dropped from **paired ✓**, run `/qa-mode:pair` again for a fresh code.

## What's in here

| File | Purpose |
|------|---------|
| `.claude-plugin/marketplace.json` | Single-plugin marketplace manifest (`source: "."`). |
| `.claude-plugin/plugin.json` | Plugin metadata (`name: qa-mode`, version). |
| `.mcp.json` | Registers the `qa-mode` MCP server (stdio). |
| `server/qa-mode-server.mjs` | **Bundled, self-contained MCP server** (generated — see below). |
| `commands/feedback.md` | The `/qa-mode:feedback` orchestrator command. |
| `commands/pair.md` | The `/qa-mode:pair` browser-pairing command. |
| `commands/add-project.md` | The `/qa-mode:add-project` command (paired-gated). |
| `agents/comment-handler.md` | The `comment-handler` sub-agent. |

Commands and agents are auto-discovered from `commands/` and `agents/`, so they are not listed in `plugin.json`.

## How the server runs

`.mcp.json` launches:

```
node --disable-warning=ExperimentalWarning ${CLAUDE_PLUGIN_ROOT}/server/qa-mode-server.mjs
```

`server/qa-mode-server.mjs` is a single esbuild bundle of the whole server — the MCP SDK, zod, and the shared schema are inlined; the only runtime dependency is Node's built-in `node:sqlite` (no native binary). Because it lives **inside** the plugin, it travels with it when Claude Code copies the plugin into its cache (`~/.claude/plugins/cache/qa-mode-plugin/qa-mode/<version>/`), so `${CLAUDE_PLUGIN_ROOT}/server/qa-mode-server.mjs` resolves wherever the plugin lands. (`--disable-warning=ExperimentalWarning` silences the one-line `node:sqlite` experimental notice.)

**One bundle, two roles.** The same file runs in either mode:

- **Session mode** (default, what `.mcp.json` launches over stdio) — serves Claude Code the MCP tools, reading/writing only *this* repo's `<repo>/.qa-mode/comments.db`. It does **not** bind the HTTP port; on startup it ensures a broker is running, spawning one detached if absent.
- **Broker mode** (`--broker`) — a single persistent, machine-global process that owns the loopback port `127.0.0.1:7842`, the token, the pairing handshake, and the project registry (`~/.qa-mode/projects.json`). It routes each incoming comment by its `projectId` → repoRoot → that repo's store. It's a singleton (a second `--broker` loses the port bind and exits) and idle-exits after ~10 min with no traffic and no live session.

This split is why the extension always has something to talk to (the broker outlives any one session) and why the popup shows the projects you paired rather than whichever session happened to grab the port. Because the broker and a session are different processes, `watch_comments` wakes by polling the repo DB's `MAX(updated_at)` rather than an in-process notifier.

`MCP_TOOL_TIMEOUT` is `120000` ms so the bounded ~25s `watch_comments` long-poll never trips Claude Code's default tool timeout.

## MCP tools (namespaced)

Claude Code namespaces these with the prefix `mcp__plugin_qa-mode_qa-mode__`:

- `…__get_pending_comments`
- `…__watch_comments`
- `…__resolve_comment`
- `…__reply_comment`
- `…__register_project` (backs `/qa-mode:add-project` and the add step of `/qa-mode:pair` — paired-gated: registers this repo only once a browser has paired)
- `…__start_pairing` (backs `/qa-mode:pair` — issues a pairing code)
- `…__wait_for_pairing` (backs `/qa-mode:pair` — blocks until the browser redeems the code)

## Security model

A comment is effectively a code-execution request, so every write to the broker is gated by the machine token — including `GET /projects` (the registry list) and `DELETE /projects/<id>` (unpair). `/pair` is the only unauthenticated endpoint (the browser has no token yet) — it's protected by a one-active-code, short-TTL, single-use, attempt-capped handshake, which is sufficient for the purely local CSRF threat model (a random web page POSTing to loopback). `/health` carries a `service: "qa-mode-broker"` marker so a session/extension can tell a real broker apart from an unrelated process holding the port.

## Maintainers: regenerating the bundle

This plugin is published from the qa-mode source workspace. After any server source change:

```bash
pnpm --filter @qa-mode/server build   # tsc + esbuild → server/qa-mode-server.mjs
# bump the version in .claude-plugin/plugin.json, then publish the minimal plugin tree.
```

If the plugin is already installed, `claude plugin update qa-mode@qa-mode` refreshes the cached snapshot.
