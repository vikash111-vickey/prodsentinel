# ProdSentinel — AI Production Failure Oracle

> **"Every post-mortem your team ever wrote is a data point you paid for with an outage. ProdSentinel makes sure you never pay twice."**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue)](https://python.org)
[![Claude API](https://img.shields.io/badge/Powered%20by-Claude%20API-purple)](https://anthropic.com)

---

## What is ProdSentinel?

ProdSentinel is an AI-powered production safety system that learns from your organization's past incidents and automatically intercepts dangerous code changes **before they reach production**.

Most AI code review tools check your code against general programming knowledge. ProdSentinel checks your code against **your company's own failure history** — post-mortems, incident reports, Slack threads, Jira tickets. It builds a unique failure fingerprint for your organization and uses it to catch history repeating itself at merge time.

**In plain English:** A developer opens a PR. ProdSentinel analyses the diff, searches your incident history, and posts a comment: *"This change is 87% similar to the auth outage from March 2024. Predicted blast radius: all authenticated endpoints. Recommended: deploy behind a feature flag."* — before a single line merges.

---

## The Problem

Engineering teams using AI coding tools are shipping more code than ever. But according to DORA 2026, **higher AI adoption correlates with higher production instability simultaneously.** The bottleneck is no longer writing code — it's validating it safely.

Existing tools miss the most important signal: your organization's own failure patterns. No linter, no generic AI reviewer, and no static analysis tool knows that *your* team has broken prod three times by touching the session invalidation logic.

ProdSentinel does.

---

## How It Works

ProdSentinel runs as a swarm of four specialized AI agents, triggered automatically on every pull request.

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub PR opened                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │   Agent 2: PR Analyst   │
              │  Parses diff → extracts │
              │  structural change map  │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │ Agent 3: Pattern Matcher│
              │ Vector search against   │
              │ incident knowledge base │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Agent 4: Report Agent  │
              │ Risk score + blast map  │
              │ Posted as PR comment    │
              └─────────────────────────┘

  ┌─────────────────────────────────────┐
  │   Agent 1: Ingestion Agent          │
  │   Runs on setup + whenever new      │
  │   incidents are added               │
  │   Jira / Notion / Slack → vectors   │
  └─────────────────────────────────────┘
```

### Agent 1 — Ingestion Agent
Scrapes your post-mortems, incident Jira tickets, Slack channel exports, and runbooks. Uses Claude to extract structured failure patterns: what changed, what broke, blast radius, root cause. Stores as vector embeddings in a local Chroma database.

### Agent 2 — PR Analyst
Triggered by GitHub webhook on every PR. Fetches the diff and uses Claude to produce a structural change map — not just syntax, but call graphs, service dependencies, config mutations, database migrations, and deployment surface area.

### Agent 3 — Pattern Matcher
Vector-searches the incident knowledge base against the structural change map. Ranks matches by similarity. Returns the top matching past incidents with confidence scores.

### Agent 4 — Report Agent
Synthesizes findings into a human-readable GitHub PR comment: risk score (0–100), matched past incident, predicted blast radius, and recommended safeguards (canary deploy, feature flag, rollback plan). Can be configured to block merge above a risk threshold via GitHub branch protection rules.

---

## Demo

| Input | Output |
|---|---|
| PR changing session timeout config | ⚠️ Risk: 87% — matches March 2024 auth outage |
| PR adding a new UI button | ✅ Risk: 3% — no matching failure patterns |
| PR modifying payment retry logic | ⚠️ Risk: 74% — matches Nov 2023 payment loop incident |

---

## Architecture Overview

```
prodsentinel/
├── agents/
│   ├── ingestion_agent.py       # Post-mortem scraping + embedding
│   ├── pr_analyst.py            # PR diff → structural change map
│   ├── pattern_matcher.py       # Vector similarity search
│   └── report_agent.py          # Risk report generation
├── core/
│   ├── vector_store.py          # Chroma DB interface
│   ├── github_client.py         # GitHub API wrapper
│   └── orchestrator.py          # Agent pipeline runner
├── api/
│   └── webhook.py               # FastAPI webhook server
├── dashboard/                   # Next.js live risk feed UI
├── scripts/
│   └── ingest.py                # CLI: run ingestion manually
├── tests/
├── .env.example
├── docker-compose.yml
└── README.md
```

---

## AI Tools Used

| Tool | Purpose |
|---|---|
| **Claude API (claude-sonnet-4-6)** | Failure pattern extraction, PR structural analysis, risk report generation, blast radius prediction |
| **Claude Embeddings** | Converting incident text and PR diffs into vector representations |
| **Chroma DB** | Local vector store for incident embeddings and similarity search |
| **LangChain** | Agent orchestration framework |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11, FastAPI |
| AI / LLM | Anthropic Claude API |
| Vector DB | Chroma (local) |
| Version Control Integration | GitHub Webhooks + REST API |
| Frontend Dashboard | Next.js 14, Tailwind CSS |
| Containerization | Docker + Docker Compose |
| Deployment | Railway / Render (free tier) |

---

## Setup Instructions

### Prerequisites
- Python 3.10+
- Node.js 18+
- A GitHub account with a test repository
- Anthropic API key ([get one here](https://console.anthropic.com))

### 1. Clone the repository
```bash
git clone https://github.com/YOUR_TEAM/prodsentinel.git
cd prodsentinel
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
cd dashboard && npm install
```

### 3. Configure environment
```bash
cp .env.example .env
```

Edit `.env`:
```env
ANTHROPIC_API_KEY=your_key_here
GITHUB_TOKEN=your_github_token
GITHUB_WEBHOOK_SECRET=your_webhook_secret
CHROMA_PERSIST_DIR=./data/chroma
```

### 4. Ingest your incident history
```bash
python scripts/ingest.py --source ./sample_incidents/
```

Sample incidents are included in `sample_incidents/` so you can demo immediately without real post-mortems.

### 5. Start the webhook server
```bash
uvicorn api.webhook:app --reload --port 8000
```

### 6. (Optional) Start the dashboard
```bash
cd dashboard && npm run dev
```
Dashboard available at `http://localhost:3000`

### 7. Connect to GitHub
- Go to your repo Settings → Webhooks → Add webhook
- Payload URL: `https://your-tunnel.ngrok.io/webhook`
- Content type: `application/json`
- Events: Pull requests

---

## Test Credentials (Live Demo)

| Field | Value |
|---|---|
| Live URL | `https://prodsentinel.up.railway.app` |
| Demo GitHub repo | `github.com/prodsentinel-demo/test-repo` |
| Dashboard login | `demo@prodsentinel.ai` / `hackathon2025` |

---

## Team

| Name | Role |
|---|---|
| [VIKASH M] | AI agent architecture, Claude API integration ,Frontend dashboard, demo & presentation |
| [YUKTHASHREE G R] | Backend, GitHub webhook pipeline |
 Frontend dashboard, demo & presentation |

---

## Why This Hasn't Been Built Before

1. **Generic AI reviewers share training data with AI code generators** — they have correlated blind spots. ProdSentinel uses org-specific data, not general knowledge.
2. **Post-mortems have always been backward-looking** — written after the incident, read by nobody. ProdSentinel makes them forward-looking and automatic.
3. **The vector similarity approach requires combining** PR diff analysis + incident embedding + blast radius reasoning in one pipeline — a multi-agent architecture that only became practical with modern LLMs.

---

## License

MIT License — see [LICENSE](LICENSE) for details.
