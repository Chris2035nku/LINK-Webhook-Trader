# LINK-Webhook-Trader
A production-grade Solana trading terminal built for speed. Routes swaps through Jupiter (Jito/Nozomi/Legacy), executes TradingView webhook signals automatically, and runs a full paper trading environment for strategy testing — all from a single dashboard with real-time balance tracking and a sleep monitor.


# Solana Webhook Trader

<img width="1036" height="1283" alt="image" src="https://github.com/user-attachments/assets/64d54829-2cac-4144-8db6-6043f6179c56" />



A production-grade Solana trading terminal. Routes swaps through Jupiter (Jito / Nozomi / Legacy), executes TradingView webhook signals automatically, and runs a full paper trading simulation environment — all from a single dashboard with real-time balance tracking, a sleep monitor, and a persistent trade log.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Startup](#startup)
6. [Application Pages](#application-pages)
7. [Swap Console](#swap-console)
8. [TradingView Webhook Integration](#tradingview-webhook-integration)
9. [Paper Thunderdome](#paper-thunderdome)
10. [Trade Log](#trade-log)
11. [Sleep Dashboard](#sleep-dashboard)
12. [Jupiter API Fallback Chain](#jupiter-api-fallback-chain)
13. [Jito Bundle System](#jito-bundle-system)
14. [Nozomi Routing](#nozomi-routing)
15. [Real-Time Dashboard (SSE)](#real-time-dashboard-sse)
16. [Caching Architecture](#caching-architecture)
17. [Telegram Notifications](#telegram-notifications)
18. [Safety & Risk Management](#safety--risk-management)
19. [API Reference](#api-reference)
20. [File Structure](#file-structure)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Browser Clients                      │
│   Splash → Swap Console | Sleep | Paper Thunderdome     │
└──────────────────────────┬──────────────────────────────┘
                           │  HTTP + SSE
┌──────────────────────────▼──────────────────────────────┐
│                   server.js  (Express)                   │
│                                                          │
│  ┌──────────────┐   ┌───────────────┐   ┌────────────┐  │
│  │ Swap Engine  │   │  TradingView  │   │   Paper    │  │
│  │  (manual)    │   │  Webhook Bot  │   │  Trading   │  │
│  └──────┬───────┘   └──────┬────────┘   └─────┬──────┘  │
│         │                  │                   │         │
│  ┌──────▼──────────────────▼──────────────┐    │         │
│  │           Jupiter API Client            │    │         │
│  │   Ultra API → Quote v6 → Lite API       │    │         │
│  └──────────────────────┬──────────────────┘    │         │
│                         │                       │         │
│  ┌──────────────────────▼────────────────┐      │         │
│  │         Submission Layer               │      │         │
│  │   Jito Bundles | Nozomi | Legacy RPC  │      │         │
│  └──────────────────────┬────────────────┘      │         │
│                         │                       │         │
│  ┌──────────────────────▼────────────┐  ┌───────▼──────┐ │
│  │       Helius RPC (Solana)         │  │   SQLite     │ │
│  │  Confirmation + Balance Fetches   │  │  Trade Log   │ │
│  └───────────────────────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                    ngrok Tunnel                          │
│      TradingView webhook → /api/tradingview/webhook      │
└─────────────────────────────────────────────────────────┘
```

**Stack:**
- **Backend:** Node.js (ESM), Express
- **Frontend:** Vanilla HTML / CSS / JavaScript — no framework, no build step
- **Database:** SQLite (trade log) + JSONL flat files (logs, live events)
- **RPC:** Helius (primary + secondary wallet RPC)
- **Routing:** Jupiter Aggregator — Ultra API → Quote v6 → Lite API fallback chain
- **Submission:** Jito block engine bundles, Nozomi private mempool, or legacy `sendRawTransaction`
- **Tunnel:** ngrok for exposing the TradingView webhook endpoint
- **Real-time UI:** Server-Sent Events (SSE)

---

## Prerequisites

- **Node.js** v18 or later (ESM modules required)
- **ngrok** on PATH (free account sufficient for webhook exposure)
- **Helius API key** — provides the Solana RPC URL with enough rate limit headroom
- **Solana wallet** — private key in base58 format, funded with USDC and SOL (for fees)
- **TradingView** Pro or higher (webhook alerts require a paid plan)
- Optional: Nozomi/Temporal API key, Telegram bot token

---

## Installation

```bash
git clone <repo>
cd solana-webhook-trader
npm install
cp .env.example .env
# fill in .env with your values
```

---

## Configuration

All configuration is in `.env`. No restart is required for runtime changes to `config/*.json`; everything else in `.env` requires a server restart to take effect.

> Never commit `.env`. Verify it is in `.gitignore`.

### Wallet & RPC

| Variable | Description |
|---|---|
| `PRIVATE_KEY` | Primary wallet private key — base58 encoded |
| `RPC_URL` | Helius RPC URL for Wallet 1 (embed your API key in the URL) |
| `SECONDARY_RPC_URL` | RPC URL for Wallet 2 (separate Helius key recommended) |
| `DEDICATED_SWAP_PORT` | HTTP port (default: `3210`) |
| `DEDICATED_SWAP_HOST` | Bind address (default: `127.0.0.1`) |
| `DEDICATED_SWAP_COMMITMENT` | Solana commitment level (default: `confirmed`) |

### Swap Guardrails

| Variable | Default | Description |
|---|---|---|
| `DEDICATED_SWAP_DEFAULT_SLIPPAGE_PCT` | `0.5` | Default slippage tolerance (%) applied to all quotes |
| `DEDICATED_SWAP_MAX_SLIPPAGE_PCT` | `1.5` | Hard cap — any swap above this is rejected before signing |
| `DEDICATED_SWAP_MAX_PRICE_IMPACT_PCT` | `0.5` | Max acceptable price impact from the Jupiter quote |
| `DEDICATED_SWAP_MAX_ROUTE_ACCOUNTS` | `24` | Rejects routes with more accounts than this limit |
| `DEDICATED_SWAP_MAX_QUOTE_AGE_MS` | `1200` | Rejects quotes older than this many milliseconds |
| `DEDICATED_SWAP_CONFIRM_TIMEOUT_MS` | `30000` | Max wait for on-chain confirmation |
| `DEDICATED_SWAP_CONFIRM_POLL_MS` | `800` | Polling interval during confirmation |
| `DEDICATED_SWAP_SOL_BUFFER_USD` | `0.25` | USD value of SOL always reserved when selling SOL |
| `DEDICATED_SWAP_SOL_FEE_RESERVE_SOL` | `0.01` | Hard SOL floor held for rent and transaction fees |
| `DEDICATED_SWAP_AUTO_CLAMP_SOL_INPUT` | — | Auto-reduce SOL sell amounts to the fee reserve floor |

### Dashboard / Logging

| Variable | Default | Description |
|---|---|---|
| `DEDICATED_TV_DEFAULT_WALLET` | `primary` | Wallet used for TradingView webhook trades |
| `DEDICATED_TV_LOG_LIMIT` | `200` | Max TradingView log entries returned by the API |
| `DEDICATED_TV_TRADE_LIMIT` | `500` | Max trade records returned by the API |
| `DEDICATED_DASHBOARD_EXECUTION_LIMIT` | `12` | Max executions sent per SSE dashboard push |
| `DEDICATED_DASHBOARD_LOG_LIMIT` | — | Max log lines sent per SSE push |
| `DEDICATED_DASHBOARD_TRADE_LIMIT` | — | Max trades sent per SSE push |
| `DEDICATED_DASHBOARD_STREAM_REFRESH_MS` | `1000` | Minimum interval between SSE state pushes |
| `DEDICATED_DASHBOARD_STREAM_KEEPALIVE_MS` | — | SSE keepalive heartbeat interval |

### Jupiter API

| Variable | Default | Description |
|---|---|---|
| `JUP_HTTP_TIMEOUT_MS` | `12000` | Jupiter API request timeout (ms) |
| `JUP_FORCE_LITE_ONLY` | `false` | Skip probing — always use Lite API |
| `JUP_ENABLE_PROBES` | `true` | Probe Ultra and v6 availability on first request |
| `JUP_ONLY_DIRECT_ROUTES` | `false` | Require single-hop routes only |
| `JUP_PREFER_DIRECT_ROUTES` | `false` | Prefer single-hop, allow multi-hop fallback |
| `JUP_DISABLE_ORCA_ROUTES` | `false` | Exclude Orca DEX from route planning |
| `JUP_DYNAMIC_COMPUTE_UNIT_LIMIT` | `true` | Auto-calculate compute unit budget |
| `JUP_PRIORITY_FEE_PRIORITY_LEVEL` | `veryHigh` | Fee priority level (`normal`, `high`, `veryHigh`) |
| `JUP_PRIORITY_FEE_MAX_LAMPORTS` | `1000000` | Cap on priority fee (lamports) |
| `JUP_PRIORITY_FEE_GLOBAL` | — | Apply priority fees globally across all instructions |
| `ULTRA_API_KEY` | — | Jupiter Ultra API key (for higher rate limits) |

### Jito Bundle Routing

| Variable | Default | Description |
|---|---|---|
| `USE_JITO` | `false` | Enable Jito bundle submission |
| `JITO_ENDPOINTS` | (all regions) | Comma-separated block-engine URLs; blank = all defaults |
| `JITO_TIP_SOL` | `0.001` | Base tip per bundle in SOL — floor for webhook trades |
| `JITO_MAX_ATTEMPTS` | `6` | Max bundle submission rounds before giving up |
| `JITO_TIMEOUT_MS` | `15000` | HTTP timeout per endpoint call (ms) |
| `JITO_CONFIRM_TIMEOUT_MS` | `90000` | Max time to wait for bundle confirmation (ms) |
| `JITO_POLL_INTERVAL_MS` | `2000` | Confirmation polling interval (ms) |
| `JITO_SHUFFLE_CANDIDATES` | `false` | Randomize endpoint order (default: cheapest-first) |
| `JITO_ALLOW_RPC_FALLBACK` | `false` | Fall back to legacy RPC if all Jito attempts fail |

**Default Jito block-engine endpoints (cheapest-first):**
- `tokyo.mainnet.block-engine.jito.wtf`
- `amsterdam.mainnet.block-engine.jito.wtf`
- `frankfurt.mainnet.block-engine.jito.wtf`
- `ny.mainnet.block-engine.jito.wtf`
- `mainnet.block-engine.jito.wtf`

> **Congestion tip:** Higher tip = faster bundle inclusion. During busy periods, raise `JITO_TIP_SOL` in `.env` as a floor, or override the lamport amount per-session in the console settings panel. The console tip setting persists across refreshes via `localStorage`.

### Nozomi Routing

| Variable | Description |
|---|---|
| `USE_NOZOMI` | Enable Nozomi private mempool routing |
| `NOZOMI_API_KEY` | Nozomi / Temporal API key |
| `NOZOMI_ORDER_URL` | Nozomi order submission endpoint URL |

### Telegram Notifications

| Variable | Default | Description |
|---|---|---|
| `DEDICATED_TV_TELEGRAM_NOTIFY_ENABLED` | `true` | Enable per-trade Telegram alerts |
| `DEDICATED_TV_TELEGRAM_BOT_TOKEN` | — | Bot token from @BotFather |
| `DEDICATED_TV_TELEGRAM_CHAT_ID` | — | Target chat or channel ID |
| `DEDICATED_TV_TELEGRAM_COMPACT_MODE` | — | Condensed single-line notifications |
| `DEDICATED_TV_TELEGRAM_TIMEOUT_MS` | `7000` | Send timeout per notification (ms) |
| `DEDICATED_TV_TELEGRAM_HOURLY_SUMMARY_ENABLED` | `true` | Send hourly P&L summaries |
| `DEDICATED_TV_TELEGRAM_HOURLY_SUMMARY_INTERVAL_MS` | `3600000` | Summary interval (default: 1 hour) |
| `DEDICATED_TV_TELEGRAM_HOURLY_SUMMARY_TIMEZONE` | — | Timezone for summary timestamps |
| `DEDICATED_TV_TELEGRAM_HOURLY_SUMMARY_SEND_ON_START` | — | Send a summary immediately on server start |
| `DEDICATED_TV_TELEGRAM_HOURLY_SUMMARY_SILENT` | — | Deliver summaries silently (no notification sound) |
| `DEDICATED_TV_PUBLIC_BASE_URL` | — | Public URL embedded in messages (Solscan links) |

### Runtime Config Files

These files in `config/` are read by the server at request time — changes take effect without a restart.

**`config/tradingview-config.json`** — Live webhook bot settings

```json
{
  "enabled": true,
  "cooldownMs": 6000,
  "wallet": "primary",
  "useJito": false,
  "useNozomi": false,
  "secret": "your-webhook-secret",
  "assetUsd": {
    "SOL":  250,
    "WETH": 250,
    "WBTC": 250,
    "HYPE": 250
  }
}
```

| Field | Description |
|---|---|
| `enabled` | Master kill switch — set `false` to halt all trading immediately |
| `cooldownMs` | Minimum ms between accepted webhook alerts |
| `wallet` | Active wallet for webhook trades (`primary` or `secondary`) |
| `useJito` | Runtime override for Jito routing (overrides `USE_JITO` in `.env`) |
| `useNozomi` | Runtime override for Nozomi routing |
| `secret` | Required in every incoming alert payload — mismatches return 403 |
| `assetUsd` | Per-asset USD allocation (buy cap per trade) |

**`config/tradingview-paper-config.json`** — Paper trading (up to 4 independent bots)

```json
{
  "bot1": { "enabled": true, "cooldownMs": 6000, "startingCashUsd": 1000, "secret": "paper-1" },
  "bot2": { "enabled": true, "cooldownMs": 6000, "startingCashUsd": 1000, "secret": "paper-2" },
  "bot3": { "enabled": false, "cooldownMs": 6000, "startingCashUsd": 1000, "secret": "paper-3" },
  "bot4": { "enabled": false, "cooldownMs": 6000, "startingCashUsd": 1000, "secret": "paper-4" }
}
```

---

## Startup

### Windows (recommended)

```cmd
START_DEDICATED_SWAP_3210.bat
```

The BAT file:
1. Kills any running `node server.js` and `ngrok` processes
2. Starts `node server.js`, piping output to `logs/tmp-server-start.out.log`
3. Launches ngrok on the configured port
4. Prints the local URL and the public ngrok webhook URL

### macOS

```bash
./START_DEDICATED_SWAP_3210.command
```

### Manual

```bash
node server.js        # terminal 1
ngrok http 3210       # terminal 2
```

---

## Application Pages

| Route | Page | Description |
|---|---|---|
| `/` | **Splash** | Zero-JS routing page — links to all sub-apps |
| `/console` | **Swap Console** | Full manual swap terminal with live dashboard |
| `/console?mode=swap-only` | **Swap Console** | Swap-only mode — hides the TradingView webhook panel |
| `/sleep` | **Sleep Dashboard** | Lightweight passive position monitor |
| `/paper-trading` | **Paper Thunderdome** | Paper trading quad-view for strategy testing |
| `/settings` | **Settings** | Server-side configuration editor |

The splash page has no JavaScript, no SSE connections, and makes no API or RPC calls — it is a pure navigation hub that adds zero overhead.

---

## Swap Console

The swap console (`/console`) is the primary manual trading interface.

### Route Protocols

| Protocol | Mechanism | Best For |
|---|---|---|
| **Legacy** | `sendRawTransaction` via Helius RPC | Low-congestion, minimal-cost swaps |
| **Jito** | Bundle posted to Jito block engine | MEV protection, high-priority guaranteed ordering |
| **Nozomi** | Private mempool via Nozomi/Temporal | Front-running protection without Jito bundling |

The protocol is selected via radio buttons and persists across refreshes via `localStorage`.

### Wallet Selection

- **Wallet 1 (Primary):** `PRIVATE_KEY` + `RPC_URL`
- **Wallet 2 (Secondary):** separate keypair + `SECONDARY_RPC_URL`

Each wallet has its own RPC connection and independent balance cache.

### Supported Tokens

SOL · USDC · WETH · WBTC · HYPE · PUMP · ZEC · TRUMP · FARTCOIN · RAY · TRX · JUP · TROLL · MET

| Symbol | Mint | Decimals |
|---|---|---|
| SOL / WSOL | `So11111111111111111111111111111111111111112` | 9 |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |
| WETH | `7vfCXTUXx5WJV5JADk17DUJ4ksgau7utNKj4b963voxs` | 8 |
| WBTC | `cbbtcf3aa214zXHbiAZQwf4122FBYbraNdFqgw4iMij` | 8 |
| HYPE | `98sMhvDwXj1RQi5c5Mndm3vPe9cBqPrbLaufMXFNMh5g` | 8 |

### Settings Panel

Click the gear icon (⚙) to open the settings drawer. All values persist across refreshes via `localStorage`:

- **Slippage %** — applied to all quotes from this session
- **Jito tip (lamports)** — live USD equivalent displayed next to the field
- **Nozomi tip (lamports)** — live USD equivalent displayed next to the field

### Quote Engine

- Auto-refreshes every **10 seconds** when a valid pair and non-zero amount are set
- A countdown timer in the UI shows seconds to next auto-refresh
- Debounced re-quote: **900ms** after amount changes, **300ms** after token/route selection changes
- Any manual input immediately cancels the countdown and triggers a new quote
- In-flight requests are cancelled via `AbortController` when a new request starts — no stale-response processing
- Quote display: output amount, price impact %, route plan hop count, exchange rate, and USD equivalents for both sides

### Quick Swap Presets

One-click preset buttons for common pairs (USDC → SOL, SOL → USDC, USDC → WETH, WETH → USDC, etc.). Amounts and token pairs are configurable. Selecting a preset populates both the from/to tokens and the amount, then immediately triggers a quote.

### Guardrails

Every swap validates the following before building any transaction:

| Guard | Check |
|---|---|
| Max slippage % | Setting must not exceed `DEDICATED_SWAP_MAX_SLIPPAGE_PCT` |
| Max price impact | Quote's price impact must not exceed `DEDICATED_SWAP_MAX_PRICE_IMPACT_PCT` |
| Max route accounts | Route plan must not exceed `DEDICATED_SWAP_MAX_ROUTE_ACCOUNTS` accounts |
| Quote freshness | Quote must be newer than `DEDICATED_SWAP_MAX_QUOTE_AGE_MS` |

If any check fails, execution is blocked with an inline error. No transaction is built or signed.

### Execution Flow

```
1.  Resolve spend amount
      → Clamp to wallet balance
      → Deduct SOL fee reserve if selling SOL
2.  Fetch Jupiter quote  (Ultra → v6 → Lite fallback)
3.  Validate all guardrails against the quote
4.  Build swap transaction
      → v0 VersionedTransaction  (Jito and Legacy)
      → Swap instructions + Nozomi tip injection  (Nozomi)
5.  Sign transaction with wallet keypair
6.  Submit via selected protocol:
      Jito    → sendBundle [tipTx, swapTx]
                 → poll getBundleStatuses + getSignatureStatuses in parallel
      Nozomi  → POST base64 tx to NOZOMI_ORDER_URL
                 → poll getSignatureStatuses
      Legacy  → sendRawTransaction
                 → poll getSignatureStatuses
7.  Parse settlement from confirmed on-chain tx meta
      → Actual input / output amounts (not quote estimates)
8.  Append record to executions.jsonl + SQLite trade log
9.  Notify all SSE clients: notifyDashboardState("post_execution")
10. 4-second delay → invalidateWalletBalanceCache → notifyDashboardState("post_execution_balance_refresh")
```

The 4-second delay in step 10 is intentional — it prevents the first post-execution SSE push from triggering a fresh RPC call on top of the confirmation traffic, which would cause 429s. The immediate push serves the cached (pre-swap) balance; the delayed invalidation fetches fresh balances once RPC traffic has settled.

---

## TradingView Webhook Integration

### Setup

1. In TradingView: **Alerts → Create Alert → Notifications → Webhook URL**
2. Set the webhook URL to your ngrok address:
   ```
   https://<subdomain>.ngrok-free.app/api/tradingview/webhook
   ```
3. Set the alert message to a JSON payload (see format below)
4. Configure the matching `secret` in the webhook settings panel or `config/tradingview-config.json`

> The ngrok URL changes on every server restart unless you have a reserved ngrok subdomain. Update your TradingView alert URL after each restart.

### Payload Format

**Buy signal:**
```json
{
  "action":    "BUY",
  "asset":     "WETH",
  "amountUsd": 100,
  "close":     "{{close}}",
  "wallet":    "primary",
  "secret":    "your-webhook-secret"
}
```

**Sell signal:**
```json
{
  "action":  "SELL",
  "asset":   "WETH",
  "percent": 100,
  "close":   "{{close}}",
  "wallet":  "primary",
  "secret":  "your-webhook-secret"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `action` | string | Yes | `BUY` or `SELL` (also `EXIT`, `TP`, `STOP`, `SWING_EXIT`, `TIME_EXIT`) |
| `asset` | string | Yes | Token symbol — must match a supported token |
| `amountUsd` | number | Buy | Fixed USD amount to purchase |
| `percent` | number | Sell | Percentage of current holding to sell (1–100) |
| `close` | string | No | TradingView `{{close}}` — fill price for paper trading |
| `wallet` | string | No | `primary` (default) or `secondary` |
| `signal` | string | No | Signal label for logging (e.g. `range_low`, `range_high`) |
| `secret` | string | Yes | Must match the configured webhook secret — requests without it return 403 |
| `paperBot` | number | Paper only | Bot ID 1–4 for paper trading endpoint |

### Execution Queue

Webhook signals are processed via a **serial promise queue** (`tradingViewRuntime.queuePromise`). Each signal waits for the previous one to fully complete before execution begins. This prevents concurrent swaps on the same wallet from double-spending or submitting conflicting transactions.

### Routing

TradingView trades use the same routing stack as manual swaps. Which protocol is active is determined by the runtime config (`useJito`, `useNozomi`) — overridable at any time without a restart by editing `config/tradingview-config.json`.

### Retry / Error Recovery

| Error | Recovery |
|---|---|
| Jupiter liquidity failure | Re-quote once with the failing DEX excluded |
| Insufficient funds | Re-quote with a reduced amount scaled to available balance |
| On-chain `InvalidInstructionData` | Re-quote with a fresh blockhash and retry |
| On-chain error `6024` / `0x1788` | Clear all DEX exclusions, re-quote, retry |
| `JITO_STALE_TRANSACTION` | Rebuild transaction with a fresh Jupiter quote + new blockhash, retry from attempt 1 |

### Pine Script Indicators

The repo ships two ready-to-use TradingView indicators in `pinescripts/Stratum/`:

- **`Stratum.pine`** — Primary signal indicator for the Stratum strategy

When creating a TradingView alert from a Pine script:
1. Set the **Webhook URL** to your ngrok endpoint
2. Set the **Message** field to a JSON payload using `{{strategy.order.action}}`, `{{strategy.order.comment}}`, and `{{close}}` placeholders

---

## Paper Thunderdome

A complete paper trading simulation running in parallel with the live system. Mirrors the live execution flow without spending real funds.

**URL:** `/paper-trading`

### Modes

| Mode | Description |
|---|---|
| **Single Bot** | Full paper console — logs, trades, P&L for one bot |
| **Quad View** | 4-panel iframe grid showing all 4 bots simultaneously |

Quad view uses iframes; each iframe detects `window.self !== window.top` and auto-hides the page header for a clean grid layout.

### Features

- Independent paper wallet per bot with configurable starting cash (USD)
- Fill price from TradingView `{{close}}` in the alert payload, or live market price if not provided
- Full trade history with P&L tracking per position (entry price vs. current price)
- Separate webhook endpoint: `POST /api/tradingview/paper-webhook`
- Include `"paperBot": 1` (or 2, 3, 4) in the payload to route to the correct bot
- Real-time sync across tabs via SSE (`GET /api/paper-trading/stream`)
- Config survives restarts via `config/tradingview-paper-config.json`
- Reset a paper wallet to starting cash: `POST /api/paper-trading/reset`

---

## Trade Log

Every live executed swap is recorded in two formats simultaneously.

### JSONL Flat Log

**File:** `logs/executions.jsonl`

Newline-delimited JSON, one record per execution. Human-readable and easily grepped. Each record contains:

- Signature, timestamp, wallet, source (`manual` / `webhook`)
- Input/output tokens and amounts — requested, effective, and actual on-chain (parsed from tx meta)
- Route protocol (`legacy` / `jito` / `nozomi`)
- Jito bundle ID, tip lamports, endpoint used, submit time (ms), confirm time (ms)
- Guardrail report — which checks passed and which would have blocked
- Settlement from parsed transaction meta — actual token delta, fee SOL paid
- P&L summary if entry price data is available

### SQLite Database

**File:** `data/trade-log/trade-log.sqlite`

Managed via `src/modules/trade-log-db.js`. Supports:
- Filtering by date range, wallet, token, source, action
- P&L aggregation by token or by date range
- Trade record export as CSV
- TradingView indicator backfill — syncs indicator state onto historical trade records

### Trade Log Dashboard

`trade-log-dashboard/` — standalone browser UI with filtering, sorting, and P&L visualization.

### Other Logs

| File | Contents |
|---|---|
| `logs/live-events.jsonl` | Dashboard live event stream — activity monitor and sleep dashboard feed |
| `logs/tradingview-logs.jsonl` | Webhook bot activity (received, accepted, rejected alerts) |
| `logs/tradingview-trades.jsonl` | Webhook trade records |
| `logs/tradingview-paper-logs.jsonl` | Paper webhook activity |
| `logs/tradingview-paper-trades.jsonl` | Paper trade records |

---

## Sleep Dashboard

A minimal passive monitoring page designed for secondary screens or devices.

**URL:** `/sleep`

- Current holdings with USD values across both wallets
- Recent execution log (last N confirmed trades)
- TradingView webhook activity feed
- Auto-updates via SSE (`GET /api/state/stream`)
- Loads stale cached state immediately on connect — zero blocking RPC calls on open
- Single SSE connection (does not compete with the console's connections)

The sleep dashboard adds zero RPC overhead on load. It reads from the most recently cached dashboard state snapshot.

---

## Jupiter API Fallback Chain

On the first quote request of each server session, Jupiter API availability is probed in order:

```
1.  Ultra API  →  https://api.jup.ag/ultra/v1/order
    If 404 / unavailable ──────────────────────────────┐
                                                        ▼
2.  Quote API v6  →  https://quote-api.jup.ag/v6        │
    If unreachable ──────────────────────────────────┐   │
                                                     ▼   ▼
3.  Lite API  →  https://lite-api.jup.ag  (always available)
```

The first successful tier is used for the entire server session — no re-probing on subsequent requests. Set `JUP_FORCE_LITE_ONLY=true` to skip probing entirely.

**Nozomi path:** Nozomi swaps use the Jupiter **swap instructions** endpoint (not the pre-built swap transaction endpoint). This returns raw instructions as a v0 `VersionedTransaction` without a pre-signed wrapper, allowing the Nozomi tip instruction (`SystemProgram.transfer` to the Nozomi tip account) to be injected into the instruction list before signing.

---

## Jito Bundle System

Implemented in `src/Jito/jito.js` (`JitoJsonRpcClient`) and `src/Jito/transport.js`.

### Bundle Construction

1. A random Jito validator tip account is selected from the configured set
2. A `SystemProgram.transfer` tip instruction is compiled into a v0 `VersionedTransaction`, signed with the wallet keypair, and base64-encoded (`tipTx`)
3. The swap transaction (pre-signed, base64-encoded) is appended: `[tipTx, swapTx]`
4. Jito requires base64 encoding for bundle payloads (base58 is deprecated)

A fresh tip transaction with a fresh blockhash is built for **each retry attempt** — this prevents the tip transaction itself from going stale across multiple rounds.

### Submission Strategy

```
attempt = 1 .. maxAttempts:

  candidates = endpoints not currently in cooldown
    (if all are cooled: use all anyway)

  tipLamports = baseTip × tipMultiplier (capped at baseTip × maxTipMultiplier)

  Build fresh tip tx for this round

  roundHadNon400Failure = false

  For each endpoint (cheapest-first by default):
    POST sendBundle
    → bundleId returned → SUCCESS (return immediately)
    → 429 → tipMultiplier × 1.15, cooldown 4s, roundHadNon400Failure = true
    → 5xx / 408 → cooldown 8s, roundHadNon400Failure = true
    → 400 → cooldown 4s  (roundHadNon400Failure stays false)

  If !roundHadNon400Failure and round was non-empty:
    consecutiveAllBadRequestRounds++
  Else:
    consecutiveAllBadRequestRounds = 0

  If consecutiveAllBadRequestRounds >= 2:
    throw Error("JITO_STALE_TRANSACTION: ...")  ← caller rebuilds with fresh quote

  Exponential backoff: min(15s, 2^attempt) + random jitter
```

### Tip Escalation

- Tip multiplier starts at 1.0× and escalates +15% per 429 received
- Capped at `maxTipMultiplier` (default: 3.0×)
- On congested networks the tip auto-escalates to stay competitive for block space without manual intervention

### Stale Transaction Detection

If **all endpoints** return `400 Bad Request` across **2 consecutive rounds**, `sendBundle` throws:

```
Error: JITO_STALE_TRANSACTION: bundle rejected with 400 on all endpoints across multiple attempts — swap transaction blockhash has likely expired
```

The caller (`executeExactInSwap`) catches this and re-runs the entire flow from step 1 — fresh Jupiter quote, new blockhash, new transaction — before retrying submission. This prevents the bot from burning all 6 attempts on a transaction whose blockhash expired mid-retry.

### Confirmation

`JitoJsonRpcClient.confirm()` polls **both** sources in parallel per tick:
- `getBundleStatuses` via Jito's JSON-RPC endpoint (fastest when available)
- `getSignatureStatuses` via the Helius RPC connection

First source to return `confirmed` or `finalized` wins. Falls back to RPC-only if the Jito endpoint is unreachable during confirmation.

### Default Validator Tip Accounts

```
DfXygSm4jCyNCybVYYK6DwvWqjKee8pbDmJGcLWNDXjh
ADuUkR4vqLUMWXxW9gh6D6L8pMSawimctcNZ5pGwDcEt
3AVi9Tg9Uo68tJfuvoKvqKNWKkC5wPdSSdeBnizKZ6jT
HFqU5x63VTqvQss8hp11i4wVV8bD44PvwucfZ2bU7gRe
ADaUMid9yfUytqMBgopwjb2DTLSokTSzL1zt6iGPaS49
Cw8CFyM9FkoMi7K7Crf6HNQqf4uEMzpKw6QNghXLvLkY
DttWaMuVvTiduZRnguLF7jNxTgiMBZ1hyAumKUiL2KRL
96gYZGLnJYVFmbjzopPSU6QiEV5fGqZNyN9nmNhvrZU5
```

A random validator is selected per bundle attempt.

---

## Nozomi Routing

Routes transactions through a private mempool to prevent front-running and sandwich attacks.

### How It Works

1. Jupiter quote is fetched normally (Ultra / v6 / Lite)
2. Transaction is built via Jupiter's **swap instructions** endpoint — returns raw swap instructions without a pre-signed wrapper
3. A `SystemProgram.transfer` tip instruction to the Nozomi tip account is injected into the instruction set
4. All instructions are compiled into a v0 `VersionedTransaction` and signed with the wallet keypair
5. The signed transaction is base64-encoded and POSTed to `NOZOMI_ORDER_URL` with `Authorization: Bearer <NOZOMI_API_KEY>`
6. The endpoint returns a signature; `getSignatureStatuses` polling begins immediately

### Per-Direction Config

For TradingView webhook trades, Nozomi buy routing and Nozomi sell routing can be toggled independently:
- Route buys through Nozomi (front-run protection on entry) while sells use Legacy or Jito
- Or enable Nozomi for both directions

---

## Real-Time Dashboard (SSE)

All browser clients receive live state pushes via Server-Sent Events.

### Connection Architecture

Each page opens one SSE connection:
- **Swap Console & Sleep** → `GET /api/state/stream`
- **Paper Trading** → `GET /api/paper-trading/stream`

Browser limit: 6 connections per origin under HTTP/1.1. One SSE connection per tab leaves 5 connections for regular HTTP requests.

### State Push Lifecycle

```
Event fires (trade execution, config update, log entry, etc.)
  → notifyDashboardState(reason)
  → 150ms debounce  (coalesces rapid events into a single push)
  → getDashboardStateSnapshot()
      → Check 15s TTL cache
          → Cache is fresh: serve stale value immediately (zero RPC)
          → Cache is stale: buildDashboardState()
              → getWalletBalances(wallet1)  [2 RPC calls]
              → getWalletBalances(wallet2)  [2 RPC calls]
              → readRecentExecutions()       [JSONL read]
              → readTradingViewLogs()        [JSONL read]
              → Price data from in-memory cache
  → Serialize state as JSON
  → Broadcast to all connected SSE clients
```

On new client connect, the most recent cached state snapshot is pushed immediately — clients see content instantly without waiting for an RPC round-trip.

### Keepalive

SSE keepalive comments (`:keepalive`) are sent on `DEDICATED_DASHBOARD_STREAM_KEEPALIVE_MS` interval to prevent proxy and firewall connection drops from idle timeouts.

---

## Caching Architecture

The server maintains independent caches to minimize Helius RPC consumption and avoid 429 rate-limit errors.

| Cache | TTL | Scope | Notes |
|---|---|---|---|
| **Token accounts** | 8s | Per owner pubkey | `getParsedTokenAccountsByOwner` result — in-flight dedup via `walletBalanceFetchPromises` Map |
| **Wallet balances** | 8s | Per wallet | Composite snapshot (SOL + all tokens); invalidated 4s post-execution |
| **Dashboard state** | 15s | Global | Stale value preserved on invalidation — stale-while-revalidate |
| **Paper trading state** | 15s | Global | Same stale-while-revalidate pattern |
| **SOL price** | Configurable | Global | Feeds USD tip display and P&L calculations |
| **Token prices** | Configurable | Per token | Coinbase + Jupiter price feeds |

### Balance Fetch Optimization

A previous implementation called `getParsedTokenAccountsByOwner` 13 times concurrently (once per tracked token) × 2 RPC calls each = **26 concurrent Helius calls per wallet refresh**. Under `force: true`, in-flight deduplication was bypassed, making each of the 13 calls independent. This was the root cause of pre- and post-swap 429 storms.

The current implementation:

```javascript
const [lamports, tokenAccounts] = await Promise.all([
  connection.getBalance(owner, "confirmed"),          // 1 RPC call
  getParsedTokenAccountsForOwner(connection, owner),  // 1 RPC call
]);
const totals = mapMintUiAmountTotals(tokenAccounts);
// All 13 token balances resolved in-memory from the single response
```

**Total: 2 RPC calls per wallet** regardless of how many tokens are tracked.

### In-Flight Deduplication

`walletBalanceFetchPromises` is a `Map` keyed by wallet address. If a balance fetch is already in flight for a wallet when a second caller requests it, the second caller receives the same promise — not a new RPC call. This prevents duplicate requests when multiple dashboard events fire within the same 8s cache window.

### Post-Execution Balance Refresh Delay

After a swap confirms, balance cache invalidation is delayed **4 seconds** via `setTimeout`. Without this delay, the immediate post-execution SSE push would trigger a fresh 2-call RPC burst on top of the confirmation traffic, reliably hitting rate limits. The first post-execution push serves the stale (pre-swap) cached balance; the delayed invalidation fetches fresh balances once confirmation RPC traffic has settled.

---

## Telegram Notifications

When `DEDICATED_TV_TELEGRAM_NOTIFY_ENABLED=true`, the server sends:

- **Trade execution alerts** — token pair, amounts (in and out), P&L vs. last buy price for that token, route protocol, Solscan transaction link
- **Hourly P&L summaries** — aggregated performance over the configured interval, timezone-aware
- **Critical error alerts** — failures in the webhook execution path requiring attention

Configure `DEDICATED_TV_PUBLIC_BASE_URL` to include clickable Solscan deep links in every trade notification.

**Setup:**
1. Create a Telegram bot via [@BotFather](https://t.me/BotFather) and copy the token
2. Get your chat ID — send `/start` to your bot, then: `https://api.telegram.org/bot<TOKEN>/getUpdates`
3. Set `DEDICATED_TV_TELEGRAM_BOT_TOKEN` and `DEDICATED_TV_TELEGRAM_CHAT_ID` in `.env`
4. Set `DEDICATED_TV_TELEGRAM_NOTIFY_ENABLED=true`

---

## Safety & Risk Management

| Guard | Behavior |
|---|---|
| **Webhook secret** | Requests without the correct `secret` field return 403 immediately |
| **Master kill switch** | Set `"enabled": false` in `config/tradingview-config.json` to halt all webhook trading instantly — no restart required |
| **Cooldown enforcement** | Signals arriving faster than `cooldownMs` are ignored |
| **Slippage cap** | Any swap whose slippage would exceed `DEDICATED_SWAP_MAX_SLIPPAGE_PCT` is rejected before signing |
| **Price impact guard** | Quotes with price impact above `DEDICATED_SWAP_MAX_PRICE_IMPACT_PCT` are rejected |
| **Route account limit** | Routes with too many accounts (potential transaction bloat) are rejected |
| **Quote expiry** | Stale quotes older than `DEDICATED_SWAP_MAX_QUOTE_AGE_MS` are discarded and re-fetched |
| **SOL fee buffer** | `DEDICATED_SWAP_SOL_FEE_RESERVE_SOL` SOL is always held back for transaction fees — sells are auto-clamped |
| **Balance clamping** | Order size is scaled down to available balance rather than failing with an error |
| **Serial execution queue** | Webhook signals queue serially — no concurrent swaps on the same wallet |
| **Stale transaction detection** | Expired Jito transactions are detected and rebuilt with fresh quotes rather than burning all retry attempts |

---

## API Reference

### Swap

| Method | Route | Description |
|---|---|---|
| `POST` | `/quote/exact-in` | Fetch a Jupiter quote (no execution) |
| `POST` | `/swap/exact-in` | Execute a manual swap |
| `POST` | `/transfer` | Wallet-to-wallet token transfer |

### Dashboard State

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/state` | Current dashboard state snapshot (JSON) |
| `GET` | `/api/state/stream` | SSE stream — pushes state to client on every change |
| `GET` | `/api/balances` | Current wallet balances (JSON) |
| `GET` | `/health` | Server health check |

### TradingView Webhook

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/tradingview/webhook` | Live trading webhook receiver |
| `GET` | `/api/tradingview/config` | Read current webhook configuration |
| `POST` | `/api/tradingview/config` | Update webhook configuration (no restart required) |
| `GET` | `/api/tradingview/logs` | Recent webhook activity logs |
| `GET` | `/api/tradingview/trades` | Recent webhook trade records |
| `GET` | `/api/tradingview/executions` | Recent webhook execution records |
| `POST` | `/api/tradingview/logs/clear` | Clear the webhook activity log |
| `GET` | `/api/ngrok-url` | Current public ngrok webhook URL |

### Paper Trading

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/tradingview/paper-webhook` | Paper trading webhook receiver |
| `GET` | `/api/paper-trading/stream` | SSE stream for paper trading state |
| `GET` | `/api/paper-trading/state` | Current paper trading state snapshot |
| `GET` | `/api/paper-trading/config` | Read paper trading configuration |
| `POST` | `/api/paper-trading/config` | Update paper trading configuration |
| `POST` | `/api/paper-trading/reset` | Reset a paper wallet to starting cash |

### Trade Log

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/trade-log` | Paginated trade log from SQLite |
| `GET` | `/api/trade-log/summary` | Aggregated P&L summary |
| `GET` | `/api/trade-log/export.csv` | Export full trade history as CSV |

### Server

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/restart` | Graceful server restart |

---

## File Structure

```
├── server.js                          # Express server — routing, swap engine, webhook bot, SSE, caching
├── .env                               # Environment configuration  (not committed)
├── .env.example                       # Template — all supported variables with defaults
├── package.json
├── START_DEDICATED_SWAP_3210.bat      # Windows one-click startup
├── START_DEDICATED_SWAP_3210.command  # macOS one-click startup
│
├── src/
│   ├── Jito/
│   │   ├── jito.js                    # JitoJsonRpcClient — bundle build, multi-endpoint send, confirm
│   │   └── transport.js               # submitSignedTransactionViaJito — top-level Jito entry point
│   ├── Jupiter/
│   │   └── jupiter.js                 # Jupiter quote + transaction build (Ultra / v6 / Lite APIs)
│   └── modules/
│       └── trade-log-db.js            # SQLite trade log — read, write, query, aggregate
│
├── public/
│   ├── splash.html                    # Zero-JS routing splash page
│   ├── index.html                     # Swap Console
│   ├── sleep.html                     # Sleep Dashboard
│   ├── paper-trading.html             # Paper Thunderdome
│   ├── settings.html                  # Settings editor
│   ├── app.js                         # Swap Console — quote engine, SSE client, all UI logic
│   ├── paper-trading.js               # Paper trading UI
│   ├── settings.js                    # Settings page logic
│   └── styles.css                     # Shared stylesheet
│
├── config/
│   ├── tradingview-config.json        # Live webhook runtime config (hot-reloaded, no restart)
│   └── tradingview-paper-config.json  # Paper trading runtime config (hot-reloaded, no restart)
│
├── logs/
│   ├── executions.jsonl               # Live swap execution records — one JSON object per line
│   ├── live-events.jsonl              # Dashboard live event stream
│   ├── tradingview-logs.jsonl         # Webhook bot activity log
│   ├── tradingview-trades.jsonl       # Webhook trade records
│   ├── tradingview-paper-logs.jsonl   # Paper webhook activity log
│   └── tradingview-paper-trades.jsonl # Paper trade records
│
├── data/
│   └── trade-log/
│       └── trade-log.sqlite           # SQLite trade database — live and paper trades
│
├── trade-log-dashboard/               # Standalone trade log browser UI
│   ├── index.html
│   ├── app.js
│   └── styles.css
│
└── pinescripts/
    └── Stratum/
        └── Stratum.pine               # TradingView Pine Script — Stratum signal indicator
```
