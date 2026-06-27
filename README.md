# PulseNet — AI-Driven Energy Supply Chain Resilience

> **ET AI Hackathon 2026 · Problem Statement 2: AI-Driven Energy Supply Chain Resilience for Import-Dependent Economies**

PulseNet is a predictive decision-support platform that monitors geopolitical and logistics risk signals in real time, models supply disruption scenarios across India's energy import network, and generates executable procurement rerouting recommendations — turning reactive crisis response into a managed, anticipatory process.

**Core principle: LLMs parse unstructured chaos → deterministic code does the math → every action ends at a human.**

---

## The Problem

India sources approximately 88% of its crude oil from imports, with 40–45% of that volume transiting through the Strait of Hormuz. India's Strategic Petroleum Reserves cover roughly 9.5 days of national consumption — a buffer that would be exhausted quickly in any sustained disruption. Traditional supply chain planning tools have no ability to model geopolitical scenario impacts in real time, dynamically evaluate alternative procurement corridors, or orchestrate coordinated response across refiners, logistics providers, and strategic reserves.

The intelligence layer to do this needs to be built. PulseNet builds it.

---

## What PulseNet Does

PulseNet ingests live shock signals from global feeds — seismic events, conflict reports, port closures, shipping disruptions — and maps them onto a trade dependency graph to identify which downstream regions and commodities are at risk, how severe the exposure is, and what alternative supply routes are available. Every step is explainable, every number is traceable, and every rerouting recommendation ends at a human decision.

### Core capabilities

**Live Signal Ingestion** — Concurrent ingestion from USGS seismic feeds, GDACS disaster alerts, ACLED conflict data, Reuters, BBC, Al Jazeera, GDELT, and maritime shipping news. No manual data collection. Keyless sources run with zero API cost.

**Byzantine Consensus Agents** — Two Gemini LLM agents (Alpha and Beta) independently parse each raw news item into a structured shock event. A third Gamma agent computes a Byzantine disagreement score (δ). When agents diverge significantly, confidence is penalised and flagged for human review. This replaces single-model hallucination risk with measurable inter-agent disagreement.

**Trade Dependency Graph** — A directed graph of supplier→consumer relationships across 20 countries and 4 critical commodities (LPG, Refined Diesel, Wheat, Pharmaceuticals), with import share and volume on each edge. Graph traversal identifies every downstream consumer exposed to an upstream shock.

**Sovereign Recuperation Index (SRI)** — A per-country resilience score combining GDP per capita (log-normalised), grid infrastructure density, and historical supply volatility. Used to weight cascade confidence and shortage time estimates so fragile nations are never silently treated as "no risk."

**Monte Carlo Cascade Simulation** — 4,000 trials per reroute candidate, producing shortage-window distributions (median and p95) and success probabilities. Every reroute card shows the worst case, not just the expected case.

**Human-in-the-Loop (HITL) Controls** — Every rerouting recommendation ends at an Approve / Reject / Adjust step. PulseNet never executes a reroute autonomously. Every decision is logged to a full audit trail with actor and timestamp.

**Equity-Weighted Confidence** — Countries with sparse monitoring infrastructure receive an explicit `⚠ LOW CONFIDENCE — VERIFY` flag on every exposure card. Under-monitored regions are never silently excluded from the risk picture.

---

## Architecture

```
[ USGS GeoJSON · GDACS · ReliefWeb · Reuters · BBC · Al Jazeera · GDELT · ACLED ]
                                     │
                                     ▼ raw feed items
                    ┌────────────────────────────────┐
                    │  Ingestion Filter               │
                    │  Alpha Agent (Gemini Key A) ──┐ │
                    │  Beta Agent  (Gemini Key B) ──┼─┤→ Gamma (Byzantine δ)
                    │  JSON mode: response_mime_type │ │   disagreement score
                    │  = application/json            │ │   logged to ledger
                    └────────────────────────────────┘
                                     │ validated ConsensusShock
                                     ▼
                    ┌────────────────────────────────┐
                    │  Ripple Evaluator               │
                    │  Graph traversal (networkx)     │
                    │  SRI cascade confidence         │
                    │  Monte Carlo (numpy, 4k trials) │
                    │  LLM-enriched reroute rationale │
                    └────────────────────────────────┘
                                     │ shocks, exposures, reroutes
                                     ▼
                    ┌────────────────────────────────┐
                    │  Shared SQLite (WAL mode)       │
                    │  Prisma schema + consensus      │
                    │  ledger                         │
                    └────────────────────────────────┘
                                     │
                                     ▼
                    ┌────────────────────────────────┐
                    │  Next.js Dashboard              │
                    │  Threat map · Exposed regions   │
                    │  Reroute queue · HITL controls  │
                    │  Audit trail · Consensus ledger │
                    └────────────────────────────────┘
```

