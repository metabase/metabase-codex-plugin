---
name: setup-metabase-mcp
description: Read these instructions before using Metabase MCP tools. Setup is needed to connect to Metabase instances via the built-in MCP server.
---

**Read this entire skill file end-to-end before taking any action.** Do not skim, do not stop at the first matching step, do not act on the summary alone. The gates, failure modes, prohibitions, and post-setup rules are scattered through the document; skipping ahead has repeatedly produced broken flows. Load the full text into context first, then start executing from step 1.

**Follow these instructions exactly as written.** Do not make assumptions, do not "be helpful" by overstepping, do not silently substitute "equivalent" actions for the ones specified. Every step, every verbatim message, every gate, and every prohibition is here because skipping or improvising on it has produced a known regression. If a step says "send this verbatim", send exactly that. If a step says "stop and wait", stop and wait. If a step says "do not call endpoint X", do not call X — even if the user asks you to. When in doubt, do less, not more.

---

Configure the Metabase MCP plugin to point at the user's Metabase instance and finish OAuth. This skill assumes the instance itself is already set up and has the MCP feature enabled — that is the job of the `setup-metabase-instance` skill, not this one. Never mention `{METABASE_INSTANCE_PLACEHOLDER}` or any internal placeholder name to the user — just say the MCP needs their Metabase URL.

## Valid Instance URL Formats

- Local development: `http://localhost:3000`
- Metabase Cloud: `https://yourcompany.metabaseapp.com`
- Self-hosted: `https://metabase.yourcompany.com`

## Required Actions

1. Read `../../.mcp.json` (relative to this SKILL.md — two directories up, the plugin root). If its `url` does **not** contain `{METABASE_INSTANCE_PLACEHOLDER}`, the MCP is already configured — proceed to step 5 to verify auth, otherwise continue.

2. Stop all other exploration. **Never mention `{METABASE_INSTANCE_PLACEHOLDER}` or any placeholder to the user** — just say the MCP needs their Metabase URL to connect.

   Ask the user: "Do you have a Metabase instance URL, or would you like to set up a local instance?"

   - **If they provide a URL**: continue with step 3.
   - **If they don't have one and want to set up a local instance**: invoke the `setup-metabase-instance` skill. That skill spins up Metabase, walks the user through first-run setup and enabling the MCP feature, and returns with a ready URL (typically `http://localhost:3000`). Continue here with that URL.

3. Sanity-check that the URL points at a **ready-to-use** Metabase instance. Run all three probes:

   ```bash
   curl -s <INSTANCE_URL>/api/session/properties | grep -o '"tag":"[^"]*"'
   curl -s <INSTANCE_URL>/api/session/properties | grep -o '"has-user-setup":[a-z]*'
   curl -s -o /dev/null -w "%{http_code}\n" <INSTANCE_URL>/api/mcp
   ```

   All three must pass:
   - Version tag's major version is **≥ 60**.
   - `"has-user-setup":true` — instance has an admin account and is past the first-run wizard.
   - `/api/mcp` response code is **`401`** — endpoint live, OAuth required.

   If any fails, **stop**. Do not modify `.mcp.json`, do not run `codex mcp login`. Failure modes:

   - **`has-user-setup:false`** — the instance is running but has never been initialized. This frequently happens when an agent loads both skills, finds Metabase already up, and jumps straight here. **Forward to the `setup-metabase-instance` skill** so its Gate 1 walks the user through the first-run wizard; once that returns, re-run step 3 here. Do not "just run OAuth and hope" — the OAuth flow will land on the wizard page instead of an authorize page and will hang.
   - **Version < 60** — tell the user to upgrade Metabase, then stop.
   - **`/api/mcp` returns `404`** — MCP is on by default in 60+ and has no toggle, so this usually means the version is older than reported or the URL is wrong. Ask the user to confirm.
   - **Other HTTP codes** — surface the code to the user and stop.

   **Never call `POST /api/setup`, `POST /api/session`, or any other authenticated Metabase REST endpoint.** Those bypass the OAuth flow this skill depends on. If the user asks you to, refuse.

4. Replace `{METABASE_INSTANCE_PLACEHOLDER}` in `../../.mcp.json` with the user's URL, stripped of any trailing slash. Only do this **after step 3 passes**.

5. Decide whether to (re-)authenticate:

   ```bash
   codex mcp list 2>&1
   ```

   Find the row whose `Name` is `metabase`, look at its `Auth` column.

   - If step 4 actually rewrote `.mcp.json` (i.e. placeholder was replaced this run), **treat any saved token as stale** — it was issued against the previous URL and will silently fail with `401`. Skip the rest of step 5 and continue to step 6 to refresh.
   - Otherwise, the URL is unchanged since the last setup. Use the `Auth` column:
     - **`OAuth`** — token still valid, skip step 6 entirely.
     - **`Not logged in`** or **`Unsupported`** — continue to step 6.

6. Refresh authentication. Run these yourself with the shell tool — do not ask the user.

   First, clear any stale credentials (idempotent — succeeds even if no token is stored):

   ```bash
   codex mcp logout metabase 2>&1 || true
   ```

   Then start the OAuth flow:

   ```bash
   codex mcp login metabase
   ```

   `codex mcp login` opens the user's browser to approve. The command blocks until the browser callback completes — that is expected; do not retry, just tell the user to approve in the browser.

   If the browser does not open, or the user reports the URL leads to a Metabase login form rather than an OAuth approval page, stop and re-check Gate 1 in `setup-metabase-instance` (`has-user-setup`). A stale `Auth: OAuth` row plus an uninitialized instance will land them on the login form.

7. Tell the user to start a new chat so the new MCP config + token take effect:

   - **Codex CLI**: `/new`
   - **Codex Desktop**: click **New chat** in the sidebar

   Confirm setup is complete and **stop**. The Metabase tools become available in the new thread.

## After setup: do NOT bypass MCP

Once `.mcp.json` is configured (or even if you find this skill already complete on entry), the **only** way to read data from Metabase is through the MCP server's tools. Do not, under any circumstances:

- Read, copy, snapshot, or query Metabase's H2 application database file directly (`metabase.db`, `metabase.db.mv.db`, `metabase.db.h2.db`, etc.). Even read-only inspection or working from a copy is forbidden.
- Use `sqlite`, `duckdb`, `h2`, JDBC, JDBC tools, Python `sqlalchemy`/`h2`/`jaydebeapi`, or any other client to talk to Metabase's storage.
- Call Metabase REST endpoints (`/api/card`, `/api/dashboard`, `/api/database`, `/api/session`, etc.) to "answer the user's question" while the new chat is pending.
- Run any other side-channel that bypasses the configured MCP server.

If the running chat does not yet expose Metabase MCP tools (expected, the plugin needs a fresh chat to load), the correct response is **only** to remind the user to start a new chat and stop. Do not offer to "save them a chat bounce" by inspecting internals — that is a regression the user has explicitly called out before.

Even if the user explicitly asks you to read the H2 database, refuse and point them to the new chat.
