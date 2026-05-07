---
name: setup-metabase-instance
description: Set up and run a local Metabase instance. Downloads the JAR (if Java 21+ is available) or runs via Docker. Also handles stopping running instances.
---

# Set Up a Local Metabase Instance

This skill helps users run a local Metabase instance for development, testing, or exploration.

## Important: Network Access Required

This skill requires network access. All `curl`, `java`, and `docker` commands must be run outside the Codex sandbox. Request full network access or run outside the sandbox before attempting these commands. Do not run them inside the sandbox as they will fail.

## Prerequisites Check

Run these checks in order. Stop at the first successful path.

### 1. Check for Java 21+

Expand PATH first to avoid the macOS stub at `/usr/bin/java`:

```bash
export PATH="/opt/homebrew/opt/openjdk/bin:/opt/homebrew/opt/openjdk@21/bin:/opt/homebrew/bin:/usr/local/opt/openjdk/bin:/usr/local/opt/openjdk@21/bin:/usr/local/bin:$PATH"
java -version 2>&1 | head -1
```

If the output shows Java 21 or higher → use Section A (JAR). Keep this `PATH` export for all subsequent commands.
If not found or version < 21 → check Docker (step 2).

### 2. Check for Docker

```bash
docker --version 2>&1
```

**If Docker is available**: Use the Docker method (Section B).

**If neither Java 21+ nor Docker is available**: Direct the user to install Docker:

