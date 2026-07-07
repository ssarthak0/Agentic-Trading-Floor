# Agentic Trading Floor

A multi-agent AI paper-trading platform where four autonomous traders research markets, make decisions, and execute trades through [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) servers. The project includes two dashboards: a **Gradio** UI for quick Python-native monitoring and a **Vite + TypeScript** SPA for a modern web experience.

> **Disclaimer:** This is a simulation for educational and research purposes. It does not execute real trades or provide financial advice.

## Features

- **Four AI traders** with distinct investment personas — Warren (value), George (macro), Ray (systematic), Cathie (disruptive innovation / crypto ETFs)
- **OpenAI Agents SDK** — multi-turn agent loops with tool use, tracing, and a researcher sub-agent
- **Six MCP servers** — accounts, push notifications, market data, web fetch, Tavily search, and per-trader memory
- **Paper trading** — $10,000 starting balance per trader, 0.2% spread, SQLite persistence
- **Simulated or live market data** — works without API keys via a built-in price simulator; optional [Massive.com](https://massive.com/) integration
- **Dual dashboards** — Gradio (in-process DB reads) and a Vite/TypeScript frontend (FastAPI JSON API)
- **Full observability** — color-coded activity logs across trace, agent, function, generation, response, and account events
- **Scheduled trading cycles** — configurable interval; alternates between new trades and portfolio rebalancing

## Project Structure

```
Agentic-Trading-Floor/
├── app.py                 # Gradio dashboard entry point
├── pyproject.toml         # Python dependencies (managed with uv)
├── .env.example           # Environment variable template
├── .gitignore
├── LICENSE
├── README.md
├── backend/               # Trading engine, MCP servers, FastAPI API
├── demo/                  # Gradio UI components
├── frontend/              # Vite + TypeScript SPA
├── sandbox/               # Local experiments and scratch work
└── memory/                # Per-trader knowledge graphs (created at runtime)
```

## Prerequisites

| Tool | Purpose |
|------|---------|
| [Python 3.11+](https://www.python.org/) | Backend, agents, Gradio UI |
| [uv](https://docs.astral.sh/uv/) | Python package manager |
| [Node.js 18+](https://nodejs.org/) | Frontend dev server, MCP servers (Tavily, memory) |

External MCP servers are launched automatically by the agents via `uvx` and `npx` — no manual install required for those.

## Quick Start

### 1. Clone and configure

```bash
git clone <your-repo-url>
cd Agentic-Trading-Floor

cp .env.example .env
# Edit .env — at minimum set OPENAI_API_KEY and TAVILY_API_KEY
```

### 2. Install Python dependencies

```bash
uv sync
```

### 3. Reset trader accounts (first run)

```bash
uv run python -m backend.reset
```

### 4. Start the trading engine

```bash
uv run python -m backend.trading_floor
```

The scheduler runs all four traders on a cycle (default: every 60 minutes). Each cycle alternates between placing new trades and rebalancing existing positions.

### 5. Open a dashboard

Choose one of the two UIs below. Both read from the same `accounts.db` database.

---

## Dashboard Options

### Option A — Gradio UI (Python)

A four-panel dashboard built with Gradio, Plotly, and pandas. Reads `accounts.db` directly — no separate API server needed.

```bash
uv run python app.py
```

Opens in your browser automatically. Portfolio refreshes every 2 minutes; activity logs poll every 0.5 seconds.

**Entry points:** `app.py` → `demo/ui.py`

### Option B — Web Frontend (Vite + FastAPI)

A TypeScript SPA with portfolio charts (uPlot), holdings heatmap, activity log, and recent trades. Requires two terminals.

**Terminal 1 — API server** (from project root):

```bash
uv run uvicorn backend.api:app --port 8000
```

**Terminal 2 — Frontend dev server:**

```bash
cd frontend
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). The Vite dev server proxies `/api` requests to port 8000.

**Production build:**

```bash
cd frontend
npm run build    # output → frontend/dist/
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Trading Engine                            │
│  trading_floor.py → traders.py → MCP Servers → accounts.db  │
└─────────────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐          ┌──────────────────────────┐
│   Gradio UI     │          │  FastAPI (:8000)         │
│   app.py        │          │  backend/api.py          │
│   (direct DB)   │          │         │                │
└─────────────────┘          │         ▼                │
                             │  Vite Frontend (:5173)   │
                             └──────────────────────────┘
```

### MCP Servers

| Server | Role | Spawned via |
|--------|------|-------------|
| `accounts_server` | Buy/sell, balance, strategy | `uv run -m backend.accounts_server` |
| `push_server` | Pushover notifications | `uv run -m backend.push_server` |
| Market data | Live or simulated prices | `uvx mcp_massive` or `uv run -m backend.market_server` |
| `mcp-server-fetch` | Web page fetching | `uvx mcp-server-fetch` |
| `tavily-mcp` | Web search | `npx -y tavily-mcp@latest` |
| `mcp-memory-libsql` | Per-trader knowledge graph | `npx -y mcp-memory-libsql` |

### The Traders

| Name | Persona | Default model |
|------|---------|---------------|
| Warren Patience | Value investing (Buffett-inspired) | `gpt-5.4-mini` |
| George Bold | Macro / contrarian (Soros-inspired) | `gpt-5.4-mini` |
| Ray Systematic | Risk parity / diversification (Dalio-inspired) | `gpt-5.4-mini` |
| Cathie Crypto | Disruptive innovation / crypto ETFs (Wood-inspired) | `gpt-5.4-mini` |

Set `USE_MANY_MODELS=true` in `.env` to assign different LLMs per trader (GPT, DeepSeek, Gemini, Grok).

---

## Environment Variables

Copy `.env.example` to `.env` and configure:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes* | — | OpenAI API key for agent models |
| `TAVILY_API_KEY` | Yes | — | Tavily web search for the researcher agent |
| `MASSIVE_API_KEY` | No | — | Live market data; omit to use simulator |
| `PUSHOVER_USER` | No | — | Pushover user key for trade notifications |
| `PUSHOVER_TOKEN` | No | — | Pushover API token |
| `USE_MANY_MODELS` | No | `false` | Use a different LLM per trader |
| `DEEPSEEK_API_KEY` | If multi-model | — | DeepSeek models |
| `GOOGLE_API_KEY` | If multi-model | — | Gemini models |
| `GROK_API_KEY` | If multi-model | — | Grok models |
| `OPENROUTER_API_KEY` | If multi-model | — | OpenRouter-hosted models |
| `RUN_EVERY_N_MINUTES` | No | `60` | Scheduler interval |
| `RUN_EVEN_WHEN_MARKET_IS_CLOSED` | No | `false` | Run cycles outside market hours |

\*Required in default mode where all traders use OpenAI models.

---

## Common Commands

```bash
# Install / update dependencies
uv sync

# Reset all trader accounts to $10,000 and default strategies
uv run python -m backend.reset

# Run the trading scheduler
uv run python -m backend.trading_floor

# Launch Gradio dashboard
uv run python app.py

# Launch FastAPI backend for the web frontend
uv run uvicorn backend.api:app --port 8000

# Launch frontend dev server (from frontend/)
npm install && npm run dev
```

---

## Runtime Artifacts

These files are created automatically and are gitignored:

| Path | Description |
|------|-------------|
| `accounts.db` | SQLite database — portfolios, transactions, logs |
| `memory/{TraderName}.db` | Per-trader knowledge graph (libSQL) |

---

## Sandbox

The `sandbox/` directory is reserved for local experiments, one-off scripts, and prototyping. It is not used by the main application.

---

## License

This project is licensed under the [MIT License](LICENSE).