**Design split:** LLMs handle unstructured chaos (text → structured event). Deterministic Python handles the math (graph traversal, SRI, Monte Carlo). This keeps every number traceable and avoids asking a language model to do arithmetic.

---

## Sovereign Recuperation Index (SRI)

The SRI models a nation's capacity to stabilise after an infrastructural shock:

```
SRI = 0.5 × GDP_norm + 0.3 × grid_density − 0.2 × historical_volatility
```

GDP per capita is log-normalised so high-income outliers do not dominate the scale. Output is clamped to [0.05, 1.0]. The cascade recuperation factor is derived as:

```
sri_scaled = 1 + SRI × 4          # rescale to [1, 5]
factor     = 1 − 1 / sri_scaled   # → [0, 0.80] for SRI ∈ [0, 1]
```

This gives fragile nations (SRI = 0.2 → vulnerability = 0.56) higher cascade confidence than resilient nations (SRI = 0.9 → vulnerability = 0.24), correctly reflecting real-world exposure asymmetry. The formula is identical in both `prisma/seed.ts` (TypeScript) and `app/compute/sri.py` (Python) — no drift between seed data and the live engine.

---

## Tech Stack

### Frontend
- **Framework**: Next.js 16 (App Router) · TypeScript 5
- **Styling**: Tailwind CSS 4 · shadcn/ui (New York) · Lucide icons
- **State**: Zustand · TanStack Query
- **Theme**: next-themes (dark default)

### Backend (Python FastAPI Engine)
- **Framework**: FastAPI 0.115 · Uvicorn
- **AI**: Google Gemini 2.5 Flash via `google-genai` SDK (JSON mode)
- **Graph**: NetworkX 3.2
- **Simulation**: NumPy 1.26 (Monte Carlo)
- **Feeds**: feedparser (RSS) · httpx (async HTTP)
- **DB**: SQLAlchemy 2.0 on SQLite (WAL mode)

### Database
- **ORM**: Prisma 6 (TypeScript schema owner)
- **Engine**: SQLite with WAL mode — zero infrastructure, portable

---

## Project Structure

```
PulseNet-alpha/
├── src/
│   ├── app/api/              # Next.js API routes (proxy to Python engine)
│   │   ├── shocks/           # GET list, POST ingest, GET/PATCH by id
│   │   ├── ripple/           # POST — run Monte Carlo ripple evaluation
│   │   ├── decisions/        # GET/POST — HITL audit trail
│   │   ├── graph/            # GET — trade dependency graph
│   │   ├── ledger/           # GET — Byzantine consensus ledger
│   │   └── stats/            # GET — dashboard headline metrics
│   ├── components/pulsenet/  # Dashboard UI components
│   │   ├── threat-map.tsx    # Canvas world map with epicenters + flow lines
│   │   ├── shock-feed.tsx    # Live shock event cards
│   │   ├── reroute-queue.tsx # Ranked reroutes with HITL controls
│   │   ├── exposed-regions.tsx
│   │   ├── audit-trail.tsx
│   │   └── responsible-ai.tsx # Byzantine ledger + governance panel
│   └── lib/pulsenet/
│       ├── ingest.ts         # TS fallback ingestion (USGS + web search)
│       ├── ripple.ts         # TS fallback ripple evaluator
│       ├── geo.ts            # Haversine, country matching, map projection
│       └── zai.ts            # Gemini SDK singleton
│
├── mini-services/pulsenet-engine/   # Python FastAPI engine
│   └── app/
│       ├── agents/           # LLM clients, Alpha/Beta parsers, Gamma consensus
│       ├── compute/          # sri.py, cascade.py, monte_carlo.py (pure math)
│       ├── feeds/            # USGS, RSS, ACLED, feed registry
│       ├── services/         # ingest_service.py, ripple_service.py
│       └── main.py           # FastAPI wiring
│
├── prisma/
│   ├── schema.prisma         # Data model
│   └── seed.ts               # Trade graph + 2 replay shock scenarios
│
└── Makefile                  # make install / dev / engine / test / seed / reset
```

