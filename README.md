# TradeLogger

A free, single-file HTML trade journal for day traders. No server, no account, no subscription — just open `TradeLog.html` in your browser.

## Features

- Import trades from **Thinkorswim**, **Interactive Brokers** and **Fidelity** CSVs
- Auto-groups fills into trade sessions with P&L, avg entry/exit, commissions, and duration
- Dashboard with equity curve, win rate, profit factor, P&L by ticker/hour/day, and more
- Calendar heatmap showing daily P&L at a glance
- Per-trade notes for journaling lessons learned
- Scaling visualization showing exactly how you built and exited positions
- All data saved locally — export/import as `.json` to back up or move between devices
- All times displayed in **Eastern Time** to match broker timestamps
- Optional **AI Coach** tab — bring your own API key from OpenAI, Z.ai (GLM), or any OpenAI-compatible provider

---

## Getting Started

1. Download `TradeLog.html` - just save it wherever you'll remember it lives.
2. Open it in Chrome or Edge (recommended for full save functionality)
3. Go to the **Import** tab (you should see this first as a default) and drag in your broker CSV. Currently configured for Thinkorswim, IBKR and Fidelity.
4. Your trades appear in the **Trade Log** tab immediately
5. Hit **Save Journal** to export a `.json` backup

No install required. Everything runs in the browser.

---

## Importing Trades

### Thinkorswim (TOS)

**Option 1 — Account Statement (recommended)**

1. In TOS, go to **Monitor → Account Statement**
2. Set your date range, then click **Export to File**
3. Drop the `.csv` into TradeLogger

The parser reads the Description column and extracts the symbol, expiration, strike, side, and quantity automatically (e.g. `BOT +5 SPX 100 (Weeklys) 1 APR 26 6625 CALL @1.80`).

**Option 2 — Trade History**

1. In TOS, open the **Trade History** tab
2. Export to CSV
3. Drop into TradeLogger

### Interactive Brokers (IBKR)

1. Go to **Reports → Flex Queries**
2. Create a Flex Query with the **Trades** section — include: DateTime, Symbol, Buy/Sell, Quantity, T. Price, Comm/Fee, Proceeds, Asset Category
3. Run the query and download the CSV
4. Drop into TradeLogger

Activity Statement CSVs also work.

### Fidelity

Use the **History** download from the **Activity & Orders** tab — the file named `History_for_Account_X…….csv`. That's the one with your actual trades in it.

1. On Fidelity.com, open the **Activity & Orders** tab
2. Select a **single account**, then the **History** sub-tab, and set your date range
3. Click **Download** — this saves `History_for_Account_X…….csv`
4. Drop the file into TradeLogger as-is

> ⛔ **Not `Portfolio_Positions_<date>.csv`.** That export is a snapshot of what you currently hold — it has no buys, sells or trade dates in it, so there is nothing to journal. TradeLogger recognises it and tells you to grab the History download instead.

Dividends, reinvestments, journal entries, and core/money-market cash sweeps are filtered out automatically, so only real trades land in the log. Options that expired worthless are imported as closing fills at $0 and dated to the real expiration date rather than the settlement date. Stock, option and fractional-share fills are all supported.

Commissions come from Fidelity's `Amount` column — the actual cash that moved — so a trade's P&L matches your account to the cent even when Fidelity folds an unlisted regulatory fee into the total.

> **Note on times:** Fidelity's history export records only a *Run Date* — there are no execution timestamps. Trades import with correct P&L, but same-day fills are sequenced one second apart from 12:00 ET in the order they appear in the file, so **hold durations and the "P&L by Hour of Day" chart aren't meaningful for Fidelity-sourced trades**. Active Trader Pro exports that include a Time column are used as-is with real times.

**Multiple files at once:** You can drag several CSVs in one drop. Duplicates are automatically filtered by matching timestamp + symbol + side + qty + price.

---

## Trade Log

Trades are grouped by symbol + date + expiration + strike. Each row shows:

