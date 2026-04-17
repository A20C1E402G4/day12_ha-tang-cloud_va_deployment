# Day 12 Lab — Mission Answers

> **Student:** dduyanhhoang@gmail.com  
> **Date:** 17/4/2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **Hardcoded secrets** (line 17–18) — `OPENAI_API_KEY` and `DATABASE_URL` written directly in source code. If pushed to GitHub, credentials are permanently exposed.
2. **Secret logged to console** (line 34) — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` prints the API key in logs, which may be stored or forwarded to external systems.
3. **`print()` instead of structured logging** (lines 33–38) — `print` output cannot be filtered, aggregated, or parsed by log management tools (Datadog, Loki, etc.).
4. **`DEBUG = True` hardcoded** (line 21) — debug mode in production exposes stack traces and internal details to clients.
5. **No health check endpoint** — no `/health` or `/ready` route, so the platform cannot detect crashes or know when the app is ready to serve traffic.
6. **Port hardcoded to `8000`** (line 52) — cloud platforms (Railway, Render) inject the port via `$PORT` env var; a fixed port will conflict or be unreachable.
7. **Bound to `localhost`** (line 51) — `host="localhost"` means the server only accepts connections from within the same machine; inside a Docker container it becomes completely unreachable from outside.
8. **`reload=True` in production** (line 53) — hot-reload is a dev-only feature that wastes CPU and can cause unexpected restarts in production.
9. **No graceful shutdown** — no SIGTERM handler, so in-flight requests are killed immediately when the platform restarts the container.
10. **No input validation** — `/ask` accepts any string as a query parameter with no length limit or type checking.

---

### Exercise 1.2: Fixes in `01-localhost-vs-production/production/app.py`

| Anti-pattern | Fix applied |
|---|---|
| Hardcoded secrets | `config.py` reads every value from `os.getenv()` |
| Secret in logs | Logs only `question_length` and `client_ip`, never the key |
| `print()` logging | `logging.basicConfig` with JSON format string |
| `DEBUG = True` | `settings.debug = os.getenv("DEBUG","false") == "true"` |
| No health check | `/health` (liveness) and `/ready` (readiness) endpoints added |
| Hardcoded port | `port = int(os.getenv("PORT", "8000"))` |
| `localhost` binding | `host = os.getenv("HOST", "0.0.0.0")` |
| `reload=True` | `reload=settings.debug` — only on when DEBUG env var is set |
| No graceful shutdown | `signal.signal(SIGTERM, handle_sigterm)` + lifespan context manager |
| No input validation | `HTTPException(422)` if question field missing |

---

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Secrets | Hardcoded in source | `os.getenv()` in `config.py` | Leaked secrets = permanent security breach |
| Port | Fixed `8000` | `$PORT` env var | Cloud platforms assign ports dynamically |
| Binding | `localhost` (127.0.0.1) | `0.0.0.0` | Container won't accept external connections otherwise |
| Logging | `print()` to stdout | Structured JSON via `logging` | JSON is parseable; plain text is not |
| Debug mode | Always `True` | Controlled by `DEBUG` env var | Exposes stack traces to end users |
| Health check | None | `/health` + `/ready` | Platform needs these to restart or route traffic |
| Shutdown | Abrupt kill | SIGTERM handler + lifespan teardown | Prevents in-flight request loss during deploys |
| Config | Mixed in code | Single `Settings` dataclass | One place to change, validates at startup |

---

### Exercise 1.4: Why `config.py` matters

`config.py` implements the **12-Factor App** principle: "Store config in the environment."  
Benefits:
- The same Docker image runs in dev, staging, and production — only env vars differ
- `settings.validate()` fails fast at startup if a required variable is missing
- No secrets ever touch source control

---

## Part 2: Docker

### Exercise 2.1: Dockerfile analysis

**Basic (`02-docker/develop/Dockerfile`):**
- Base image: `python:3.11` (~1 GB full distribution)
- Working directory: `/app`
- Stages: **1** (single-stage)
- Build tools included in final image: yes (gcc, pip, etc.)
- Runs as: `root`
- Health check: no

**Production (`02-docker/production/Dockerfile`):**
- Stage 1 base image: `python:3.11-slim` (builder)
- Stage 2 base image: `python:3.11-slim` (runtime)
- Working directory: `/app`
- Stages: **2** (builder + runtime)
- Build tools in final image: **no** — only copied site-packages
- Runs as: `appuser` (non-root)
- Health check: yes — `python -c "import urllib.request; ..."` every 30s

---

### Exercise 2.2: Multi-stage build explanation

**Stage 1 — builder:**
```
FROM python:3.11-slim AS builder
RUN apt-get install gcc libpq-dev    ← build tools needed to compile C extensions
RUN pip install --user -r requirements.txt   ← installs to /root/.local
```

**Stage 2 — runtime:**
```
FROM python:3.11-slim AS runtime     ← fresh slim image, no build tools
COPY --from=builder /root/.local ... ← only the compiled packages
COPY main.py .
USER appuser                         ← non-root for security
CMD ["uvicorn", ...]
```

Result: build tools, gcc, and pip cache never reach the final image.

---

### Exercise 2.3: Image size comparison

Run these commands and fill in:
```bash
# From project root
docker build -f 02-docker/develop/Dockerfile -t agent-develop .
docker build -f 02-docker/production/Dockerfile -t agent-production .
docker images | grep agent
```

| Image | Size | Notes |
|-------|------|-------|
| `agent-develop` | 1.66 GB | Single-stage, full python:3.11 |
| `agent-production` | 240 MB | Multi-stage, python:3.11-slim runtime |
| Difference | ~85% smaller | Multi-stage strips build tools from final image |

---

### Exercise 2.4: Docker Compose stack

Services in `02-docker/production/docker-compose.yml`:

| Service | Image | Role | Port exposed? |
|---------|-------|------|--------------|
| `agent` | built from Dockerfile | FastAPI AI agent | No (internal only) |
| `redis` | redis:7-alpine | Session cache, rate limiting | No (internal only) |
| `qdrant` | qdrant/qdrant:v1.9.0 | Vector DB for RAG | No (internal only) |
| `nginx` | nginx:alpine | Reverse proxy + load balancer | **80, 443** → public |

Key design decisions:
- Only Nginx exposes ports — agent is never directly reachable from outside
- All services share the `internal` bridge network
- `agent` depends on `redis` and `qdrant` with `condition: service_healthy` — won't start until they pass health checks
- Redis uses LRU eviction at 256 MB (`--maxmemory-policy allkeys-lru`)

---

### Exercise 2.5: Security improvements in production Dockerfile

1. **Non-root user** — `RUN groupadd -r appuser && useradd -r -g appuser appuser` + `USER appuser`  
   If the container is compromised, the attacker has only `appuser` permissions, not root.

2. **Multi-stage build** — build tools (gcc, pip, apt cache) are stripped from the final image  
   Smaller attack surface: fewer binaries means fewer exploitable vulnerabilities.

3. **`HEALTHCHECK` directive** — Docker daemon restarts the container automatically if health check fails  
   Self-healing without platform intervention.

4. **No secrets baked in** — env vars injected at runtime via `env_file: .env.local` (git-ignored)

---

## Part 3: Cloud Deployment

### Exercise 3.1: Deployment Details

- **Deployed URL:** `https://delightful-ambition-production-37e2.up.railway.app`
- **Platform chosen:** Railway
- **Why Railway:** Fastest zero-config deployment — `railway init` + `railway up` reads `railway.toml` automatically, assigns a public HTTPS domain, and injects `$PORT` without any manual setup. No credit card required for hobby tier.

