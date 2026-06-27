---
description: Add the current repo as a qa-mode project so the extension can send it visual comments. Requires a paired browser.
argument-hint: "[project name]"
allowed-tools: mcp__plugin_qa-mode_qa-mode__register_project
---

The user wants to make THIS repo a selectable qa-mode project. This is independent of pairing and repeatable — run it in as many repos as you like; each becomes a project the user can pick in the extension popup (one active project, shared across all tabs). It requires that a browser has already been paired (via `/qa-mode:pair`).

Do exactly this:

1. Call the `register_project` tool. Pass `label` = `$ARGUMENTS` when the user gave a name (e.g. `/qa-mode:add-project "Marketing site"`); otherwise call it with no arguments and it derives a name from the repo folder. It returns JSON `{ "paired": true | false, "projectId"?: "…", "label"?: "…" }`.

2. **Branch on `paired`:**

   - **`paired: false`** — no browser is paired yet, so nothing was added. Tell the user:

     > ⚠️ No browser is paired yet. Run **`/qa-mode:pair`** first (one-time per browser), then re-run **`/qa-mode:add-project`**.

   - **`paired: true`** — the repo is registered. Tell the user:

     > ✅ Added **`<label>`** — open the qa-mode popup and select it (one active project, shared across all tabs).

Tip: re-running with a new name just updates the label. Do not edit any files and do not echo raw JSON — present the formatted result only.