| Column | Description |
|--------|-------------|
| Date / Time | Entry date and entry–exit time range (ET) |
| Instrument | Ticker, expiration, and strike for options |
| Type | CALL, PUT, or STOCK badge |
| Bought / Avg Entry | Total contracts/shares bought and average fill price |
| Avg Exit / Sold | Average sell price and total quantity sold |
| Comm | Total commissions and fees |
| Net P&L | Profit or loss after commissions (green/red), or OPEN if position not yet closed |
| R:R % | Return as a percentage of cost basis |
| DTE | Days to expiration at time of trade |
| Duration | Time between first and last fill |

Click the **▶** arrow on any row to expand the fill detail, position timeline, and notes.

### Filtering

Use the filter bar above the log to narrow by:
- **Date range**
- **Ticker** (partial match)
- **Outcome** — Wins, Losses, or Open
- **Type** — Calls, Puts, or Stocks

The header updates to show P&L and win rate for your current filter.

---

## Dashboard

The dashboard shows statistics and charts for all trades (or your current filter).

**Stats:** Net P&L, Win Rate, Profit Factor, Avg Win, Avg Loss, R:R Ratio, Largest Win/Loss, Max Drawdown, Avg Duration, Current Streak, Open Trades

**Charts:**
- Equity Curve — cumulative P&L over time
- P&L Distribution — histogram by $100 buckets
- P&L by Ticker — top 10 symbols
- P&L by Day of Week
- P&L by Hour of Day — entry time analysis
- Calls vs Puts vs Stock — doughnut breakdown
- Win Rate by DTE — 0DTE, 1–7, 8–14, 15–30, 30+
- Monthly P&L — bar chart by month

---

## Calendar

The **Calendar** tab shows a monthly heatmap of daily P&L. Click any trading day to see a breakdown of trades for that day.

---

## Scaling Visualization

Expanding a trade shows a position timeline — a fill-by-fill view of how you scaled in and out:

- Each fill shows time, side, quantity, price, current position size, and leg P&L
- A color-coded bar visualizes position size (green = long, red = short)
- Summary shows max position size, total fills, and total capital at risk
- Entry/exit scaling patterns are described in plain text (e.g. "Scaled in: 5 + 3 + 2 contracts")

---

## Notes / Journaling

Each trade has a notes field (visible when expanded). Use it for post-trade analysis, lessons learned, or anything else. Notes are saved automatically and included in your `.json` export.

---

## Saving & Loading

**Save Journal** — exports all trades and notes to a `.json` file. Save this file ideally in the same place where you downloaded the HTML file.

- In **Chrome/Edge**: uses the File System Access API so **Quick Save** overwrites the same file after the first save
- In **Safari/Firefox**: saves to your Downloads folder each time

**Load Journal** — imports a previously saved `.json`, merging trades and notes with anything already loaded. Duplicates are skipped.

> Your data is also kept in `localStorage` between sessions, but the `.json` file is your reliable backup. Save it regularly.

---

## AI Coach

The **Coach** tab sends a summary of your trades — stats plus up to 200 trade rows and your notes, never raw fills — to a language model and talks through the patterns in them.

Click the key indicator (top right of the tab) to configure it:

| Field | What it's for |
|-------|----------------|
| Provider | OpenAI, Z.ai (GLM), Zhipu BigModel, or Custom |
| Model | Model name, prefilled with the provider's default — overwrite it with whatever you want to run |
| Endpoint URL | Custom provider only: the full `/chat/completions` URL |
| Max output tokens | Optional. Reasoning models spend part of this thinking before they reply, so leave room — blank uses the provider default |
| API key | Your key, stored only in this browser's localStorage |

Any provider with an **OpenAI-compatible** `/chat/completions` API works. The modal shows the exact endpoint your key will be sent to before you save, and it is never sent anywhere else.

> The coach is prompted to discuss process and psychology only, and is explicitly told not to give financial advice or tell you what to buy or sell.

---

## Browser Compatibility

| Browser | Import | Save / Quick Save | Notes |
|---------|--------|-------------------|-------|
| Chrome / Edge | ✅ | ✅ Full File System API | Recommended |
| Firefox | ✅ | ⚠️ Downloads folder only | No Quick Save |
| Safari | ✅ | ⚠️ Downloads folder only | No Quick Save |

---

## License

Free to use and modify. No warranty.