### Exercise 3.2: Verification

```bash
# Root endpoint — confirms app is running
curl https://delightful-ambition-production-37e2.up.railway.app/
# {"message":"AI Agent running on Railway!","docs":"/docs","health":"/health"}

# Health check
curl https://delightful-ambition-production-37e2.up.railway.app/health
# {"status":"ok","uptime_seconds":804.8,"platform":"Railway","timestamp":"2026-04-17T08:16:33.562930+00:00"}
```

### Exercise 3.3: Deployment Config (`railway.toml`)

Key settings that made it work:
- `builder = "nixpacks"` — auto-detects Python, installs deps from `requirements.txt`
- `startCommand` uses `$PORT` — Railway injects the assigned port at runtime
- `healthcheckPath = "/health"` — Railway restarts the container if this stops returning 200

---

## Part 4: API Security

> **Note:** This app uses **JWT authentication**.
> Flow: `POST /auth/token` → get Bearer token → include in every request.

### Exercise 4.1: Auth required (401)

```bash
curl -s -w "\nHTTP %{http_code}\n" -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"hello"}'
```

Output:
```
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
HTTP 401
```

### Exercise 4.2: Auth works (200)

```bash
# Step 1 — get JWT token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token -H "Content-Type: application/json" -d '{"username":"student","password":"demo123"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Step 2 — use token
curl -s -w "\nHTTP %{http_code}\n" -X POST http://localhost:8000/ask -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question":"what is docker?"}'
```

