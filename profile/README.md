<div align="center">

# VYNN AI

**Autonomous Financial Intelligence Platform**

*Institutional-quality equity research in under 7 minutes. Built end-to-end by a single engineer.*

[![Users](https://img.shields.io/badge/Pilot%20Users-~500-brightgreen)]()
[![Python](https://img.shields.io/badge/Backend-15%2C000%2B%20LOC-blue)]()
[![Agents](https://img.shields.io/badge/Agents-5%20Specialized-orange)]()
[![Latency](https://img.shields.io/badge/E2E%20Latency-%3C7%20min-red)]()
[![Live](https://img.shields.io/badge/Production-vynnai.com-black)](https://vynnai.com)

[Live Demo](https://vynnai.com/demo) · [Architecture](#architecture) · [Agent Pipeline](#agent-backend-stock-analyst) · [API Layer](#api-layer-api-runner) · [Frontend](#frontend-gpt-web) · [Performance](#performance-benchmarks) · [Contact](#contact)

</div>

---

## Why VYNN AI?

Equity research at institutional firms takes 6–12 hours per ticker — an analyst manually pulls financials, builds a DCF model in Excel, reads through dozens of news articles, writes up a report, and formulates a recommendation. Most retail investors and small firms simply can't afford this.

VYNN AI compresses that entire workflow into a single autonomous pipeline. You give it a ticker. It gives you a professional analyst report — complete with a 9-tab DCF model, sector-specific valuation, news-driven catalyst/risk analysis, and a validated recommendation with multi-horizon price targets.

No prompt engineering. No manual data entry. No hallucinated numbers.

**Key results:**
- **<7 min** end-to-end for a comprehensive equity analysis
- **~500 pilot users** in production on Hetzner Cloud
- **0.985** reproducibility score (CV 0.016) across repeated runs
- **100%** intent recognition stability across paraphrased natural language queries
- **72%** latency reduction via parallel agent execution and result caching
- **$0** external data vendor costs — all data sourced from public APIs

---

## Architecture

Three-layer stack — agent backend, API orchestration layer, and React frontend — all designed, built, and deployed by a sole engineer.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Frontend (gpt-web)                           │
│         React 18 · TypeScript · Vite · Tailwind · shadcn/ui        │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│   │  AI Chat UI  │  │   Market     │  │  Portfolio Management   │  │
│   │  (SSE Stream) │  │  Dashboard   │  │  (6 Chart Types)       │  │
│   └──────┬───────┘  └──────┬───────┘  └────────────┬────────────┘  │
│          │ SSE              │ WebSocket (×2)        │ REST          │
├──────────┴──────────────────┴───────────────────────┴──────────────-┤
│                      API Layer (api-runner)                          │
│              FastAPI · Docker SDK · MongoDB · Redis                  │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐  │
│   │  Job Manager │  │  WebSocket   │  │   Auth / Sessions       │  │
│   │  (DinD)      │  │  Hub         │  │   (OAuth + Email)       │  │
│   └──────┬───────┘  └──────┬───────┘  └─────────────────────────┘  │
│          │ Docker SDK       │ yfinance + MongoDB                    │
├──────────┴──────────────────┴──────────────────────────────────────-┤
│                   Agent Backend (stock-analyst)                      │
│         LangGraph · Python 3.11 · 15,000+ LOC · 40+ modules        │
│                                                                     │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │                    Supervisor Agent                         │   │
│   │          (LLM-powered intent routing + fallback)           │   │
│   └────┬────────┬────────┬────────┬────────┬───────────────────┘   │
│        ▼        ▼        ▼        ▼        ▼                       │
│   ┌────────┐┌────────┐┌────────┐┌────────┐┌─────────────────┐     │
│   │Financial││  DCF   ││  News  ││ Report ││ Recommendation  │     │
│   │  Data  ││ Model  ││ Intel  ││  Gen   ││    Engine       │     │
│   │ Agent  ││ Agent  ││ Agent  ││ Agent  ││  (3-layer)      │     │
│   └────────┘└────────┘└────────┘└────────┘└─────────────────┘     │
│                                                                     │
│   Shared State: FinancialState (Blackboard Pattern)                 │
│   Prompts: 33 externalized markdown templates in prompts/           │
│   LLM Layer: Provider-agnostic (OpenAI + Anthropic)                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Agent Backend (`stock-analyst`)

The core reasoning engine. ~15,000+ lines of Python across 40+ modules with 33 externalized prompt templates.

### Orchestration

Built on **LangGraph's cyclical state graph** using a Supervisor-Worker architecture:

- **Supervisor Agent** — LLM-powered orchestrator that extracts tickers from natural language, classifies user intent into one of four modes (`COMPREHENSIVE` | `MODEL_ONLY` | `QUICK_NEWS` | `CUSTOM`), and dynamically routes between worker agents with dependency resolution. Includes a deterministic fallback path when LLM routing fails, ensuring 100% request completion.
- **Blackboard State** — All agents read from and write to a shared `FinancialState` frozen dataclass. This guarantees data consistency across the pipeline and enables any agent to access upstream results without direct coupling.
- **Execution** — Agents run with maximum parallelism where dependencies allow. Financial Data and News Intelligence execute concurrently; DCF Model waits for Financial Data; Report Generator waits for all upstream agents.

### Specialized Agents

#### 1. Financial Data Agent
Collects financial statements (income statement, balance sheet, cash flow) from Yahoo Finance via `yfinance`. Normalizes raw pandas DataFrames into clean, structured JSON suitable for downstream agents.

#### 2. Financial Model Agent (DCF Builder)
Generates a **9-tab Excel workbook** with live formulas:

| Tab | Purpose |
|-----|---------|
| Raw Data | Normalized financial statements |
| Keys Map | Standardized field mapping across data sources |
| Assumptions | FY0 actuals + FY1–FY5 projections (LLM-inferred growth rates, margins, capex) |
| Historical Metrics | Computed ratios and trends from raw data |
| 5-Year Projections | Revenue, EBITDA, FCF, working capital projections |
| Perpetual Growth DCF | Terminal value via Gordon Growth Model |
| Exit Multiple DCF | Terminal value via EV/EBITDA exit multiple |
| Sensitivity Matrices | Price sensitivity across discount rate × growth rate |
| Summary Dashboard | Consolidated valuation output with dual-method comparison |

A custom **Formula Evaluator** (1,293 lines) interprets Excel formula syntax programmatically, enabling downstream agents to query computed values without opening the workbook.

**6 sector-specific DCF strategies**, auto-selected based on company classification:

| Strategy | Sector | Key Methodology |
|----------|--------|-----------------|
| Generic | Default | Standard UFCF → WACC discounting |
| SaaS / Rule of 40 | Technology | Revenue growth + FCF margin ≥ 40% |
| REIT / FFO | Real Estate | Funds From Operations, NAV-based |
| Bank / Excess Returns | Financials | ROE vs. cost of equity spread |
| Utility | Utilities | Regulated asset base, dividend yield |
| Energy NAV | Energy | Reserve-based net asset value |

#### 3. News Intelligence Agent
Three-stage pipeline:

1. **Query Generation** — LLM generates targeted search queries from the ticker and context
2. **Scraping + Filtering** — Google News via SerpAPI → full article extraction via `newspaper3k` → LLM batch relevance scoring → MongoDB persistence (deduplication via `urlHash`)
3. **Deep Analysis** — Structured extraction of catalysts, risks, mitigations, sentiment, confidence scores, direct quotes, and evidence chains for each relevant article

#### 4. Report Generator Agent
Synthesizes all agent outputs into an **institutional-quality analyst report**:

- Executive Summary
- Investment Thesis (bull/bear/base cases)
- Financial Analysis (historical trends, margin analysis, growth trajectory)
- Valuation (dual DCF with sensitivity analysis)
- News & Catalyst Analysis (with evidence chains from News Intelligence)
- Risk Assessment (systematic, company-specific, sector-level)
- Recommendation with multi-horizon price targets (3-month, 6-month, 12-month)

Output is rendered as structured markdown and converted to downloadable PDF via ReportLab.

#### 5. Recommendation Engine
A unique **3-layer architecture** that ensures no hallucinated financial numbers:

```
Layer 1: RecommendationCalculator (Deterministic Python)
  → Expected returns, price targets, rating bands
  → The LLM never invents numbers — all figures come from this layer

Layer 2: EvidenceExtractor + LLM (Narrative Generation)
  → Builds evidence pack with unique citation IDs (e.g., [FIN-001], [NEWS-003])
  → LLM writes narrative prose referencing citations

Layer 3: RecommendationValidator (Regex-Based Verification)
  → Every number in the narrative is cross-checked against Layer 1 source
  → Requires ≥95% citation coverage
  → Auto-correction loop if validation fails
```

### Daily Intelligence Reports

Automated pre-market reports generated at 8:30 AM ET (Mon–Fri):

- **Company Daily** — Last 24h news, catalyst/risk mapping, peer context, sentiment shift tracking
- **Sector Daily** — Cross-company aggregation, sector rotation trends, thematic signals

Each report follows a 3-step LLM workflow: information gathering → structured synthesis → quality validation.

### LLM Abstraction Layer

Provider-agnostic interface supporting runtime model switching:

- **Supported providers:** OpenAI (GPT-4o, GPT-4o-mini), Anthropic (Claude 3.5 Sonnet, Claude 3.5 Haiku, Claude 3 Opus)
- **Features:** Per-call cost tracking, automatic retry with exponential backoff, token usage logging, graceful fallback between providers
- **Prompt management:** All 33 prompts are externalized as versioned markdown files in `prompts/` — version-controlled, auditable, hot-swappable without code changes

### Key Design Patterns

| Pattern | Where | Why |
|---------|-------|-----|
| Supervisor + Worker | LangGraph orchestration | Dynamic routing with dependency resolution |
| Blackboard | `FinancialState` dataclass | Decoupled agents sharing structured state |
| Builder | Excel tab generation | Each tab is an independent, testable builder class |
| Strategy | DCF sector selection | Pluggable valuation methodologies without conditional logic |
| Prompt Externalization | `prompts/` directory | Iterate on prompts without touching agent code |

---

## API Layer (`api-runner`)

FastAPI 0.104 + Uvicorn ASGI orchestration service. Bridges the agent backend with the frontend and manages all real-time data streams.

### Docker-in-Docker Execution

The API layer does **not** run analysis in-process. Instead:

1. User submits analysis request via REST
2. API Runner spawns an **ephemeral Docker container** (`fuzanwenn/stock-analyst:latest`, ~975 MB) via the Docker SDK, mounting the host Docker socket
3. Container runs the full agent pipeline in isolation
4. Logs stream back via SSE; results persist to MongoDB
5. Container is automatically cleaned up on completion or timeout

This provides complete process isolation, prevents memory leaks from affecting the API, and enables horizontal scaling by running multiple analysis containers concurrently.

### Real-Time Streaming

| Protocol | Purpose | Implementation |
|----------|---------|----------------|
| SSE | Job progress + agent logs | Batched log emission, 15s heartbeats, session ID extraction from container stdout, completion signal detection |
| WebSocket #1 | Live stock prices | 10s polling interval via yfinance, subscriber-based fan-out, per-ticker subscription management |
| WebSocket #2 | News feed | MongoDB change streams + background refresh, dead-connection pruning, per-ticker subscriptions |

### Authentication

Multi-provider system with HTTP-only cookie sessions:

- **Google OAuth 2.0** — Full OAuth flow with PKCE
- **GitHub OAuth** — Token exchange + profile fetch
- **Email Verification Code** — Passwordless login via 6-digit code with TTL

### Additional Services

- **Daily Report Scheduler** — Pre-market cron (8:30 AM ET, Mon–Fri), auto-skips weekends and NYSE holidays
- **PDF Generation** — Markdown → PDF via ReportLab with table of contents, internal cross-links, and custom styling
- **Health Monitoring** — `/health` (comprehensive system check: Docker, MongoDB, Redis) and `/healthz` (Kubernetes-style liveness probe)
- **Shared Data Layer** — `vynn-core` package provides MongoDB + Redis client wrappers used across all services

---

## Frontend (`gpt-web`)

React 18 + TypeScript 5.9 + Vite 5 + Tailwind CSS 3.4 + shadcn/ui (40+ Radix UI primitives)

### AI Chat Interface

- Multi-conversation management with session persistence
- SSE streaming with log batching and natural-language summary extraction
- Downloadable artifacts: `.xlsx` (DCF model), `.pdf` (analyst report)
- Virtualized message list (`react-window`) for performance with long conversations
- Rich markdown rendering with syntax highlighting

### Market Dashboard

- **Live Stock Prices** — Persistent WebSocket connection, real-time ticker cards with sparkline charts
- **Interactive Charts** — Recharts-based with 7 timeframe options (1D, 5D, 1M, 3M, 6M, 1Y, All)
- **News Aggregation** — WebSocket-streamed, ticker-based subscriptions, article deduplication
- **Market Status** — Algorithmic NYSE holiday computation (including Easter via anonymous Gregorian algorithm), pre-market/after-hours/regular session detection

### Portfolio Management

- Multi-portfolio CRUD with real-time P&L calculations via WebSocket price feed
- **6 interactive chart types:** Area, Bar, Pie, Radar, Scatter, Treemap (Recharts)
- One-click PNG export for any chart
- Holdings table with live gain/loss, allocation percentages, and cost basis tracking

### AI-Generated Daily Reports

- Three report categories: Company Daily, Sector Daily, Global Market Overview
- Smart batch generation with polling-based status tracking
- Dual-report capture via module-scoped singleton SSE refs (survive React component unmounts)

### Design System

- **Theme:** Luxury dark mode with amber/gold accent palette, glass-morphism card effects, serif branding typography
- **Light mode:** Full support with automatic system preference detection
- **Components:** 40+ shadcn/ui components built on Radix UI primitives

### Frontend Engineering

| Challenge | Solution |
|-----------|----------|
| Two persistent WebSocket connections | Subscriber-based architecture with exponential backoff reconnection and health-check pings |
| SSE streams outliving React components | Module-scoped singleton refs (not component state) for stream continuity |
| NYSE market hours with holidays | Algorithmic holiday computation including Easter, no hardcoded date lists |
| User-scoped data isolation | `userStorage` wrapper over localStorage with user ID namespacing |
| Complex provider nesting | 5 context providers with explicit dependency ordering to prevent circular updates |
| TypeScript adoption in legacy codebase | Progressive migration strategy — strict mode for new modules, ambient declarations for legacy |

---

## Performance Benchmarks

| Metric | Value | Notes |
|--------|-------|-------|
| End-to-end analysis | **383s (6.4 min)** | Full comprehensive analysis for a single ticker |
| Reproducibility | **0.985** (CV 0.016) | Consistency across repeated identical runs |
| Intent recognition | **100%** | Stability across paraphrased natural language prompts |
| Financial data + DCF build | **<10s combined** | Non-LLM operations are fast |
| LLM-intensive operations | **~93% of total time** | News analysis + report generation dominate latency |
| Latency reduction | **72%** | Via parallel agent execution and cross-run result caching |

---

## Tech Stack

| Layer | Technologies |
|-------|-------------|
| **Agent Backend** | Python 3.11, LangGraph, yfinance, SerpAPI, newspaper3k, openpyxl, ReportLab |
| **API Layer** | FastAPI 0.104, Uvicorn, Docker SDK, MongoDB (Motor), Redis, SSE, WebSocket |
| **Frontend** | React 18, TypeScript 5.9, Vite 5, Tailwind CSS 3.4, shadcn/ui, Recharts, react-window |
| **Infrastructure** | Docker Compose, Hetzner Cloud VPS, Caddy (reverse proxy + automatic HTTPS), Nginx (SPA) |
| **LLM Providers** | OpenAI (GPT-4o, GPT-4o-mini), Anthropic (Claude 3.5 Sonnet, Haiku, Opus) |
| **Data Storage** | MongoDB (documents + news), Redis (caching + sessions) |
| **Auth** | Google OAuth 2.0, GitHub OAuth, email verification codes |
| **CI/CD** | Multi-arch Docker builds (linux/amd64 + linux/arm64), zero-downtime deployments |

---

## Repository Structure

```
vynn-ai/
├── stock-analyst/          # Agent backend — LangGraph multi-agent pipeline
│   ├── agents/             # 5 specialized agent implementations
│   ├── prompts/            # 33 externalized markdown prompt templates
│   ├── models/             # LLM abstraction layer + provider configs
│   ├── financial/          # DCF builders, formula evaluator, sector strategies
│   ├── news/               # News scraping, filtering, and analysis pipeline
│   └── reporting/          # Report generation + recommendation engine
├── api-runner/             # FastAPI orchestration layer
│   ├── routes/             # REST endpoints + SSE/WebSocket handlers
│   ├── services/           # Docker job manager, auth, scheduling
│   └── core/               # MongoDB/Redis clients, config, middleware
├── gpt-web/                # React frontend
│   ├── src/components/     # Chat, dashboard, portfolio, report UIs
│   ├── src/contexts/       # WebSocket, auth, theme providers
│   ├── src/hooks/          # Custom hooks for streaming, real-time data
│   └── src/utils/          # Market hours, formatting, storage helpers
├── vynn-core/              # Shared package (MongoDB + Redis wrappers)
└── docker-compose.yml      # Full-stack local development
```

---

## Getting Started

Visit **[vynnai.com](https://vynnai.com)** to explore the platform.

VYNN AI is currently in limited access. For early access or to join the waitlist, reach out:

**Email:** zanwen.fu@duke.edu
**LinkedIn:** [linkedin.com/in/zanwenfu](https://linkedin.com/in/zanwenfu)

---

## Contact

For inquiries, collaborations, or technical discussions:

- **Email:** zanwen.fu@duke.edu
- **Website:** [zanwenfu.com](https://zanwenfu.com)
- **GitHub:** [github.com/zanwenfu](https://github.com/zanwenfu)
- **LinkedIn:** [linkedin.com/in/zanwenfu](https://linkedin.com/in/zanwenfu)

---

<div align="center">

*Built with conviction that AI agents should ship to production, not just demo well.*

</div>
