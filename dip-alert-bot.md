# Dip Alert Bot

A personal investment alert bot that monitors Mutual Fund NAVs and Stock prices, and notifies you on Telegram when a configured dip threshold is breached — so you can invest at better entry points instead of blindly following a fixed SIP date.

---

## Table of Contents

1. [Why This Project](#1-why-this-project)
2. [How It Works — Big Picture](#2-how-it-works--big-picture)
3. [Concepts You Need to Understand First](#3-concepts-you-need-to-understand-first)
   - 3.1 [How MF NAVs Work](#31-how-mf-navs-work)
   - 3.2 [How Stock Prices Work](#32-how-stock-prices-work)
   - 3.3 [Where the Data Comes From](#33-where-the-data-comes-from)
   - 3.4 [What a Telegram Bot Is](#34-what-a-telegram-bot-is)
4. [Project Architecture](#4-project-architecture)
5. [Tech Stack](#5-tech-stack)
6. [Project Structure](#6-project-structure)
7. [Deep Dive: Each Component](#7-deep-dive-each-component)
   - 7.1 [Config Layer](#71-config-layer)
   - 7.2 [Data Fetchers](#72-data-fetchers)
   - 7.3 [Threshold Engine](#73-threshold-engine)
   - 7.4 [Notifier](#74-notifier)
   - 7.5 [Scheduler](#75-scheduler)
   - 7.6 [CLI / Entry Point](#76-cli--entry-point)
8. [Data Flow — Step by Step](#8-data-flow--step-by-step)
9. [Edge Cases and How to Handle Them](#9-edge-cases-and-how-to-handle-them)
10. [Environment Setup](#10-environment-setup)
11. [How to Run](#11-how-to-run)
12. [Task Checklist](#12-task-checklist)
13. [Future Extensions (SDE2/SDE3)](#13-future-extensions-sde2sde3)

---

## 1. Why This Project

A SIP (Systematic Investment Plan) deducts money on a fixed calendar date every month. The NAV or stock price on that specific date may not be the best entry point for that month — it could be at a local high.

This bot flips the approach:

- You define a **reference price** (e.g., last month's NAV, or your average buy price)
- You define a **dip threshold** (e.g., 3%)
- The bot monitors the price daily
- The moment the price drops by that threshold from your reference, it **sends you a Telegram alert**
- You then manually place the order at that favorable price

This is not about beating the market over the long run — it's about being deliberate with your entry points rather than leaving it to calendar luck.

---

## 2. How It Works — Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                        Scheduler                         │
│        Runs daily: stocks at 3:30 PM, MF at 11 PM       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                      Data Fetcher                        │
│   MF: AMFI API (mftool)  │  Stocks: Yahoo Finance       │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    Threshold Engine                      │
│   current_price <= reference_price * (1 - dip% / 100)  │
└───────────────────────────┬─────────────────────────────┘
                            │ (if threshold breached)
                            ▼
┌─────────────────────────────────────────────────────────┐
│                       Notifier                           │
│              Sends Telegram message to you               │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Concepts You Need to Understand First

### 3.1 How MF NAVs Work

- **NAV** (Net Asset Value) is the per-unit price of a mutual fund.
- It is calculated **once per day**, after market close (~3:30 PM IST).
- AMFI (Association of Mutual Funds in India) publishes all NAVs on their website at around **11 PM IST** every business day.
- NAV = (Total Assets of Fund - Liabilities) / Total Units Outstanding
- When you invest ₹1000 in a fund with NAV = ₹50, you get 20 units.
- If NAV rises to ₹55, your investment is worth ₹1100.

**Key implication for this bot:** MF NAVs only need to be fetched **once per day**, after 11 PM.

### 3.2 How Stock Prices Work

- Stock prices update **every second** during market hours (9:15 AM – 3:30 PM IST, Mon–Fri).
- After 3:30 PM, the price is fixed until next trading day (closing price).
- For this bot, we fetch the **end-of-day closing price** once daily (after 3:30 PM).
- This avoids the complexity of real-time intraday tracking.

**Key implication:** Stocks should be fetched **once per day after 3:30 PM IST** — not real-time.

### 3.3 Where the Data Comes From

#### AMFI (for Mutual Funds)
- AMFI publishes a public text file with all NAVs daily.
- URL: `https://www.amfiindia.com/spages/NAVAll.txt`
- The `mftool` Python library wraps this cleanly.
- It is **completely free**, no API key needed.
- Example:

```python
from mftool import Mftool
mf = Mftool()
data = mf.get_scheme_quote('119598')  # scheme code for a fund
print(data['nav'])  # → '152.4300'
```

To find the scheme code for a fund, search on AMFI's website or use `mf.get_scheme_codes()`.

#### Yahoo Finance (for Stocks)
- `yfinance` is an unofficial Python library that scrapes Yahoo Finance.
- No API key required. Free.
- NSE stocks use the suffix `.NS` (e.g., `RELIANCE.NS`)
- BSE stocks use `.BO` (e.g., `RELIANCE.BO`)
- Example:

```python
import yfinance as yf
ticker = yf.Ticker("RELIANCE.NS")
price = ticker.fast_info['last_price']
print(price)  # → 2934.5
```

**Caveat:** `yfinance` is unofficial and can break if Yahoo changes their API. For a personal bot this is acceptable. For production use, you'd pay for a proper data feed.

### 3.4 What a Telegram Bot Is

- A Telegram Bot is a special Telegram account operated by a program, not a person.
- Bots are created via `@BotFather` on Telegram, which gives you a **Bot Token** (a secret string).
- Your Python script uses this token to call Telegram's Bot API over HTTPS.
- The API is a simple REST API — sending a message is just a POST request.
- `python-telegram-bot` library wraps this into clean Python functions.

**How you receive alerts:**
1. You create the bot via BotFather → get a token.
2. You start a chat with your bot on Telegram.
3. You find your personal **chat_id** (a numeric ID Telegram uses to identify your conversation).
4. Your script sends messages to that chat_id using the token.

You never need to run a server to receive messages — you're only **sending** alerts to yourself, not receiving commands (at this SDE1 stage).

---

## 4. Project Architecture

```
dip-alert-bot/
├── config.yaml              ← Your watchlist: instruments, thresholds
├── main.py                  ← Entry point, wires everything together
├── scheduler.py             ← Schedules when fetchers run
├── fetchers/
│   ├── __init__.py
│   ├── mf_fetcher.py        ← Fetches NAV from AMFI via mftool
│   └── stock_fetcher.py     ← Fetches price from Yahoo Finance via yfinance
├── engine.py                ← Threshold comparison logic
├── notifier.py              ← Telegram message sender
├── models.py                ← Data classes: Instrument, AlertResult
├── logger.py                ← Structured logging setup
├── .env                     ← BOT_TOKEN and CHAT_ID (never commit this)
├── .env.example             ← Template showing what .env needs
├── requirements.txt         ← Python dependencies
└── README.md                ← Quick start guide
```

This is a **flat, single-process** architecture. No database, no web server, no queue. Just a Python process that wakes up on a schedule, does its work, and goes back to sleep.

---

## 5. Tech Stack

| Component | Tool | Why |
|---|---|---|
| Language | Python 3.11+ | Best library ecosystem for finance data |
| MF data | `mftool` | Purpose-built AMFI wrapper, free |
| Stock data | `yfinance` | Easiest Yahoo Finance client, free |
| Telegram | `python-telegram-bot` v20+ | Well-maintained, async-capable |
| Scheduling | `APScheduler` | Cron-like scheduling inside Python |
| Config | `PyYAML` | Human-readable watchlist config |
| Env vars | `python-dotenv` | Keeps secrets out of code |
| Logging | `loguru` | Better than stdlib logging |

---

## 6. Project Structure

### `config.yaml` — Your Watchlist

```yaml
instruments:
  - name: "Mirae Asset Flexi Cap"
    type: mf
    scheme_code: "119598"
    reference_price: 152.43
    dip_threshold_pct: 3.0

  - name: "HDFC Mid Cap Opportunities"
    type: mf
    scheme_code: "118989"
    reference_price: 98.12
    dip_threshold_pct: 2.5

  - name: "Reliance Industries"
    type: stock
    ticker: "RELIANCE.NS"
    reference_price: 2950.00
    dip_threshold_pct: 3.0
```

### `.env` — Secrets

```
BOT_TOKEN=123456789:ABCdef...
CHAT_ID=987654321
```

### `.env.example` — Template (safe to commit)

```
BOT_TOKEN=your_telegram_bot_token_here
CHAT_ID=your_telegram_chat_id_here
```

---

## 7. Deep Dive: Each Component

### 7.1 Config Layer

**File:** `models.py`

```python
from dataclasses import dataclass
from typing import Literal

@dataclass
class Instrument:
    name: str
    type: Literal['mf', 'stock']
    reference_price: float
    dip_threshold_pct: float
    scheme_code: str | None = None   # for MF only
    ticker: str | None = None         # for stock only

@dataclass
class AlertResult:
    instrument: Instrument
    current_price: float
    drop_pct: float
    threshold_price: float
```

**File:** `config.py`

```python
import yaml
from models import Instrument

def load_config(path: str = "config.yaml") -> list[Instrument]:
    with open(path) as f:
        data = yaml.safe_load(f)
    instruments = []
    for item in data['instruments']:
        instruments.append(Instrument(**item))
    return instruments
```

**Why dataclasses?** They give you free `__repr__`, type hints, and no boilerplate — perfect for config objects that are just bags of data.

### 7.2 Data Fetchers

**File:** `fetchers/mf_fetcher.py`

```python
from mftool import Mftool
from models import Instrument

_mf = Mftool()

def fetch_nav(instrument: Instrument) -> float:
    data = _mf.get_scheme_quote(instrument.scheme_code)
    if not data or 'nav' not in data:
        raise ValueError(f"Could not fetch NAV for {instrument.name}")
    return float(data['nav'])
```

**File:** `fetchers/stock_fetcher.py`

```python
import yfinance as yf
from models import Instrument

def fetch_price(instrument: Instrument) -> float:
    ticker = yf.Ticker(instrument.ticker)
    price = ticker.fast_info.get('last_price')
    if price is None:
        raise ValueError(f"Could not fetch price for {instrument.name}")
    return float(price)
```

**Why separate files?** Each fetcher is independently testable and swappable. If yfinance breaks tomorrow, you only change `stock_fetcher.py`.

### 7.3 Threshold Engine

**File:** `engine.py`

```python
from models import Instrument, AlertResult

def check_threshold(instrument: Instrument, current_price: float) -> AlertResult | None:
    threshold_price = instrument.reference_price * (1 - instrument.dip_threshold_pct / 100)
    drop_pct = ((instrument.reference_price - current_price) / instrument.reference_price) * 100

    if current_price <= threshold_price:
        return AlertResult(
            instrument=instrument,
            current_price=current_price,
            drop_pct=round(drop_pct, 2),
            threshold_price=round(threshold_price, 2)
        )
    return None
```

**The core formula:**
```
threshold_price = reference_price × (1 - dip% / 100)

Example:
  reference_price = ₹152.43
  dip_threshold   = 3%
  threshold_price = 152.43 × 0.97 = ₹147.86

If current NAV ≤ ₹147.86 → fire alert
```

**Why return `None` instead of `False`?** Returning `None` means "no alert" and `AlertResult` means "alert with full context". The caller can do `if result:` naturally, and if an alert fires, it already has all the data needed to format the message.

### 7.4 Notifier

**File:** `notifier.py`

```python
import asyncio
from telegram import Bot
from models import AlertResult

class TelegramNotifier:
    def __init__(self, token: str, chat_id: str):
        self.bot = Bot(token=token)
        self.chat_id = chat_id

    def send_alert(self, result: AlertResult):
        message = self._format_message(result)
        asyncio.run(self.bot.send_message(chat_id=self.chat_id, text=message, parse_mode='Markdown'))

    def _format_message(self, result: AlertResult) -> str:
        r = result
        emoji = "📉" if r.instrument.type == "stock" else "🔔"
        type_label = "Stock" if r.instrument.type == "stock" else "Mutual Fund"
        return (
            f"{emoji} *Dip Alert — {r.instrument.name}*\n\n"
            f"Type: {type_label}\n"
            f"Reference Price: ₹{r.instrument.reference_price:.2f}\n"
            f"Alert Threshold: ₹{r.threshold_price:.2f} (−{r.instrument.dip_threshold_pct}%)\n"
            f"Current Price: ₹{r.current_price:.2f}\n"
            f"Drop from Reference: *−{r.drop_pct}%*\n\n"
            f"✅ Good time to consider investing!"
        )
```

**Why a class?** The `Bot` object is created once and reused for all alerts in a run cycle, instead of creating a new connection per message.

### 7.5 Scheduler

**File:** `scheduler.py`

```python
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.triggers.cron import CronTrigger
import pytz

IST = pytz.timezone('Asia/Kolkata')

def start_scheduler(stock_job, mf_job):
    scheduler = BlockingScheduler(timezone=IST)

    # Stocks: run at 3:35 PM IST on weekdays (5 min after market close)
    scheduler.add_job(
        stock_job,
        CronTrigger(day_of_week='mon-fri', hour=15, minute=35, timezone=IST)
    )

    # MFs: run at 11:00 PM IST on weekdays (after AMFI publishes NAVs)
    scheduler.add_job(
        mf_job,
        CronTrigger(day_of_week='mon-fri', hour=23, minute=0, timezone=IST)
    )

    print("Scheduler started. Waiting for scheduled runs...")
    scheduler.start()
```

**Why IST timezone explicitly?** If you ever run this on a server (e.g., a VPS with UTC timezone), hardcoding IST ensures the schedule is always correct regardless of the host machine's timezone.

### 7.6 CLI / Entry Point

**File:** `main.py`

```python
import os
from dotenv import load_dotenv
from loguru import logger
from config import load_config
from fetchers.mf_fetcher import fetch_nav
from fetchers.stock_fetcher import fetch_price
from engine import check_threshold
from notifier import TelegramNotifier
from scheduler import start_scheduler

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")
CHAT_ID = os.getenv("CHAT_ID")

instruments = load_config()
notifier = TelegramNotifier(token=BOT_TOKEN, chat_id=CHAT_ID)

def run_stock_checks():
    logger.info("Running stock price checks...")
    for inst in [i for i in instruments if i.type == 'stock']:
        try:
            price = fetch_price(inst)
            logger.info(f"{inst.name}: ₹{price}")
            result = check_threshold(inst, price)
            if result:
                logger.warning(f"ALERT: {inst.name} dipped {result.drop_pct}%")
                notifier.send_alert(result)
        except Exception as e:
            logger.error(f"Failed to check {inst.name}: {e}")

def run_mf_checks():
    logger.info("Running MF NAV checks...")
    for inst in [i for i in instruments if i.type == 'mf']:
        try:
            nav = fetch_nav(inst)
            logger.info(f"{inst.name}: ₹{nav}")
            result = check_threshold(inst, nav)
            if result:
                logger.warning(f"ALERT: {inst.name} dipped {result.drop_pct}%")
                notifier.send_alert(result)
        except Exception as e:
            logger.error(f"Failed to check {inst.name}: {e}")

if __name__ == "__main__":
    # For testing: run once immediately
    import sys
    if "--run-now" in sys.argv:
        run_stock_checks()
        run_mf_checks()
    else:
        start_scheduler(stock_job=run_stock_checks, mf_job=run_mf_checks)
```

The `--run-now` flag is essential during development — you don't want to wait until 3:35 PM to test if your code works.

---

## 8. Data Flow — Step by Step

Let's trace what happens when the stock scheduler fires at 3:35 PM:

```
1. APScheduler fires run_stock_checks()

2. Loop: for each stock instrument in config.yaml
   ├── Call fetch_price(instrument)
   │     └── yfinance fetches closing price from Yahoo Finance
   │         Returns: 2890.50 (for RELIANCE.NS)
   │
   ├── Call check_threshold(instrument, 2890.50)
   │     reference_price = 2950.00
   │     dip_threshold   = 3%
   │     threshold_price = 2950 × 0.97 = 2861.50
   │     2890.50 > 2861.50 → No alert → returns None
   │
   └── No alert → log the price, continue to next instrument

3. Next day, RELIANCE drops to 2840.00:
   ├── check_threshold returns AlertResult(
   │     current_price=2840.00,
   │     drop_pct=3.73,
   │     threshold_price=2861.50
   │   )
   │
   └── notifier.send_alert(result)
         → Telegram message sent to your chat_id
```

---

## 9. Edge Cases and How to Handle Them

| Edge Case | What Happens | How We Handle It |
|---|---|---|
| AMFI hasn't published NAV yet | `mftool` returns stale data | Run MF check at 11 PM to be safe |
| Stock market holiday | yfinance returns previous close | Acceptable — closing price is still valid |
| yfinance/mftool throws exception | Bot crashes for that instrument | `try/except` per instrument, log error, continue others |
| Telegram API is down | Message fails to send | `try/except` in notifier, log the failure |
| Bot restarts mid-day | Scheduler restarts, next run is scheduled fresh | Acceptable for SDE1 — no missed-run recovery |
| Same alert fires every day | Gets annoying fast | Track last-alerted date per instrument in a simple JSON file — skip if already alerted today |
| Network timeout | Request hangs forever | Set timeout on yfinance and requests calls |

**The "already alerted today" problem is worth solving at SDE1 level.** Without it, if Reliance stays below threshold for 5 days, you get 5 identical messages. Store a simple state file:

```python
# alert_state.json
{
  "RELIANCE.NS": "2024-01-15",
  "119598": "2024-01-14"
}
```

Before firing an alert, check if today's date is already in this file for that instrument. If yes, skip.

---

## 10. Environment Setup

### Step 1 — Create Telegram Bot

1. Open Telegram, search for `@BotFather`
2. Send `/newbot`
3. Give it a name (e.g., "My Dip Alert Bot") and a username (e.g., `my_dip_alert_bot`)
4. BotFather gives you a **Bot Token** — save it
5. Search for your new bot in Telegram and send `/start`
6. Find your **chat_id**: visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in browser — look for `"chat":{"id": <number>}`

### Step 2 — Python Environment

```bash
python3 -m venv venv
source venv/bin/activate       # Linux/Mac
# OR
venv\Scripts\activate          # Windows

pip install -r requirements.txt
```

### Step 3 — Create `.env`

```bash
cp .env.example .env
# Edit .env and fill in BOT_TOKEN and CHAT_ID
```

### Step 4 — Find MF Scheme Codes

```python
from mftool import Mftool
mf = Mftool()
codes = mf.get_scheme_codes()
# Search for your fund:
for code, name in codes.items():
    if 'mirae' in name.lower():
        print(code, name)
```

### Step 5 — Fill `config.yaml`

Add your instruments with their scheme codes / tickers, reference prices, and thresholds.

---

## 11. How to Run

```bash
# Test immediately (don't wait for schedule)
python main.py --run-now

# Run with scheduler (long-running process)
python main.py

# Keep it running on a Linux server
nohup python main.py &

# Or with systemd (recommended for always-on)
# Create /etc/systemd/system/dipalert.service
```

**To run this always-on for free:**
- Deploy on [Railway](https://railway.app) or [Render](https://render.com) free tier
- Or on an Oracle Cloud free-tier VPS (always free, 1GB RAM ARM instance)
- Or just leave your laptop running — it uses negligible CPU

---

## 12. Task Checklist

### Setup
- [ ] Create Telegram bot via `@BotFather`, save token
- [ ] Find your Telegram `chat_id`
- [ ] Create Python virtual environment
- [ ] Install dependencies (`requirements.txt`)
- [ ] Create `.env` with `BOT_TOKEN` and `CHAT_ID`

### Core Implementation
- [ ] `models.py` — `Instrument` and `AlertResult` dataclasses
- [ ] `config.py` — YAML loader that returns list of `Instrument`
- [ ] `config.yaml` — Add at least 2 MF instruments and 1 stock
- [ ] `fetchers/mf_fetcher.py` — Fetch NAV using `mftool`
- [ ] `fetchers/stock_fetcher.py` — Fetch price using `yfinance`
- [ ] `engine.py` — Threshold check logic
- [ ] `notifier.py` — Telegram message sender with formatted message
- [ ] `scheduler.py` — APScheduler with IST timezone, separate jobs for stocks and MFs
- [ ] `main.py` — Wire everything together, support `--run-now` flag
- [ ] `logger.py` — Set up loguru with file + console output

### Edge Case Handling
- [ ] `try/except` around each instrument check (one failure doesn't kill others)
- [ ] Alert deduplication — `alert_state.json` to avoid repeat alerts on same day
- [ ] Skip weekends in scheduler (market is closed)
- [ ] Handle `None` / missing data from fetchers gracefully

### Testing
- [ ] Test `--run-now` end to end — verify Telegram message arrives
- [ ] Manually set a reference price above current price to force an alert
- [ ] Test with an invalid scheme code — verify error is caught and logged
- [ ] Test with market closed (run on weekend) — verify no crash

### Polish
- [ ] `.env.example` file committed (not `.env`)
- [ ] `.gitignore` with `venv/`, `.env`, `__pycache__/`, `*.pyc`, `alert_state.json`
- [ ] `requirements.txt` with pinned versions
- [ ] `README.md` with quick start (5 steps to get running)
- [ ] Meaningful log messages at INFO / WARNING / ERROR levels

---

## 13. Future Extensions (SDE2/SDE3)

This section is intentionally brief — these are the next chapters, not this one.

**SDE2 additions:**
- REST API (FastAPI) to manage watchlist without editing YAML
- PostgreSQL for storing instruments and alert history
- Multi-user support with JWT auth
- Docker + docker-compose setup
- Deploy on a VPS with systemd

**SDE3 additions:**
- Multi-source data ingestion with fallback and circuit breakers
- Persistent alert queue with at-least-once delivery guarantees
- Statistical dip detection (moving average + standard deviation)
- Per-instrument-type scheduling with market calendar awareness
- Observability: Prometheus metrics + Grafana dashboard
- Rewrite core pipeline in Go

---

*Document version: 1.0 — SDE1 scope*
