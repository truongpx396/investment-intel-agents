# Agent Gateway Abstraction Layer

> **Status:** Draft
> **Created:** 2026-04-12
> **Purpose:** Define a pluggable interface so the project can swap between agent gateway implementations without changing application code.

---

## 1. Overview

This project uses an **Agent Gateway** — an external process that owns digest orchestration, LLM summarisation, and Telegram digest delivery. The gateway is configured declaratively (Markdown skill files, YAML/TOML/JSON config) and communicates with the Go backend and Python ai-service over HTTPS REST.

The architecture is designed so the gateway implementation can be swapped by changing Docker image + config files, **without modifying Go, Python, or React code**.

---

## 2. Supported Frameworks

| Framework | Language | Docker Image | Config Format | Repo |
|-----------|----------|-------------|---------------|------|
| **GoClaw** (default) | Go | `nextlevelbuilder/goclaw:latest` | Markdown (AGENT.md, SKILL.md) | [github.com/nextlevelbuilder/goclaw](https://github.com/nextlevelbuilder/goclaw) |
| **OpenClaw** | TypeScript | `ghcr.io/openclaw/openclaw:latest` | JSON (`openclaw.json`) | [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **PicoClaw** | Go | `ghcr.io/sipeed/picoclaw:latest` | JSON (`config.json`) | [github.com/sipeed/picoclaw](https://github.com/sipeed/picoclaw) |
| **nanoBot** | Python | `ghcr.io/hkuds/nanobot:latest` | JSON (`config.json`) | [github.com/HKUDS/nanobot](https://github.com/HKUDS/nanobot) |
| **ZeroClaw** | Rust | `ghcr.io/zeroclaw-labs/zeroclaw:latest` | TOML (`config.toml`) | [github.com/zeroclaw-labs/zeroclaw](https://github.com/zeroclaw-labs/zeroclaw) |

---

## 3. Required Gateway Capabilities

Any agent gateway used with this project **MUST** support the following capabilities:

### 3.1 Core Capabilities

| Capability | Description | Used For |
|-----------|-------------|----------|
| **Cron scheduling** | Schedule agent tasks at fixed times (e.g. `08:30 UTC daily`) | Daily digest trigger |
| **HTTP / web_fetch** | Make outbound HTTPS requests to external APIs | Calling Go backend (`/internal/`) and Python ai-service (`/news/`, `/enrich/`, `/projects/`) |
| **LLM provider** | Connect to an LLM (Anthropic Claude 3.5 Haiku or equivalent) | Digest summarisation |
| **Telegram channel** | Send messages to a Telegram chat/group | Digest delivery |
| **Subagent / parallel** | Run multiple tasks in parallel and wait for all results | Parallel news fetch per project |
| **Scoped permissions** | Restrict each agent/skill to specific tools and endpoints | Per-agent grants, principle of least privilege |
| **Observability** | Export traces/metrics via OTLP or equivalent | Performance monitoring, SLA tracking |
| **Web dashboard** | Embedded admin UI for health and configuration | Operational visibility |

### 3.2 Communication Contract

The gateway communicates with other services via **HTTPS REST** only. It does NOT:

- Connect to NATS JetStream directly
- Dispatch real-time alerts (Go backend owns this)
- Own Postgres migrations
- Compute technical indicators (Python ai-service owns this)

**Outbound calls:**

| Target | Protocol | Auth | Endpoints |
|--------|----------|------|-----------|
| Go backend | HTTPS | Static bearer token (`AGENT_GATEWAY_INTERNAL_TOKEN`) | `POST /internal/digest`, `GET /internal/projects` |
| Python ai-service | HTTPS | None (internal network) | `GET /news/{slug}`, `POST /enrich/news`, `GET /projects` |

### 3.3 Digest Skill Contract

The gateway must implement a **digest skill** that:

1. Triggers at a configurable cron schedule (default: `08:30 UTC`)
2. Fetches active projects from the Go backend
3. For each project, fetches latest news via the Python ai-service (in parallel)
4. Sends each batch to the LLM for summarisation
5. Delivers the formatted digest to the configured Telegram channel
6. Completes within **5 minutes** (NFR-PERF-004)

---

## 4. Directory Structure

```
agent-gateway/
├── README.md                         # Gateway selection guide
├── docker-compose.gateway.yml        # Active gateway compose (symlink or copy)
│
├── goclaw/                           # GoClaw-specific config (DEFAULT)
│   ├── docker-compose.goclaw.yml
│   ├── .env.goclaw
│   └── agents/
│       └── digest-agent/
│           ├── AGENT.md
│           ├── HEARTBEAT.md
│           └── skills/
│               └── crypto-digest.md
│
├── openclaw/                         # OpenClaw-specific config
│   ├── docker-compose.openclaw.yml
│   ├── .env.openclaw
│   └── openclaw.json
│
├── picoclaw/                         # PicoClaw-specific config
│   ├── docker-compose.picoclaw.yml
│   ├── .env.picoclaw
│   └── config.json
│
├── nanobot/                          # nanoBot-specific config
│   ├── docker-compose.nanobot.yml
│   ├── .env.nanobot
│   └── config.json
│
└── zeroclaw/                         # ZeroClaw-specific config
    ├── docker-compose.zeroclaw.yml
    ├── .env.zeroclaw
    └── config.toml
```

---

## 5. Environment Variables

Common environment variables that all gateway implementations require (variable names may be mapped differently per gateway):

| Variable | Purpose | Example |
|----------|---------|---------|
| `AGENT_GATEWAY_INTERNAL_TOKEN` | Bearer token for Go backend `/internal/` calls | `changeme-static-token` |
| `AGENT_GATEWAY_LLM_API_KEY` | API key for the LLM provider (e.g. Anthropic) | `sk-ant-...` |
| `AGENT_GATEWAY_TELEGRAM_BOT_TOKEN` | Telegram bot token for digest delivery | `123456:ABC-DEF...` |
| `AGENT_GATEWAY_TELEGRAM_CHAT_ID` | Target Telegram chat/group ID | `-1001234567890` |
| `AGENT_GATEWAY_BACKEND_URL` | Go backend base URL | `http://backend:8080` |
| `AGENT_GATEWAY_AI_SERVICE_URL` | Python ai-service base URL | `http://ai-service:8000` |

Each gateway adapter directory (e.g. `goclaw/.env.goclaw`) maps these generic variables to the gateway-specific variable names.

---

## 6. Switching Gateways

### 6.1 Quick Switch

```bash
# 1. Stop the current gateway
docker compose -f agent-gateway/docker-compose.gateway.yml down

# 2. Switch to a different gateway (e.g. from GoClaw to nanoBot)
cp agent-gateway/nanobot/docker-compose.nanobot.yml agent-gateway/docker-compose.gateway.yml
cp agent-gateway/nanobot/.env.nanobot .env.gateway

# 3. Start the new gateway
docker compose -f agent-gateway/docker-compose.gateway.yml up -d

# 4. Verify health
curl http://localhost:18790/health
```

### 6.2 Validation Checklist

After switching gateways, verify:

- [ ] Gateway health endpoint returns 200
- [ ] Cron job is registered (check gateway dashboard)
- [ ] Test digest fires correctly (`/internal/digest` called)
- [ ] Telegram message delivered
- [ ] OTLP traces visible in collector
- [ ] Digest completes within 5 minutes

---

## 7. Framework Comparison (for this project)

| Concern | GoClaw | OpenClaw | PicoClaw | nanoBot | ZeroClaw |
|---------|--------|----------|----------|---------|----------|
| **Language** | Go | TypeScript | Go | Python | Rust |
| **RAM (idle)** | ~35 MB | ~390 MB+ | ~10 MB | ~100 MB+ | ~5 MB |
| **Config format** | Markdown | JSON | JSON | JSON | TOML |
| **Cron** | ✅ Native | ✅ Native | ✅ Native | ✅ Native | ✅ Native |
| **web_fetch / HTTP** | ✅ `web_fetch` | ✅ Tools | ✅ `web_fetch` | ✅ `web_fetch` | ✅ Tools |
| **Telegram** | ✅ Channel | ✅ Channel | ✅ Channel | ✅ Channel | ✅ Channel |
| **LLM Anthropic** | ✅ Provider | ✅ Provider | ✅ Provider | ✅ Provider | ✅ Provider |
| **Subagent / parallel** | ✅ `waitAll` | ✅ Sessions | ✅ Spawn | ✅ Subagent | ✅ Hands |
| **Scoped permissions** | ✅ Per-agent grants | ✅ Multi-agent routing | ✅ Config-based | ✅ Config-based | ✅ Autonomy levels |
| **OTLP / observability** | ✅ Native | ✅ Native | ✅ Log-based | ✅ Log-based | ✅ Native |
| **Web dashboard** | ✅ Embedded | ✅ Embedded | ✅ WebUI | ✅ API-based | ✅ React dashboard |
| **MCP support** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **License** | Proprietary | MIT | MIT | MIT | MIT / Apache-2.0 |

---

## 8. Design Principles

1. **Gateway as a black box** — The Go backend and Python ai-service treat the gateway as an opaque HTTP client. They expose REST endpoints and don't care which gateway calls them.

2. **Config-per-gateway** — Each gateway gets its own config directory. No shared config files that require framework-specific syntax.

3. **Single Telegram bot** — All gateways share the same Telegram bot token. Only one gateway runs at a time to avoid conflicts.

4. **Digest skill parity** — The digest output format is defined by the LLM prompt, not the gateway. Switching gateways should produce identical digests given the same LLM and prompt.

5. **Docker-first** — All gateways run as Docker containers in the same Docker network. Gateway switching is a compose-file swap.

---

## 9. Default: GoClaw

GoClaw v1.74+ is the **default and recommended** gateway for this project because:

- Battle-tested in production for the original spec
- Smallest Go binary (~25 MB) with low RAM (~35 MB idle)
- Native OTLP observability
- Markdown-based agent config (AGENT.md / SKILL.md) is version-control friendly
- Built-in web dashboard at port 18790
- Per-agent grants for fine-grained security

The other frameworks are supported as alternatives for teams that prefer a different language ecosystem or have existing infrastructure.
