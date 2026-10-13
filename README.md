# This repository has been replaced by github.com/getstratum/stratum

# Stratum — AI Governance Platform

**Open-source AI gateway for enterprise teams. Control, visibility, and governance over every AI API call.**

> 📖 [Leer en español](./README.es.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Version](https://img.shields.io/badge/version-0.0.2--beta-violet)](./CHANGELOG.md)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)](./docker-compose.yml)
[![Node.js](https://img.shields.io/badge/Node.js-20+-339933?logo=node.js)](./proxy/package.json)

---

## What is Stratum?

Stratum sits between your teams and every AI provider. You define the rules once. Every request — from developers, business users, and automated agents — passes through Stratum before reaching OpenAI, Anthropic, Gemini, or any other provider.

```
Developer / Agent / Playground user
          ↓
    Stratum Gateway          ← runs inside your network
    (auth · policy · log)
          ↓
  OpenAI / Anthropic / Gemini / Local models
```

No prompt content is ever stored. Only metadata: who called what, when, how many tokens, at what cost.

---

## The problem

When teams adopt AI independently, organizations end up with:

- **No visibility** — nobody knows the real AI spend or who uses which model
- **No control** — any developer can call GPT-4o with no limits
- **No governance** — sensitive data reaches providers without oversight
- **Fragmented costs** — invoices scattered across teams and tools

Stratum is a single layer that addresses all four.

---

## Key features

| Feature | Description |
|---------|-------------|
| **API Proxy** | Drop-in replacement for OpenAI/Anthropic base URLs. Zero code changes for developers. |
| **Policy engine** | Model whitelists, token quotas, and budget caps enforced in real time via Redis. |
| **Dashboard** | Analytics by team, provider, and model. Date filters. CSV export. No prompt content stored. |
| **Dashboard auth** | Engineering team members access the admin dashboard. Other teams are redirected to the Playground. |
| **AI Playground** | Chat interface for non-technical users. Same policies as the API. File attachments. Persistent history. |
| **Agents** | Register automated processes with dedicated keys (`aig_agt_`). Admin configures the model centrally. |
| **Provider management** | Add, enable, or disable AI providers from the UI — no code changes required. |
| **Context compression** | Local LLM (Ollama) summarizes older messages before sending to the provider. Reduces token spend. |
| **Alerts** | Email and Slack notifications when teams approach budget or token limits. |
| **Observability** | Per-request timing (auth, policy, provider). Prometheus `/metrics` endpoint. |
| **Split-plane architecture** | Gateway runs inside the customer's network. Prompts never leave their infrastructure. |

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Control Plane                       │
│  Dashboard (Next.js) · Policies · Analytics          │
└─────────────────────┬────────────────────────────────┘
                      │ Metadata only — no prompt content
┌─────────────────────▼────────────────────────────────┐
│                   Data Plane                          │  ← Customer's network
│   AI Gateway (Node.js/Fastify)                       │
│   PostgreSQL (logs) · Redis (quotas) · Ollama (local)│
└─────────────────────┬────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
     OpenAI API  Anthropic API  Gemini API  …
```

---

## Quick start

### Prerequisites
- Docker and Docker Compose
- API keys for at least one provider (OpenAI or Anthropic)

### 1. Clone and configure

```bash
git clone https://github.com/matiasmospan/stratum.git
cd stratum
cp .env.example .env
```

Edit `.env` and fill in:

```env
POSTGRES_PASSWORD=your-secure-db-password
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Dashboard — gateway URL visible to the browser
NEXT_PUBLIC_GATEWAY_URL=http://localhost:8080

# Dashboard session security (change before going to production)
SESSION_SECRET=generate-a-random-string-here
```

### 2. Start the stack

```bash
docker compose up -d --build
```

### 3. Run migrations and seed demo data

```bash
# Apply all database migrations
docker exec -it $(docker compose ps -q proxy) node /app/src/migrate.js

# Set demo user passwords
docker exec -it $(docker compose ps -q proxy) node /app/src/seed-passwords.js
```

### 4. Access

| Service | URL | Credentials |
|---------|-----|-------------|
| Dashboard (admin) | http://localhost:3000 | `dev@acme.com` / `dev123` (Engineering team) |
| Playground | http://localhost:3000/prompt | `dev@acme.com` / `dev123` or `marketing@acme.com` / `mkt123` |
| Gateway API | http://localhost:8080 | API key or JWT |

---

## Making API calls

Change only the base URL. Everything else stays the same.

```bash
# Before (direct to OpenAI)
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'

# After (through Stratum)
curl http://localhost:8080/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer aig_sk_eng_test1234567890" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hello"}]}'
```

### Agent calls — model-agnostic

When an agent has a model configured in the dashboard, use `/proxy/auto/`:

```python
# Agent doesn't need to know which model or provider to use
client = openai.OpenAI(
    api_key="aig_agt_xxxx",
    base_url="http://localhost:8080/proxy/auto/v1"
)
response = client.chat.completions.create(
    model="any",          # overridden by Stratum with the configured model
    messages=[{"role": "user", "content": prompt}]
)
```

---

## Token types

| Prefix | Type | Use case |
|--------|------|----------|
| `aig_sk_` | Developer key | API calls, CI/CD, SDK usage |
| `aig_agt_` | Agent key | Automated processes, pipelines |
| JWT (`.`) | Session token | Playground and dashboard (email + password) |

---

## Supported providers

| Provider | Status | Type | Notes |
|----------|--------|------|-------|
| OpenAI | ✅ Active | openai-compatible | GPT-4o, GPT-4o-mini, o3 |
| Anthropic | ✅ Active | anthropic | Claude Sonnet, Haiku, Opus |
| Google Gemini | ✅ Active | openai-compatible | Gemini 3.6 Flash, 3.5 Flash, 3.5 Flash-Lite |
| Bonsai 8B (local) | ✅ Via Ollama | openai-compatible | PrismML ternary-quantized, ~1 GB, on-premise |
| Ollama (local) | ✅ Via Ollama | openai-compatible | Any model via local Ollama instance |
| Groq | 🔧 Configurable | openai-compatible | Add API key to enable |
| Azure OpenAI | 🔧 Configurable | openai-compatible | Add endpoint and API key to enable |
| AWS Bedrock | 🔜 Planned | aws-bedrock | Requires AWS Sig V4 — not yet implemented |

Adding a new provider is a data operation — no code changes needed. Go to `/providers` in the dashboard.

---

## Dashboard access model

- **Engineering team** → full dashboard (`/`)
- **Other teams** → redirected to Playground (`/prompt`)
- **Playground** → always accessible, has its own email/password auth

To change which teams have dashboard access, edit `DASHBOARD_TEAMS` in `dashboard/middleware.js`.

---

## Context compression

When enabled in the Playground, Stratum uses a local LLM (Ollama `llama3.2:1b`) to summarize older messages before sending to the provider. This reduces token spend on long conversations without losing context.

```
First 6 messages → sent normally
Message 7 → older messages compressed to summary → summary + last 4 messages sent
```

Enable by clicking the **⬡ Compress** toggle in the Playground header. The compression panel shows tokens saved and a countdown to the next compression.

---

## Project structure

```
stratum/
├── proxy/                  # AI Gateway (Node.js + Fastify)
│   └── src/
│       ├── index.js        # Entry point and route registration
│       ├── auth.js         # Three-token auth (API key · agent key · JWT)
│       ├── proxy.js        # HTTP proxying, model injection, SSE token counting
│       ├── providers.js    # Dynamic provider loader from DB (60s cache)
│       ├── policy.js       # Redis quota enforcement
│       ├── logger.js       # Async request logging to PostgreSQL
│       ├── alerter.js      # Email and Slack threshold alerts
│       ├── metrics.js      # Prometheus /metrics endpoint
│       ├── login.js        # Password auth + JWT signing
│       └── migrate.js      # DB migration runner
├── dashboard/              # Admin UI + Playground (Next.js 14)
│   ├── middleware.js       # Route protection (Engineering team gate)
│   └── app/
│       ├── login/          # Dashboard login page
│       ├── page.jsx        # Overview — General / By provider / By model tabs
│       ├── teams/          # Team management + policy assignment
│       ├── users/          # User management (password + API key)
│       ├── agents/         # Agent management (model injection + metrics)
│       ├── logs/           # Request log viewer with filters and trace detail
│       ├── policy/         # Policy editor (model whitelist, quotas, budgets)
│       ├── alerts/         # Alert configuration (email + Slack)
│       ├── providers/      # Provider CRUD (add/enable/disable)
│       └── prompt/         # AI Playground (persistent history, file upload, compression)
├── db/
│   ├── 001_schema.sql      # Full schema with partitioned request_logs
│   ├── 002_seed.sql        # Demo org, teams, users, models
│   ├── 003_auth.sql        # Password hash column
│   ├── 004_agents.sql      # Agents table
│   ├── 005_observability.sql # Per-stage timing columns
│   ├── 006_providers.sql   # Providers table with seeded entries
│   └── 007_bonsai.sql      # Bonsai local provider
├── scripts/
│   └── migrate.js          # Safe, idempotent migration runner
├── docs/
│   ├── en/                 # English documentation
│   └── es/                 # Spanish documentation
├── .env.example
├── docker-compose.yml
└── LICENSE                 # MIT
```

---

## GCP deployment

```bash
# Create VM (e2-standard-4 recommended — 4 vCPUs, 16 GB RAM)
gcloud compute instances create stratum-dev \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd \
  --tags=stratum-server

gcloud compute firewall-rules create stratum-allow-web \
  --allow=tcp:3000 --target-tags=stratum-server
gcloud compute firewall-rules create stratum-allow-gateway \
  --allow=tcp:8080 --target-tags=stratum-server

# SSH and setup
gcloud compute ssh stratum-dev --zone=us-central1-a
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker
sudo apt-get install -y docker-compose-plugin
```

Upload the project and configure `.env` with the public IP:

```env
NEXT_PUBLIC_GATEWAY_URL=http://YOUR_VM_IP:8080
GATEWAY_INTERNAL_URL=http://proxy:8080
```

Then:

```bash
docker compose up -d --build
docker exec -it $(docker compose ps -q proxy) node /app/src/migrate.js
docker exec -it $(docker compose ps -q proxy) node /app/src/seed-passwords.js
```

To reduce costs when not in use:
```bash
gcloud compute instances stop stratum-dev --zone=us-central1-a
gcloud compute instances start stratum-dev --zone=us-central1-a
```

---

## Roadmap

### v0.0.2-beta (current)
- [x] Dashboard login with team-based access control
- [x] Light mode UI with violet accent
- [x] Analytics tabs (General / By provider / By model) with date filters
- [x] CSV export for logs and analytics
- [x] Provider management from UI (add/enable/disable)
- [x] Google Gemini 3.x integration
- [x] Bonsai 8B local model via Ollama
- [x] Playground: file attachments (PDF, images, text)
- [x] Playground: persistent chat history (user-scoped localStorage)
- [x] Context compression with local LLM + savings panel
- [x] Alert system (email + Slack)
- [x] Observability: per-stage timing + Prometheus `/metrics`
- [x] Auto-migration script
- [x] GCP deployment guide

### v0.1.0 (next)
- [ ] SSO connectors (Azure AD, Okta)
- [ ] AWS Bedrock provider
- [ ] Kubernetes Helm chart
- [ ] Multi-organization support
- [ ] Prompt caching (Anthropic + OpenAI)
- [ ] Conversation history in database (not just localStorage)

### Enterprise (commercial)
- PII / data loss prevention scanning
- Compliance audit exports (SOC 2, GDPR, EU AI Act)
- On-premise deployment support + SLA
- Dedicated support and onboarding

---

## Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `POSTGRES_PASSWORD` | ✅ | PostgreSQL password |
| `OPENAI_API_KEY` | ✅ (if using OpenAI) | OpenAI API key |
| `ANTHROPIC_API_KEY` | ✅ (if using Anthropic) | Anthropic API key |
| `GEMINI_API_KEY` | If using Gemini | Google AI Studio API key |
| `NEXT_PUBLIC_GATEWAY_URL` | ✅ | Gateway URL visible to the browser |
| `GATEWAY_INTERNAL_URL` | ✅ in Docker | Internal gateway URL (`http://proxy:8080`) |
| `SESSION_SECRET` | ✅ | Dashboard session signing secret (min 32 chars) |
| `JWT_SECRET` | Recommended | Proxy JWT signing secret |
| `OLLAMA_URL` | If using Ollama | Ollama endpoint (default: `http://ollama:11434`) |
| `OLLAMA_MODEL` | If using compression | Model for context compression (default: `llama3.2:1b`) |
| `OLLAMA_API_KEY` | If using Ollama | Any value — Ollama doesn't require auth |
| `SECURE_COOKIES` | In production with HTTPS | Set to `true` to enable secure cookie flag |

---

## Contributing

Contributions are welcome. Please open an issue first to discuss significant changes.

---

## License

MIT — free to use, modify, and distribute. The open-source core will always remain free.

---

## Commercial support

Need enterprise features, dedicated support, or deployment assistance?
Open an issue or reach out via the repository.