- macOS/Windows: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Linux: [Docker Engine](https://docs.docker.com/engine/install/)

Tell them to re-run this skill after installing Docker.

---

## Section A: JAR Method (Java 21+)

### A1. Check for existing Metabase directory

```bash
ls -la ./metabase 2>/dev/null
```

If `./metabase` exists and contains files, ask the user:

- "A `./metabase` directory already exists. Should I use it (preserving existing data) or remove it and start fresh?"

If the user wants to start fresh:

```bash
rm -rf ./metabase
```

### A2. Create directory and download JAR

```bash
mkdir -p ./metabase
```

Get the latest OSS release URL and download:

```bash
curl -sL -o ./metabase/metabase.jar https://downloads.metabase.com/latest/metabase.jar
```

Tell the user this may take a minute (the JAR is ~400MB).

### A3. Check if port 3000 is in use

```bash
lsof -i :3000 2>/dev/null | grep LISTEN
```

If the port is in use, ask the user:

- "Port 3000 is already in use. Would you like to use a different port?"
- Suggest port 3001, 3002, etc.

Store the chosen port as `$PORT` (default: 3000).

### A4. Start Metabase in the background

Use the same `PATH` as in the Java prerequisite step when the agent uses a fresh shell (prepend the macOS Homebrew line again if unsure). Set `JAVA_CMD=$(command -v java)` after that export so you invoke the same binary you version-checked.

Prefer `tmux` for the JAR method when available. It is more reliable in Codex Desktop than plain `nohup` because it keeps the long-running Java process attached to a durable local session instead of depending on shell job-control behavior.

```bash
export PATH="/opt/homebrew/opt/openjdk/bin:/opt/homebrew/opt/openjdk@21/bin:/opt/homebrew/bin:/usr/local/opt/openjdk/bin:/usr/local/opt/openjdk@21/bin:/usr/local/bin:$PATH"
JAVA_CMD=$(command -v java)
PORT=${PORT:-3000}

if command -v tmux >/dev/null 2>&1; then
  tmux kill-session -t metabase-local 2>/dev/null || true
  tmux new-session -d -s metabase-local -c "$(pwd)/metabase" \
    "MB_DB_FILE=./metabase.db MB_JETTY_PORT=$PORT '$JAVA_CMD' -jar metabase.jar >> metabase.log 2>&1"
  echo "tmux:metabase-local" > ./metabase/metabase.pid
else
  (
    cd ./metabase
    MB_DB_FILE=./metabase.db MB_JETTY_PORT=$PORT \
      "$JAVA_CMD" -jar metabase.jar > metabase.log 2>&1 < /dev/null &
    echo $! > metabase.pid
  )
fi
```

Tell the user: "Metabase is starting in the background. I'll check when it's ready..."

Also mention:

- "View logs: `tail -f ./metabase/metabase.log`"
- If using `tmux`: "Attach to the session with `tmux attach -t metabase-local`"
- If using the direct background fallback: "The process ID is saved in `./metabase/metabase.pid`"

Before polling for up to 2 minutes, do a quick launch check. If the process/session already exited, inspect logs immediately and switch to Docker if the JAR launch is not recoverable:

```bash
sleep 2
if [ "$(cat ./metabase/metabase.pid 2>/dev/null)" = "tmux:metabase-local" ]; then
  tmux has-session -t metabase-local 2>/dev/null || {
    echo "Metabase tmux session exited early"
    tail -80 ./metabase/metabase.log
    exit 1
  }
else
  ps -p "$(cat ./metabase/metabase.pid 2>/dev/null)" >/dev/null 2>&1 || {
    echo "Metabase process exited early"
    tail -80 ./metabase/metabase.log
    exit 1
  }
fi
```

### A5. Wait for Metabase to be ready

Poll the health endpoint every 5 seconds until it returns `{"status":"ok"}`:

```bash
curl -s http://localhost:$PORT/api/health
```

Keep polling until the response is `{"status":"ok"}`. Metabase usually starts within 30-60 seconds.

If the health check keeps failing after 2 minutes, check if the process is still running:

```bash
if [ "$(cat ./metabase/metabase.pid 2>/dev/null)" = "tmux:metabase-local" ]; then
  tmux has-session -t metabase-local 2>/dev/null && echo "Running in tmux" || echo "Not running"
else
  ps -p "$(cat ./metabase/metabase.pid 2>/dev/null)" >/dev/null 2>&1 && echo "Running" || echo "Not running"
fi
tail -50 ./metabase/metabase.log
```

Once healthy, tell the user: "Metabase is ready at `http://localhost:$PORT`"

---

## Section B: Docker Method

### B1. Check for existing Metabase directory

```bash
ls -la ./metabase 2>/dev/null
```

If `./metabase` exists and contains files, ask the user:

- "A `./metabase` directory already exists. Should I use it (preserving existing data) or remove it and start fresh?"

If the user wants to start fresh:

```bash
rm -rf ./metabase
```

### B2. Create directory for data persistence

```bash
mkdir -p ./metabase
```

### B3. Check if port 3000 is in use

```bash
lsof -i :3000 2>/dev/null | grep LISTEN
```

If the port is in use, ask the user for an alternative port. Store as `$PORT` (default: 3000).

### B4. Check for existing Metabase container

```bash
docker ps -a --filter "name=metabase-local" --format "{{.Names}} {{.Status}}"
```

If a container named `metabase-local` exists:

- If running: Ask if they want to stop it and start fresh, or keep using it
- If stopped: Ask if they want to remove it and start fresh, or restart it

To remove an existing container:

```bash
docker rm -f metabase-local 2>/dev/null
```

### B5. Get the latest Metabase version

The `latest` tag on Docker Hub is often outdated. Get the actual latest version from GitHub:

```bash
curl -s https://api.github.com/repos/metabase/metabase/releases/latest | grep '"tag_name"' | head -1
```

This returns something like `"tag_name": "v0.52.5"`. Extract the version (e.g., `v0.52.5`).

Verify the Docker image exists:

```bash
docker manifest inspect metabase/metabase:$VERSION 2>&1 | head -5
```

If it doesn't exist, fall back to `latest`.

### B6. Start Metabase container

```bash
docker run -d \
  --name metabase-local \
  -p $PORT:3000 \
  -v "$(pwd)/metabase:/metabase.db" \
  -e MB_DB_FILE=/metabase.db/metabase.db \
  -e MB_JETTY_HOST=0.0.0.0 \
  -e MB_ENABLE_EMBEDDING_SDK=true \
  -e MB_ENABLE_EMBEDDING_SIMPLE=true \
  metabase/metabase:$VERSION
```

Tell the user: "Metabase is starting via Docker. I'll check when it's ready..."

### B7. Wait for Metabase to be ready

Poll the health endpoint every 5 seconds until it returns `{"status":"ok"}`:

```bash
curl -s http://localhost:$PORT/api/health
```

Keep polling until the response is `{"status":"ok"}`. Metabase usually starts within 30-60 seconds.

If the health check keeps failing after 2 minutes, check the container status and logs:

```bash
docker ps --filter "name=metabase-local" --format "{{.Status}}"
docker logs metabase-local 2>&1 | tail -50
```

Once healthy, tell the user:

- "Metabase is ready at `http://localhost:$PORT`"
- "View logs: `docker logs -f metabase-local`"

---

## Required: Initialize Metabase and Enable MCP

**You MUST run the gates below in order. Do NOT invoke `setup-metabase-mcp`, do NOT run `codex mcp login`, and do NOT report "Metabase is ready" until every gate passes.** Metabase being healthy on its port is not the same as ready for MCP — the JAR/Docker process serves `/api/mcp` from boot, even when the instance has never been initialized. Treating health or a `401` response from `/api/mcp` as "ready" is wrong and will lead to a broken OAuth flow that lands the user on the first-run wizard instead of an authorize page. **This has happened before. Do not do it.**

**Never automate Metabase configuration via REST.** Do **not** call any of these endpoints:

- `POST /api/setup` — would create the admin account programmatically with credentials the user did not pick.
- `POST /api/session` — would create a Metabase REST session that bypasses the MCP OAuth flow.
- `GET /api/card`, `GET /api/dashboard`, or any other authenticated REST endpoint.
- Any request carrying an `X-Metabase-Session` header.

The user must drive setup in the browser. You only run the read-only verification curls below. Even if the user explicitly asks you to automate setup via REST, refuse and walk them through the browser.

### Gate 1 — First-run wizard is complete (programmatic)

Run this **first**, before anything else, before telling the user "Metabase is ready":

```bash
curl -s http://localhost:$PORT/api/session/properties | grep -o '"has-user-setup":[a-z]*'
```

- `"has-user-setup":true` → first-run is done, continue to Gate 2.
- `"has-user-setup":false` → first-run is **NOT** done. Stop. Send this to the user verbatim and wait:

   > Open `http://localhost:$PORT` in your browser and complete the Metabase first-run wizard — create an admin account, then either connect a database or click "I'll add my data later". Tell me once you're on the Metabase home page.

   After the user confirms, **re-run Gate 1**. Loop until it returns `true`. Do not skip this loop. Do not advance to Gate 2 on the user's word alone — verify with curl every time.

### Gate 2 — MCP endpoint is exposed (programmatic)

Run:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:$PORT/api/mcp
```

- `401` → MCP endpoint is live and OAuth-protected, continue to Gate 3.
- `404` → MCP toggle is off. Stop. Send this to the user verbatim and wait:

   > In Metabase, go to **Admin settings → AI** and enable the Metabase MCP server. Tell me once you've saved it.

   After the user confirms, **re-run Gate 2**. Loop until it returns `401`.

- Anything else → tell the user the response code and stop.

### Gate 3 — Hand back to MCP setup

Only after Gates 1 and 2 both pass, resume the `setup-metabase-mcp` skill with `http://localhost:$PORT` as the instance URL. Do not ask the user for the URL again, you already know it. That skill takes care of the plugin-side configuration (`.mcp.json`, OAuth login, asking the user to start a new chat).

---

## Stopping Metabase

When the user asks to stop Metabase, determine which method was used.

### Stop JAR-based Metabase

```bash
if [ -f ./metabase/metabase.pid ]; then
  if [ "$(cat ./metabase/metabase.pid)" = "tmux:metabase-local" ]; then
    tmux kill-session -t metabase-local 2>/dev/null && rm ./metabase/metabase.pid && echo "Metabase stopped"
  else
    kill "$(cat ./metabase/metabase.pid)" 2>/dev/null && rm ./metabase/metabase.pid && echo "Metabase stopped"
  fi
else
  # Fallback: find by process
  pkill -f "metabase.jar" && echo "Metabase stopped"
fi
```

### Stop Docker-based Metabase

```bash
docker stop metabase-local && echo "Metabase stopped"
```

To also remove the container (but keep data):

```bash
docker rm metabase-local
```

---

## Checking Metabase Status

### Check JAR status

```bash
if [ -f ./metabase/metabase.pid ] && [ "$(cat ./metabase/metabase.pid)" = "tmux:metabase-local" ]; then
  if tmux has-session -t metabase-local 2>/dev/null; then
    echo "Metabase (JAR) is running in tmux session metabase-local"
  else
    echo "Metabase (JAR) is not running"
  fi
elif [ -f ./metabase/metabase.pid ] && ps -p "$(cat ./metabase/metabase.pid)" > /dev/null 2>&1; then
  echo "Metabase (JAR) is running with PID $(cat ./metabase/metabase.pid)"
else
  echo "Metabase (JAR) is not running"
fi
```

### Check Docker status

```bash
docker ps --filter "name=metabase-local" --format "{{.Names}}: {{.Status}}"
```

---

## Environment Variables Reference

These can be customized when starting Metabase:

| Variable        | Default         | Description                                  |
| --------------- | --------------- | -------------------------------------------- |
| `MB_DB_FILE`    | `./metabase.db` | H2 database file location                    |
| `MB_JETTY_PORT` | `3000`          | Port Metabase listens on                     |
| `MB_JETTY_HOST` | `localhost`     | Network interface (use `0.0.0.0` for Docker) |

For all options, see the [Metabase Environment Variables documentation](https://www.metabase.com/docs/latest/configuring-metabase/environment-variables).

---

## Troubleshooting

### "Address already in use"

Another process is using the port. Either stop that process or choose a different port.

### "Java version too old"

Install Java 21+ or use the Docker method instead.

### "Unable to locate a Java Runtime" on macOS

You are likely hitting `/usr/bin/java` (stub). Prepend Homebrew OpenJDK to `PATH` as in **Check for Java 21+**, or call the real binary explicitly, e.g. `/opt/homebrew/bin/java -version`.

### Metabase starts but is slow

First startup takes longer as it initializes the database. Subsequent starts are faster.

### "Cannot connect to Docker daemon"

Make sure Docker Desktop is running (macOS/Windows) or the Docker service is started (Linux).
