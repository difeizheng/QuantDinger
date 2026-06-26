# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QuantDinger is a self-hosted quantitative trading platform combining AI-assisted research, Python-native strategies, backtesting, and live execution across crypto (Binance, OKX, Bybit, etc.), IBKR stocks, and MT5 forex. The platform consists of a Flask API backend, prebuilt Vue frontend, PostgreSQL database, Redis cache, and an MCP server for AI agent integration.

## Common Commands

### Docker (Primary Development)

```bash
# Start full stack
docker-compose up -d --build

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Restart services
docker-compose restart backend
docker-compose restart postgres redis

# Stop all services
docker-compose down
```

### Backend Development (Local, without Docker)

```bash
cd backend_api_python
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp env.example .env  # Edit .env with SECRET_KEY and other config
python run.py  # Dev server on http://localhost:5000
```

### Testing

```bash
cd backend_api_python
pip install pytest
pytest tests/ -v                    # Run all tests
pytest tests/test_specific.py -v    # Run single test file
pytest tests/ -k "test_name" -v     # Run tests matching pattern
```

### Frontend (Prebuilt, from Separate Repo)

The Vue source lives in [QuantDinger-Vue](https://github.com/brokermr810/QuantDinger-Vue). After building:

```bash
# Linux/macOS
export QUANTDINGER_VUE_SRC=/path/to/QuantDinger-Vue
./scripts/build-frontend.sh

# Windows PowerShell
robocopy C:\path\to\QuantDinger-Vue\dist frontend\dist /MIR

# Rebuild frontend container
docker-compose build frontend
docker-compose up -d frontend
```

### MCP Server

```bash
cd mcp_server
pip install -e .
quantdinger-mcp  # Local stdio mode
```

## Architecture Overview

### Backend Structure (`backend_api_python/`)

The Flask API uses Blueprints for route organization:

- **`app/routes/`** - REST endpoints organized by domain (auth, strategies, trading, AI, billing, etc.)
- **`app/services/`** - Business logic layer:
  - `ai_analysis/` - LLM integration, market analysis, NL→code generation
  - `backtest/` - Strategy backtesting engine
  - `live_trading/` - Exchange/broker execution adapters (crypto, IBKR, MT5)
  - `strategy/` - Strategy management, IndicatorStrategy and ScriptStrategy execution
  - `billing/` - Credits, membership, USDT payment
  - `notifications/` - Telegram, email, SMS, Discord, webhooks
- **`app/data_providers/`** - Market data fetchers (crypto, forex, stocks)
- **`app/data_sources/`** - Exchange/broker adapters (CCXT, yfinance, Twelve Data, etc.)
- **`app/utils/`** - Database helpers, authentication, caching, logging

### Strategy Development Modes

Two strategy authoring models are supported:

1. **IndicatorStrategy** - Dataframe-based Python scripts generating `buy`/`sell` signals. Best for indicator logic and visual prototyping.
2. **ScriptStrategy** - Event-driven `on_init(ctx)` / `on_bar(ctx, bar)` with explicit `ctx.buy()`, `ctx.sell()`. Best for stateful strategies and live execution.

Strategy examples are in `docs/examples/`.

### Agent Gateway & MCP

The Agent Gateway at `/api/agent/v1` allows AI agents (Cursor, Claude Code, Codex) to interact with QuantDinger. Key design principles:

- **Paper-only by default** - Trading-class tokens are paper-only unless both `paper_only=false` on token AND `AGENT_LIVE_TRADING_ENABLED=true` on server
- **Audit logging** - Every agent call is logged with route, scope class, status code, duration
- **Scopes** - R (read), W (write), B (backtest), T (trading, restricted)

The MCP server in `mcp_server/` wraps the Agent Gateway as Model Context Protocol tools. It supports both local stdio and remote HTTP transports.

### Data Flow

```
Market Data → Data Providers → Data Sources → Strategy Engine → Backtest/Live Execution
                                    ↓
                              AI Analysis → LLM Providers
                                    ↓
                              Agent Gateway → MCP Server → AI Clients
```

## Environment Configuration

Primary config: `backend_api_python/.env` (copy from `env.example`)

**Mandatory:**
- `SECRET_KEY` - JWT signing key (API refuses to start if still default)
- `ADMIN_USER` / `ADMIN_PASSWORD` - Initial admin credentials

**Optional but important:**
- `DATABASE_URL` - PostgreSQL connection
- `LLM_PROVIDER` / `OPENROUTER_API_KEY` / `OPENAI_API_KEY` - AI features
- `TWELVE_DATA_API_KEY` - Forex/commodities data
- `CACHE_ENABLED` - Redis caching (auto-set in Docker)
- `AGENT_LIVE_TRADING_ENABLED` - Enable live trading for agents (default: false)
- `BILLING_ENABLED` - Credits/membership system
- `USDT_PAY_ENABLED` - USDT payment integration

Optional root `.env` (same directory as `docker-compose.yml`) controls Compose-level settings:
- `FRONTEND_PORT` / `BACKEND_PORT` - Custom ports
- `IMAGE_PREFIX` - Docker mirror for slow pulls

## Key Services (Docker Compose)

| Service | Port | Description |
|---------|------|-------------|
| `frontend` | 8888 | Nginx serving Vue SPA from `frontend/dist/` |
| `backend` | 5000 | Flask API (gunicorn in production) |
| `postgres` | 5432 | PostgreSQL 16 |
| `redis` | 6379 | Cache layer (LRU, 128 MB) |

## Adding New Integrations

### New Data Source

1. Create `backend_api_python/app/data_sources/<name>.py` implementing `get_ticker(symbol)` and `get_kline(symbol, timeframe, limit)`
2. Register in `data_sources/factory.py`
3. If serving global market dashboard, add fetcher in `data_providers/` and wire into fallback chain

### New Exchange (Live Trading)

1. Create `backend_api_python/app/services/live_trading/<exchange>.py` inheriting from `BaseLiveTrading`
2. Implement `place_order`, `cancel_order`, `get_balance`, etc.
3. Register in `live_trading/factory.py`

## Default Credentials

- **Web UI**: http://localhost:8888
- **API**: http://localhost:5000
- **Admin user**: `quantdinger` / `123456` (change immediately in production)

## Cursor Skills

The repository includes a Cursor Skill at `.cursor/skills/quantdinger-agent-workflow/SKILL.md` that explains Agent Gateway internals, safety red lines, and verification procedures.

## Documentation

- **Strategy Development**: `docs/STRATEGY_DEV_GUIDE.md` (EN), `docs/STRATEGY_DEV_GUIDE_CN.md` (CN)
- **Agent Gateway**: `docs/agent/AGENT_ENVIRONMENT_DESIGN.md`, `docs/agent/AI_INTEGRATION_DESIGN.md`
- **Cloud Deployment**: `docs/CLOUD_DEPLOYMENT_EN.md`
- **Exchange Integrations**: `docs/IBKR_TRADING_GUIDE_EN.md`, `docs/MT5_TRADING_GUIDE_EN.md`
- **API Reference**: `docs/agent/agent-openapi.json`

## Troubleshooting

- **Backend exits immediately** - Check `SECRET_KEY` in `.env`, read `docker-compose logs backend`
- **Heatmap shows "暂无数据"** - Usually NaN in yfinance data; global JSON encoder sanitizes NaN/Inf to `null`
- **Redis connection refused** - Ensure `redis` service is running; set `CACHE_ENABLED=false` to fall back to in-memory cache
- **Many live strategies "start denied"** - Raise `STRATEGY_MAX_THREADS` in `.env` and restart API
