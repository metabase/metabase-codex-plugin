---
name: setup-metabase-mcp
description: Read these instructions before using Metabase MCP tools. Setup is needed to connect to Metabase instances via the built-in MCP server.
---

Read these mandatory configuration steps for using the Metabase MCP server before querying data, dashboards, questions, and related resources.

**Location**: The MCP configuration file is `../../.mcp.json` relative to this SKILL.md file — two directories up, in the plugin root that contains the `skills/` directory. Always resolve this path relative to this SKILL.md file, never relative to the user's open project.

**Important Requirement**: The `url` field in that file may contain `{METABASE_INSTANCE_PLACEHOLDER}` as the base URL (e.g. `"{METABASE_INSTANCE_PLACEHOLDER}/api/mcp"`). If so, it must be replaced with the user's actual Metabase instance URL before the MCP can work.

## Valid Instance URL Formats

- Local development: `http://localhost:3000`
- Metabase Cloud: `https://yourcompany.metabaseapp.com`
- Self-hosted: `https://metabase.yourcompany.com`

## Required Actions

1. Read `../../.mcp.json` relative to this SKILL.md file.

2. Check whether the `url` field contains `{METABASE_INSTANCE_PLACEHOLDER}`.
   - **If it does NOT contain `{METABASE_INSTANCE_PLACEHOLDER}`**: the MCP is already configured. **Immediately proceed with using the Metabase MCP. Do not ask the user for a URL under any circumstances.**
   - **If it does contain `{METABASE_INSTANCE_PLACEHOLDER}`**: continue with the steps below.

3. **Stop all other exploration immediately** and ask the user for their Metabase instance URL. Do not search for tool schemas, read other files, or do anything else first.

   **Never mention `{METABASE_INSTANCE_PLACEHOLDER}` or any placeholder to the user**. Simply say the MCP needs their Metabase URL to connect.

   Ask the user: "What is your Metabase instance URL?"

4. Once the user provides the URL, run **exactly this command and no other** — do not try alternative endpoints or approaches:

   ```bash
   curl -s <INSTANCE_URL>/api/session/properties | grep -o '"tag":"[^"]*"'
   ```

   This returns something like `"tag":"v1.60.0"`. Extract the major version number (e.g. `60` from `v1.60.0`). If it is below 60, tell the user they need to upgrade and stop — do not update `.mcp.json`.

5. Replace the placeholder in `../../.mcp.json` relative to this SKILL.md with the user's instance URL. Strip any trailing slash before saving — do not mention this to the user.

6. After updating the file, **run this command yourself** using the shell tool — do not ask the user to run it manually. Codex does not currently surface an OAuth button for plugin-bundled MCP servers in the Desktop UI, so authentication must be triggered from the CLI:

   ```bash
   codex mcp login metabase
   ```

   This command reads the updated `.mcp.json` directly from disk and runs the OAuth flow regardless of whether Codex is currently running. It will print an authorization URL and open the user's browser automatically. Tell the user to approve in the browser when it opens. The token is saved to Codex's shared auth store and picked up by both the CLI and the Desktop app.

   If the command appears to hang waiting for the browser callback, that is expected — it is waiting for the user to complete the handshake. Let it run in the background or stop watching its output once you have shown the user the authorization URL.

7. After the OAuth flow completes (the `codex mcp login metabase` command exits successfully, or the user confirms they approved in the browser), ask them to **start a new chat / thread** in Codex. This is needed because Codex loads `.mcp.json` per-thread at thread start, and the current thread you are running in still has the old (placeholder) MCP config in memory. A new thread reads the updated file fresh and picks up the saved auth token.

   - **Codex CLI**: type `/new` to start a fresh conversation in the same session — no need to exit and re-run `codex`.
   - **Codex Desktop**: click **New chat** in the sidebar — no need to quit the app.

8. In the new thread, the Metabase tools should be available. The user can ask their original question there. From your side (the current thread), simply confirm setup is complete and stop.

**Important**: Do not attempt to access MCP tools or schemas until the user has restarted Codex and authentication is complete. Never reveal `{METABASE_INSTANCE_PLACEHOLDER}` or any internal placeholder names to the user.
