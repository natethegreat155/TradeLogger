# TradeLogger

A free, single-file HTML trade journal for day traders. No server, no account, no subscription — just open `TradeLog.html` in your browser.

## Features

- Import trades from **Thinkorswim** and **Interactive Brokers** CSVs
- Auto-groups fills into trade sessions with P&L, avg entry/exit, commissions, and duration
- Dashboard with equity curve, win rate, profit factor, P&L by ticker/hour/day, and more
- Calendar heatmap showing daily P&L at a glance
- Per-trade notes for journaling lessons learned
- Scaling visualization showing exactly how you built and exited positions
- All data saved locally — export/import as `.json` to back up or move between devices
- All times displayed in **Eastern Time** to match broker timestamps

---

## Getting Started

1. Download `TradeLog.html`
2. Open it in Chrome or Edge (recommended for full save functionality)
3. Go to the **Import** tab and drag in your broker CSV
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

**Save Journal** — exports all trades and notes to a `.json` file.

- In **Chrome/Edge**: uses the File System Access API so **Quick Save** overwrites the same file after the first save
- In **Safari/Firefox**: saves to your Downloads folder each time

**Load Journal** — imports a previously saved `.json`, merging trades and notes with anything already loaded. Duplicates are skipped.

> Your data is also kept in `localStorage` between sessions, but the `.json` file is your reliable backup. Save it regularly.

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
