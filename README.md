# Gd-ipac-paypal
Goal-driven investing engine: turns plain-English financial goals into ETF portfolios, stress-tests them with Monte Carlo simulation, and funds or pays out via the PayPal sandbox. FastAPI + React.
# GD-IPCA™ Mark 2

**Goal-Driven Intent-to-Portfolio Compilation, Stochastic Forecasting & PayPal Execution Engine**

[![CI](https://github.com/<your-username>/<your-repo>/actions/workflows/ci.yml/badge.svg)](https://github.com/<your-username>/<your-repo>/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
![Python](https://img.shields.io/badge/Python-3.11-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-backend-009688)
![React](https://img.shields.io/badge/React-19-61dafb)

Describe a financial goal in plain English, such as *"I want $50,000 in 5 years for a house down payment"*. GD-IPCA compiles it into a structured goal, builds a matching ETF portfolio, stress-tests it with a Monte Carlo simulation, and funds or pays out through the **PayPal Developer Platform** (sandbox).

---

## Features

| Capability | What it does | Code |
|---|---|---|
| **Intent compiler** | Turns natural-language goals into a validated intent structure. Uses OpenAI or Gemini when an API key is set, and falls back to deterministic parsing when not. | `backend/app/routers/intent.py` |
| **Monte Carlo engine** | 5,000 vectorized simulation paths (default) across 4 market regimes (normal, bull, bear, crisis), with fat-tailed Student-t shocks. | `backend/app/services/monte_carlo.py` |
| **PayPal integration** | OAuth2 token exchange, Orders v2 (create and capture goal-funding deposits), and Payouts v1 (milestone disbursements). Runs against the PayPal sandbox, or in local simulation if no credentials are set. | `backend/app/services/paypal_service.py`, `backend/app/routers/paypal.py` |
| **Agentic planner** | Checks portfolio readiness against Monte Carlo confidence thresholds and prepares human-in-the-loop PayPal funding, payout, or rebalance actions. | `backend/app/routers/agent.py`, `backend/app/services/ai_agent.py` |
| **Portfolio and audit ledger** | Trades across 16 ETFs, drift-based rebalancing, and a persistent transaction history (MySQL 8 or SQLite fallback). | `backend/app/routers/portfolios.py`, `backend/app/routers/rebalance.py` |
| **Market data** | Yahoo Finance via `yfinance`, with local CSV fallback and caching. | `backend/app/services/market_data.py` |

## Tech Stack

- **Backend:** Python 3.11, FastAPI, SQLAlchemy + Alembic, NumPy / pandas, JWT auth (python-jose, bcrypt)
- **Database / cache:** MySQL 8.0 (production), SQLite (local), Redis 7 (optional)
- **Frontend:** React 19, Vite, Tailwind CSS 4, Recharts, Framer Motion
- **Infra:** Docker Compose, Nginx, GitHub Actions CI

## Project Structure

```
.
├── backend/
│   ├── app/
│   │   ├── core/          # settings, security (JWT, hashing)
│   │   ├── db/            # database session
│   │   ├── models/        # SQLAlchemy entities, intent schema
│   │   ├── routers/       # auth, intent, simulate, portfolios, rebalance, market, paypal, agent, health
│   │   └── services/      # monte_carlo, paypal_service, ai_agent, market_data, broker, cache
│   ├── alembic/           # database migrations
│   ├── scripts/           # backup/restore, deployment verification
│   ├── tests/             # integration and end-to-end tests
│   ├── .env.example       # environment template
│   └── requirements.txt
├── frontend/              # React + Vite app (src/, test/, nginx.conf, Dockerfile)
├── data/                  # assets, historical prices, market parameters, sample intents
├── docs/                  # INSTALLATION, API_DOCUMENTATION, ARCHITECTURE
├── docker-compose.yml
└── .github/workflows/ci.yml
```

---

## Quick Start

### 1. Configure environment

```bash
cp backend/.env.example backend/.env
```

Edit `backend/.env`. Every key below is **optional**; the app runs with local simulation if they are blank:

```env
PAYPAL_MODE=sandbox
PAYPAL_CLIENT_ID=your_sandbox_client_id
PAYPAL_CLIENT_SECRET=your_sandbox_client_secret
OPENAI_API_KEY=
GEMINI_API_KEY=
SECRET_KEY=<generate a long random string>
```

Get free sandbox credentials from the [PayPal Developer Dashboard](https://developer.paypal.com/dashboard/applications).
Generate a secret key with `python -c "import secrets; print(secrets.token_hex(32))"`.

> **Never commit `.env` files or real API keys.** Only `.env.example` belongs in the repo.

### 2. Run with Docker (recommended)

```bash
docker compose up --build -d
```

- App: <http://localhost>
- API docs (Swagger): <http://localhost:8000/api/docs>

### 3. Run locally without Docker

**Backend**
```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt email-validator
uvicorn app.main:app --reload --port 8000
```

**Frontend**
```bash
cd frontend
npm install
npm run dev
```
Open the URL Vite prints (usually <http://localhost:5173>). Set `VITE_API_URL` in `frontend/.env` if your backend is not on the default address (see `frontend/.env.example`).

---

## API Overview

All routes are prefixed with `/api/v1`. Full details are in [docs/API_DOCUMENTATION.md](docs/API_DOCUMENTATION.md).

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register`, `/auth/login`, `/auth/token`, `/auth/refresh`, `GET /auth/me` |
| Intent | `POST /intent/compile`, `GET /intent/seeds`, `/intent/goals` |
| Simulation | `POST /simulate/monte-carlo` |
| Portfolios | `GET/POST /portfolios`, `GET /portfolios/{id}`, `POST /portfolios/{id}/deposit`, `/withdraw`, `/trade`, `GET /portfolios/{id}/transactions`, `/orders` |
| Rebalance | `POST /rebalance/execute` |
| Market | `GET /market/universe`, `/market/quote/{symbol}`, `/market/history/{symbol}`, `POST /market/sync` |
| PayPal | `GET /paypal/status`, `POST /paypal/orders/create`, `/orders/{id}/capture`, `/payouts/create`, `GET /paypal/records` |
| Agent | `GET /agent/tools`, `POST /agent/plan`, `POST /agent/execute-paypal` |
| Health | `GET /health` |

## Demo Accounts

Seeded automatically on first run, **for local development only**. Change or remove these before any real deployment.

| Role | Email | Password |
|---|---|---|
| Demo investor | `demo@gdipca.io` | `Demo12345!` |
| Admin | `admin@gdipca.io` | `Admin12345!` |

## Testing

```bash
# Backend integration and end-to-end tests
cd backend && pytest tests/ -v

# Frontend unit tests
cd frontend && npm test
```

CI runs the backend tests on every push and pull request (`.github/workflows/ci.yml`).

## Security Notes

- Keep PayPal and LLM keys in `backend/.env` only. It is gitignored, so do not force-add it.
- The `SECRET_KEY` and database passwords in `docker-compose.yml` are placeholders. Replace them before deploying anywhere public.
- PayPal is wired to the **sandbox**. Review and test thoroughly before ever using live credentials.
- Trading runs in **paper mode** (`BROKER_MODE=paper`); no real orders are placed.

## Disclaimer

This project is for education and demonstration. It is **not financial advice**. Simulation outputs are model estimates, not guarantees of future returns.

## Documentation

- [Installation Guide](docs/INSTALLATION.md)
- [REST API Specification](docs/API_DOCUMENTATION.md)
- [Architecture Overview](docs/ARCHITECTURE.md)

## License

Released under the [MIT License](LICENSE).
