# Nocturne Trading Agent

Nocturne is an AI-powered trading loop for Hyperliquid perpetuals. It collects live market data from TAAPI, summarizes account health, prompts an LLM for the next action, and executes the resulting trades with risk controls.

## Quick Facts
- Runs on Hyperliquid (mainnet or testnet) using your wallet keys.
- Pulls technical indicators from TAAPI and streams account/order state each cycle.
- Leverages OpenRouter-hosted models with structured JSON outputs and TAAPI tool calls.
- Ships with a lightweight HTTP API (`/diary`, `/logs`) for live monitoring.

## Requirements
- Python 3.11+
- Poetry (or compatible virtual environment)
- Accounts + keys:
  - `TAAPI_API_KEY`
  - `HYPERLIQUID_PRIVATE_KEY` or `LIGHTER_PRIVATE_KEY` (or a mnemonic)
  - `OPENROUTER_API_KEY`
- Recommended: `.env` based on `.env.example`

## How It Works
1. **Ingest** – Every interval, the agent fetches Hyperliquid balance, positions, fills, funding, and open orders.  
2. **Enrich** – TAAPI supplies intraday (5m) and higher-timeframe (4h) indicators for each configured asset.  
3. **Decide** – The LLM receives a compact JSON context and returns a schema-checked plan: buy, sell, or hold with sizing, TP/SL, and rationale.  
4. **Execute** – Orders are sent through the Hyperliquid SDK with retries, precision rounding, and follow-up reconciliation.  
5. **Observe** – Prompts, responses, and trade diary entries are persisted and exposed through the built-in aiohttp server.

See `docs/ARCHITECTURE.md` for a subsystem deep dive and the accompanying architecture diagram.

## Project Layout
- `src/main.py` – CLI entry point and orchestration loop.
- `src/agent/decision_maker.py` – LLM wrapper, tool calling, and response sanitation.
- `src/indicators/taapi_client.py` – TAAPI helper with retry/backoff.
- `src/trading/hyperliquid_api.py` – Exchange facade with async retries and order helpers.
- `src/config_loader.py` – Centralized `.env` parsing and defaults.

## Getting Started
```bash
poetry install
cp .env.example .env   # fill in keys, ASSETS, INTERVAL
poetry run python src/main.py --assets BTC ETH --interval 1h
```

### Local API
- `GET /diary?limit=200` – recent JSONL diary entries as JSON or raw text.
- `GET /logs?path=llm_requests.log&limit=2000` – tails any local log file.
Configure the bind address with `API_HOST` and port with `API_PORT` / `APP_PORT` (defaults to `0.0.0.0:3000`).

### Docker
```bash
docker build --platform linux/amd64 -t trading-agent .
docker run --rm -p 3000:3000 --env-file .env trading-agent
# curl http://localhost:3000/diary
```

## Deployment (EigenCloud)
1. Install the EigenX CLI:  
   - macOS/Linux: `curl -fsSL https://eigenx-scripts.s3.us-east-1.amazonaws.com/install-eigenx.sh | bash`  
   - Windows: `curl -fsSL https://eigenx-scripts.s3.us-east-1.amazonaws.com/install-eigenx.ps1 | powershell -`
2. Authenticate: `docker login` then `eigenx auth login` (or `eigenx auth generate --store`).
3. Deploy from the repo root:
   ```bash
   cp .env.example .env
   eigenx app deploy
   ```
4. Monitor with `eigenx app info --watch` and `eigenx app logs --watch`. Upgrade using `eigenx app upgrade <app-name>`.

## Live Agents
- GPT-5 Pro — [Portfolio](https://hypurrscan.io/address/0xa049db4b3dfcb25c3092891010a629d987d26113) | [Logs](https://35.190.43.182/logs/0xC0BE8E55f469c1a04c0F6d04356828C5793d8a9D)  
- DeepSeek R1 (paused) — [Portfolio](https://hypurrscan.io/address/0xa663c80d86fd7c045d9927bb6344d7a5827d31db) | [Logs](https://35.190.43.182/logs/0x4da68B78ef40D12f378b8498120f2F5A910Af1aD)  
- Grok 4 (paused) — [Portfolio](https://hypurrscan.io/address/0x3c71f3cf324d0133558c81d42543115ef1a2be79) | [Logs](https://35.190.43.182/logs/0xe6a9f97f99847215ea5813812508e9354a22A2e0)

## Disclaimer
Trading carries risk. This project is unaudited and provided as-is—use at your own discretion.