---

## Getting Started

### Prerequisites
- Node.js 18+ and npm
- Python 3.11+
- `uv` package manager for Python (`pip install uv`)

### 1. Clone and install

```bash
git clone <repo-url>
cd PulseNet-alpha
make install
```

This runs `npm install`, pushes the Prisma schema to SQLite, and installs the Python engine dependencies in a virtual environment.

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` and fill in your API keys:

```env
# Required — get both free from https://aistudio.google.com → "Get API key"
# Use two separate Google accounts to double your rate limit budget
GEMINI_API_KEY_A=your_key_here
GEMINI_API_KEY_B=your_second_key_here

# Default model (free tier, 15 RPM per key)
GEMINI_MODEL=gemini-2.5-flash

# Python engine URL (default for local dev)
PULSENET_ENGINE_URL=http://localhost:8000

# Optional — ACLED conflict feed (free academic registration at acleddata.com)
ACLED_KEY=
ACLED_EMAIL=

# Optional — web search in TS fallback mode (free tier at serper.dev)
SEARCH_API_KEY=
```

### 3. Seed demo data

```bash
make seed
```

Loads 20 countries, 4 commodities, 44 trade edges, and 2 replay shock scenarios:
- **Persian Gulf seismic event** — pre-evaluated with exposures and reroutes
- **Black Sea port closures** — awaiting evaluation (run it yourself in the demo)

### 4. Run

In two terminals:

```bash
# Terminal 1 — Next.js dashboard
make dev

# Terminal 2 — Python FastAPI engine
make engine
```

Open `http://localhost:3000`.

### 5. Run tests

```bash
make test   # 41 tests, all pass
```

---

## API Keys Summary