Output:
```
{"question":"what is docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
HTTP 200
```

### Exercise 4.3: Rate limiting (429)

```bash
for i in {1..15}; do curl -s -o /dev/null -w "Request $i: %{http_code}\n" -X POST http://localhost:8000/ask -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question":"test"}'; done
```

Output:
```
Request 1: 200
Request 2: 200
Request 3: 200
Request 4: 200
Request 5: 200
Request 6: 200
Request 7: 200
Request 8: 200
Request 9: 200
Request 10: 200
Request 11: 429
Request 12: 429
Request 13: 429
Request 14: 429
Request 15: 429
```

First 10 requests succeed (sliding window limit = 10/min per user). Requests 11–15 return 429 with `Retry-After` header.

### Exercise 4.4: Cost guard explanation

The cost guard (`cost_guard.py`) protects against surprise LLM bills:

- **Token cost formula:**
  - Input: `(tokens / 1000) × $0.00015` (GPT-4o-mini rate)
  - Output: `(tokens / 1000) × $0.0006`
  - Tokens estimated as `len(text.split()) × 2`
- **Per-user daily budget:** $1.00 — returns `402 Payment Required` when exhausted
- **Global daily budget:** $10.00 — returns `503 Service Unavailable` when exhausted
- **Warning threshold:** logs a warning at 80% usage
- **Reset:** compares `time.strftime("%Y-%m-%d")` — resets automatically at midnight UTC
- **Multi-instance note:** in-memory state means each instance tracks separately; production should use Redis for accurate global tracking

---

## Part 5: Scaling & Reliability

*(To be completed)*

### Exercise 5.1: Why in-memory state breaks scaling

When running 3 instances, the load balancer routes each request to a different container. If session data (conversation history) is stored in instance memory, a user's second request may land on a different instance that has no record of the first. Redis solves this by providing a shared store all instances read/write to.

### Exercise 5.2: Stateless test output
```bash
# Run:
cd 05-scaling-reliability/production
docker compose up --scale agent=3
python test_stateless.py

# Paste actual output here
```

### Exercise 5.3: SIGTERM graceful shutdown
1. Platform sends `SIGTERM` to container
2. `signal.signal(SIGTERM, handler)` catches it and logs the event
3. Uvicorn's `timeout_graceful_shutdown=30` waits up to 30s for in-flight requests
4. FastAPI lifespan teardown block runs: `_is_ready = False`, Redis connections closed
5. Container exits with code 0 — no requests are dropped
