# SpeedtestConsolidator

A self-hosted, cross-platform speed test system: a **server** that collects
speed test results from one or more **agents**, with a web dashboard to view
and chart the history. Works on Windows, macOS, and Linux.

```
+---------------------+      HTTP      +-----------------------+
|   Speedtest Agent   |---------------->|  Consolidator Server  |
|  (agent.py)         |  POST /results |  (run_server.py)      |
|  - ping / download  |                |  - SQLite database    |
|    / upload tests   |                |  - REST API           |
|  - scheduled loop   |                |  - Web dashboard        |
+---------------------+                +-----------------------+
```

## Project layout

```
server/          Flask server: API, SQLite storage, speedtest endpoints, dashboard
agent/           Agent: speed test engine, result sender, main loop, service installers
common/          Shared constants and data models (used by both server and agent)
tests/           Unit + integration + security / vulnerability tests
installer/       Cross-platform installer (bundles all dependencies)
static/          Dashboard CSS and JavaScript
templates/       Dashboard HTML
config.yaml      Server configuration
agent.yaml.example  Agent configuration template
requirements.txt Python dependencies (pinned, cross-platform)
```

## Server

The server gathers speed test results from agents and exposes them through a
web dashboard and a REST API.

### Features
- **REST API** — `POST /api/results` for agents (authenticated with `X-API-Key`)
- **Self-hosted speed test endpoints** — `GET /speedtest/ping`,
  `GET /speedtest/download`, `POST /speedtest/upload`
- **SQLite storage** — single file database, no separate DB server required
- **Web dashboard** — stats cards, recent results, agents table, download /
  upload / ping history charts, auto-refreshing
- **Rate limiting** — per-IP submission limits to prevent abuse
- **Security headers** — CSP, `X-Content-Type-Options`, `X-Frame-Options`,
  `Referrer-Policy`
- **Parameterised queries** — no SQL injection surface
- **Production-ready** — waitress WSGI server (`--production`)

### Quick start

```bash
# install dependencies
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS / Linux: source .venv/bin/activate
pip install -r requirements.txt

# configure (edit api.key to a strong secret)
python -c "import os; print(os.urandom(24).hex())"

# start the server
python run_server.py --production
```

Open the dashboard at `http://localhost:8080/dashboard`.

### Configuration (`config.yaml`)

| Section | Key | Meaning |
|---------|-----|---------|
| `server.host` | bind address (default `0.0.0.0`) |
| `server.port` | bind port (default `8080`) |
| `server.debug` | Flask debug mode |
| `api.key` | shared secret agents send in `X-API-Key` |
| `database.path` | SQLite file path |
| `rate_limit.*` | submissions per window per IP |
| `test.*` | default payload sizes / ping count / timeout |

Environment variables prefixed with `SPEEDTEST_*` override any config value
(see `server/config.py`).

## Agent

The agent runs speed tests against your server and reports the results.

### Features
- **Ping, download, upload** measurements streamed in bounded chunks
  (constant memory on Windows, macOS, Linux)
- **One-shot mode** (`--once`) or **continuous loop** (interval from config)
- **Authenticated submission** with retries and exponential backoff
- **Service installation** — `python run_agent.py --install-service` sets up
  systemd (Linux), launchd (macOS), or a Windows service (`sc.exe`)

### Quick start

```bash
cp agent.yaml.example agent.yaml
# edit agent.yaml: set server url and api_key to match the server

# single test
python run_agent.py --once

# continuous loop
python run_agent.py

# install as a background service
python run_agent.py --install-service
```

### Configuration (`agent.yaml`)

| Section | Key | Meaning |
|---------|-----|---------|
| `server.url` | consolidator server base URL |
| `server.interval_seconds` | seconds between tests |
| `api_key` | must match the server's `api.key` |
| `test.download_size_mb` | download payload size in MB |
| `test.upload_size_mb` | upload payload size in MB |
| `test.ping_count` | number of ping samples |
| `test.timeout_seconds` | per-test timeout |
| `agent.name` | display name in the dashboard |
| `agent.agent_id` | blank = generated from machine identity |

## Installer (includes all dependencies)

The installer creates a virtual environment, installs every dependency from
`requirements.txt`, writes default configs, initialises the SQLite database,
and optionally installs the agent as a background service.

```bash
# full install (server + agent)
python installer/install.py

# server only / agent only
python installer/install.py --server-only
python installer/install.py --agent-only

# install agent as a background service
python installer/install.py --install-service
```

Platform scripts are also provided:
- `installer/posix/install.sh` — macOS / Linux
- `installer/windows/install.bat` — Windows

## Testing

Run the full test suite (unit, integration, and vulnerability tests):

```bash
pip install pytest pytest-mock
python -m pytest tests/ -v
```

The suite covers:
- SQLite storage (including SQL injection attempts)
- REST API (authentication, validation, rate limiting)
- Result sender (correct headers, no retry on 401, retry on network failure)
- Speed test calculations
- Security: SQL injection, XSS storage/escaping, missing API key, rate
  limits, security headers

## Security

- Change the default `api.key` before exposing the server to a network.
- For public use, run the server behind a reverse proxy (nginx / Caddy)
  with TLS.
- The dashboard is read-only; only authenticated agents can write results.

## License

MIT