| Key | Source | Cost | Required |
|---|---|---|---|
| `GEMINI_API_KEY_A` | [aistudio.google.com](https://aistudio.google.com) | Free (15 RPM) | **Yes** |
| `GEMINI_API_KEY_B` | Second Google account | Free (15 RPM) | Recommended |
| `ACLED_KEY` + `ACLED_EMAIL` | [acleddata.com/access](https://acleddata.com/access) | Free (academic) | Optional |
| `SEARCH_API_KEY` | [serper.dev](https://serper.dev) | Free (2,500/month) | Optional |

Keyless data sources (zero cost, no signup): USGS earthquakes, GDACS disasters, ReliefWeb, GDELT, Reuters RSS, BBC RSS, Al Jazeera RSS, Maritime Executive RSS.

---

## Live Demo Flow (90 Seconds)

**Step 1 — Open the dashboard.** Stats bar shows live metrics: shock count, exposed regions, pending reroutes, low-confidence count.

**Step 2 — Run ingestion.** Click **Run Ingestion**. The engine concurrently fetches all configured feeds, Alpha and Beta agents independently parse each item into structured shock events, Gamma computes Byzantine disagreement scores, and new events appear in the Live Shock Feed with source badges (USGS / WebSearch).

**Step 3 — Evaluate a ripple.** Click the **Black Sea port closures** card (status: NEW). Its epicenter appears on the threat map. Click **Evaluate Ripple**. The graph traversal identifies every downstream consumer of Russian and Ukrainian exports, SRI-weighted cascade confidence is computed per country, Monte Carlo runs 4,000 trials per reroute candidate, and the Exposed Regions list populates (Egypt wheat risk 76, Kenya wheat 61, Pakistan diesel 51, ...). Countries with monitoring density below 0.55 show a red `⚠ LOW CONFIDENCE — VERIFY` flag.

**Step 4 — Review and decide.** The Reroute Queue shows ranked alternatives with success probability, shortage window (median + p95), cost increase, and time-to-add. Click **Approve**, **Adjust** (add a note), or **Reject**. The Decision Log records every action with actor and timestamp.

**Step 5 — Inspect the governance panel.** The Responsible AI panel shows the Byzantine consensus ledger: every ingestion run's Alpha/Beta agreement score, disagreement delta, and confidence adjustment.

---

## Responsible AI Design

| Principle | Implementation |
|---|---|
| Decision-support only | PulseNet never executes a reroute. Every recommendation ends at a human Approve / Reject / Adjust step. |
| Equity-weighted confidence | Countries with sparse monitoring infrastructure are flagged `⚠ LOW CONFIDENCE — VERIFY` — never silently excluded from the risk picture. |
| Explainable by design | Every exposure has a causal path string. Every reroute has a source citation and full Monte Carlo breakdown (median, p95, success probability, trial count). |
| Honest uncertainty | Monte Carlo reports both the expected outcome and the worst case (p95 shortage window). |
| Byzantine consensus transparency | Inter-agent disagreement (δ) is logged to the consensus ledger and visible in the governance panel. |
| Full audit trail | Every action — ingest, evaluate, approve, reject, adjust, dismiss — is logged with actor and timestamp. |

---

## Test Suite

41 tests across the Python engine, all passing.

| File | Coverage |
|---|---|
| `test_sri.py` | SRI formula, log-normalisation, weight correctness |
| `test_cascade.py` | Recuperation factor, path confidence, DAG structure |
| `test_monte_carlo.py` | Determinism with seeded RNG, probability bounds |
| `test_gamma_consensus.py` | Byzantine δ math, confidence penalisation |
| `test_feeds_parse.py` | Haversine distance, country matching, RSS/USGS parsing (mocked network) |
| `test_parser.py` | LLM JSON parse, deterministic keyword fallback, country-code filtering |
| `test_ripple_service.py` | Graph traversal, exposure creation, ledger row, idempotent re-evaluation |
| `test_ingest_service.py` | Feed → shock pipeline, deduplication, ledger write |
| `test_api.py` | All FastAPI endpoints via TestClient |
| `test_backward_chain.py` | 2022 neon → semiconductor cascade: asserts P > 0.12 for downstream manufacturers |

---

## REST API Reference

All routes return JSON.

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/stats` | Dashboard headline metrics |
| `GET` | `/api/shocks` | All shock events with exposure and reroute counts |
| `POST` | `/api/shocks/ingest` | Run the ingestion agent pipeline |
| `GET` | `/api/shocks/:id` | Full shock detail: exposures, reroutes, Monte Carlo outcomes |
| `PATCH` | `/api/shocks/:id` | Update shock status (`{ "status": "dismissed" }`) |
| `POST` | `/api/ripple` | Run ripple evaluator (`{ "shockId": "..." }`) |
| `GET` | `/api/decisions` | Audit trail (latest 60 entries) |
| `POST` | `/api/decisions` | Record HITL decision (`{ "suggestionId", "action", "note?" }`) |
| `GET` | `/api/graph` | Trade dependency graph (countries, commodities, edges) |
| `GET` | `/api/ledger` | Byzantine consensus ledger entries |

---

## Resetting the Demo

```bash
make reset
```

Wipes all runtime data (ingested shocks, evaluations, decisions) and reloads the trade graph plus two replay scenarios. Safe to run any time during a demo.

---

## Deployment

PulseNet runs as two services: the Next.js frontend (port 3000) and the FastAPI Python engine (port 8000). Both use only public packages and standard cloud runtimes.

**Railway (recommended):** Deploy each service as a separate Railway service from the same repo. Set `PULSENET_ENGINE_URL` in the frontend's environment variables to the Railway URL of the Python service. SQLite persists on a mounted volume.

**Docker Compose (local/offline demo):**

```bash
docker compose up
```

The included `docker-compose.yml` starts both services and seeds the database automatically.

**Vercel + Render:** Deploy the Next.js frontend to Vercel and the Python engine to Render's free tier. Set `PULSENET_ENGINE_URL` accordingly.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Empty feed on load | Run `make seed` to reload demo data, then refresh |
| Ingestion returns 0 new events | No qualifying events in the current feed window, or all items deduplicated. Try targeting a specific source or wait a few minutes. |
| Evaluate Ripple shows 0 exposures | The shock's country codes have no supplier edges in the trade graph. Honest output — not a bug. The toast shows the engine's note. |
| Reroute cards show generic text | LLM call failed or timed out. Deterministic fallback text is shown — reroutes are still valid. |
| Prisma errors after schema change | Run `npx prisma db push` then `make seed` |

---

*PulseNet — civic infrastructure, not a corporate or defense product.*