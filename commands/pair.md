---
description: Pair this browser with qa-mode (one-time per browser), then optionally add the current repo as a project.
argument-hint: "[project name]"
allowed-tools: mcp__plugin_qa-mode_qa-mode__start_pairing, mcp__plugin_qa-mode_qa-mode__wait_for_pairing, mcp__plugin_qa-mode_qa-mode__register_project
---

The user wants to pair THIS browser with qa-mode. Pairing is a one-time, machine-wide handshake that authorizes the extension to talk to your coding agent — it is separate from adding projects. After pairing succeeds, OFFER to add the current repo as a project (a repeatable step that also has its own command, `/qa-mode:add-project`). Do not add the project automatically — ask first.

Do exactly this:

1. Call `start_pairing` (no arguments). It returns JSON `{ "code": "########", "expiresInSeconds": N }`. Print the code **prominently**, split into two four-digit groups (e.g. `4827 1593`):

   > 🔗 **Pairing code: `4827 1593`** — valid ~`N` seconds, single use
   >
   > Open the **qa-mode extension popup**, find **Pair browser**, and type this code in.

2. Immediately call `wait_for_pairing` (no arguments). It blocks until the browser redeems the code, returning JSON `{ "paired": true | false }`. **Branch on it:**

   - **`paired: true`** — pairing succeeded. Tell the user:

     > ✅ Browser paired.

     Then ASK (do not assume): **"Add this repo as a qa-mode project now? (yes / no)"** — if the user passed a name in `$ARGUMENTS`, note you'll use it as the label.

       - **yes** → call `register_project` (pass `label` = `$ARGUMENTS` when provided, else no arguments). On `{ "paired": true, "label": "…" }`, tell the user:

         > ✅ Added **`<label>`** — open the qa-mode popup and select it (one active project, shared across all tabs).

       - **no** → tell them they can add this (or any) repo anytime with **`/qa-mode:add-project`**.

   - **`paired: false`** — the code expired before it was entered (or `wait_for_pairing` timed out). Tell the user and offer to run **`/qa-mode:pair`** again for a fresh code.

Do not edit any files and do not echo raw JSON — present the formatted result and instructions only.
