---
name: setup-metabase-mcp
description: Read these instructions before using Metabase MCP tools. Setup is needed to connect to Metabase instances via the built-in MCP server.
---

Configure the Metabase MCP server before querying data, dashboards, questions, and related resources. Never mention `{METABASE_INSTANCE_PLACEHOLDER}` or any internal placeholder name to the user — just say the MCP needs their Metabase URL.

## Valid Instance URL Formats

- Local development: `http://localhost:3000`
- Metabase Cloud: `https://yourcompany.metabaseapp.com`
- Self-hosted: `https://metabase.yourcompany.com`

## Required Actions

1. Read `../../.mcp.json` (relative to this SKILL.md — two directories up, the plugin root). If its `url` does **not** contain `{METABASE_INSTANCE_PLACEHOLDER}`, the MCP is already configured — proceed to step 6 to verify auth, otherwise continue.

2. Stop all other exploration. **Never mention `{METABASE_INSTANCE_PLACEHOLDER}` or any placeholder to the user** — just say the MCP needs their Metabase URL to connect.

   Ask the user: "Do you have a Metabase instance URL, or would you like to set up a local instance?"

   - **If they provide a URL**: continue with step 3.
   - **If they don't have one and want to set up a local instance**: invoke the `setup-metabase-instance` skill. Once the local instance is running, return here and use `http://localhost:3000` (or whatever port was chosen) as the instance URL.

3. Verify the version with this exact command:

   ```bash
   curl -s <INSTANCE_URL>/api/session/properties | grep -o '"tag":"[^"]*"'
   ```

   It returns something like `"tag":"v1.60.0"`. If the major version is below 60, tell the user to upgrade and stop.

4. Replace `{METABASE_INSTANCE_PLACEHOLDER}` in `../../.mcp.json` with the user's URL, stripped of any trailing slash.

5. Check current auth status:

   ```bash
   codex mcp list 2>&1
   ```

   Find the row whose `Name` is `metabase`, look at its `Auth` column.

   - **`OAuth`** — already authorized, skip step 6.
   - **`Not logged in`** or **`Unsupported`** — continue to step 6.

6. Run this yourself with the shell tool — do not ask the user to run it. Codex Desktop has no OAuth button for plugin-bundled MCP servers, so login must come from the CLI:

   ```bash
   codex mcp login metabase
   ```

   It opens the user's browser to approve. The command blocks until the browser callback completes — that is expected; do not retry, just tell the user to approve in the browser.

7. Tell the user to start a new chat so the new MCP config + token take effect:

   - **Codex CLI**: `/new`
   - **Codex Desktop**: click **New chat** in the sidebar

   Confirm setup is complete and stop. The Metabase tools become available in the new thread.
