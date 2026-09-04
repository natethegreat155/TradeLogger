# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

TradeLogger is a **single-file, zero-dependency, offline-first trade journal**: the entire app is `TradeLog.html` (~2700 lines). There is no build step, no package manager, no test suite, no server, and no lint config. The only external resource is Chart.js loaded from a CDN (`TradeLog.html:7`).

To run it: open `TradeLog.html` in a browser (`open TradeLog.html`). To test a change, reload the page. Chrome/Edge are the target browsers — the File System Access API path (Quick Save, saved file handle in IndexedDB) only exists there; Firefox/Safari fall back to Downloads.

**Keep it single-file.** Do not split into modules, add a bundler, or add npm dependencies — distributability as one downloadable HTML file is the product. New third-party libraries must be CDN `<script>` tags or inlined.

### File layout inside `TradeLog.html`

Three regions: `<style>` (lines 8–235, CSS custom properties dark theme), markup (236–465, five tab panels + the API-key modal), `<script>` (466–2716). The script is divided by full-width banner comments — grep `'^//  [A-Z]'` to list them. Match that convention when adding a region.

## Data pipeline

```
CSV text → detectBroker() → parseTOSStatement() | parseTOS() | parseIBKR() | parseFidelity()
        → fills[]  (global `allTrades`, sorted by timestamp)
        → groupTrades()
        → tradeGroups[]  → renderLog / renderDashboard / buildCalendar / coach context
```

**`allTrades` (fills) is the source of truth; `tradeGroups` is always derivable** via `groupTrades(allTrades)`. The JSON export (`buildPayload`, version 2) persists only `trades` + `notes` — groups are rebuilt on load. `localStorage` additionally caches groups under `tj_groups` as an optimization. Any change to grouping or P&L math must therefore be safe to re-run over historical fills.

### The fill contract

All four parsers must emit the identical object shape, or downstream code breaks silently:

```js
{ broker, timestamp, date, symbol, underlying, expDate, strike, optType,
  isOption, side: 'BUY'|'SELL', qty, price, commission, netAmount, spread }
```

`qty` is always positive with direction carried by `side`; `commission` is always positive (fees added, not netted). `date` is the **Eastern Time** calendar date (`fmtDateET`), not UTC — everything user-facing is ET.

### Grouping semantics (`groupTrades`, line 1384)

Fills are bucketed by instrument key `underlying|expDate|strike|optType`, sorted by timestamp, then walked with a running position counter; **a trade is flushed every time position returns to zero**, so re-entries on the same instrument the same day become separate trades. A leftover non-zero position becomes an OPEN trade with `netPnl: null` (open trades are excluded from every stat).

The group `id` is `underlying|openDate|expDate|strike|optType|firstFillTimestamp` and **notes are keyed by that id** (`notes[g.id]`). Changing the id format or the grouping boundaries orphans every existing user note in saved journals — don't, without a migration.

### Time handling

Broker CSVs give local-ET wall-clock times with no offset. `getETOffset()` resolves EST/EDT for a given date via `Intl.DateTimeFormat`, and `parseDateTime()` appends it before constructing the `Date`. Never parse a broker timestamp with a bare `new Date(str)` — it silently becomes UTC or browser-local and shifts every trade's date, hour-of-day chart, and calendar cell.

### Multipliers

Options use `MULTIPLIER = 100`. Non-options go through `getFuturesMultiplier()` (line 488), which strips a leading `/` and a trailing month-year code (`/MESM26` → `MES`) and looks up a dollars-per-point map, defaulting to `1` for equities. Add contracts to that map rather than special-casing at call sites.

### Deduplication

`tradeKey(f)` = `timestamp|symbol|side|qty|price`. It guards both CSV import (re-importing overlapping date ranges) and journal merge. `parseTOSStatement` also dedups internally because a TOS Account Statement can list the same fill in both the Cash Balance and Account Trade History sections.

## Parsers

`detectBroker()` sniffs the first 3000 chars and returns `'fidelity' | 'fidelity_positions' | 'tos_stmt' | 'tos' | 'ibkr' | 'unknown'`. `fidelity_positions` is a *rejection* case, not a parser: Fidelity's `Portfolio_Positions_<date>.csv` is a holdings snapshot with no transactions in it, and users reach for it first, so it is detected only to explain what to download instead. All parsers are section-aware — broker CSVs concatenate several tables with different headers into one file, so they scan for a section header row and only read rows beneath it (TOS: `Filled Orders` only, skipping `Working`/`Canceled`/`Rolling`; IBKR: `Trades,Header` rows). Column access is by lowercased header name via a local `get(k)` helper, never by fixed index.

TOS Account Statement fills are extracted by regex from the free-text Description column (`BOT +5 SPX 100 (Weeklys) 1 APR 26 6625 CALL @1.80`), which is why expiration parsing has several fallbacks (`parseOptionSymbol` handles OCC symbols, separate exp/strike columns, and description text; `parseTOSExp` handles `1 APR 26`).

**Fidelity** (`parseFidelity`) handles two shapes. The web Accounts History download carries trades in the free-text `Action` column (`YOU BOUGHT OPENING TRANSACTION CALL (SPY) … NOV 15 24 $585`) and has **no execution time at all** — only a Run Date. That drives three pieces of machinery worth knowing before editing it:

- **Non-trade filtering.** Side is decided by *anchoring on the Action prefix* (`FIDELITY_BUY` / `FIDELITY_SELL` / `FIDELITY_EXPIRED`), never by scanning the string for words. The Action text embeds the full company name, so a substring scan for `DISTRIBUTION`/`INTEREST`/`SPLIT` silently eats real trades in stocks like DISTRIBUTION SOLUTIONS GROUP. Everything not matching a trade prefix (`JOURNALED`, `REINVESTMENT`, `DIVIDEND RECEIVED`) is a cash movement. `FIDELITY_MMF` additionally drops core-position sweeps, which *do* arrive as `YOU BOUGHT`. `EXPIRED` rows are kept as $0 closing fills (their Price column is empty) so worthless options don't sit OPEN forever, and are re-dated from the `as of YYYY-MM-DD` in the Action text — Fidelity books them up to 3 days after the fact.
- **Sort direction.** The export is newest-first, so fills are flipped to chronological before grouping, or buys land after the sells that closed them. Direction is resolved from the **running `Cash Balance` column**: `balance[older] + amount[newer] == balance[newer]` holds only in the true direction, which makes it exact and effective even within a single day (fall back to the Run Date gradient, then to newest-first). **Do not** infer direction from `OPENING`/`CLOSING TRANSACTION` labels — verified against a real export, a close legitimately precedes an open when a position is exited and re-entered the same day, which is common.
- **Commissions.** Taken from `Amount` (the cash Fidelity actually moved) via `implied = notional − |amount|`, not from the Commission/Fees columns. Where those columns are populated this reproduces them exactly; where they aren't, it still captures unlisted regulatory fees and display-rounded fill prices. Validated against a real export: every closed trade matched the broker's booked P&L to the cent.
- **Synthetic timestamps.** Same-day fills are sequenced one second apart from 12:00 ET. This keeps grouping and dedup deterministic across re-imports, but means `durationMin` and the P&L-by-Hour chart are meaningless for these trades — the import log warns the user. Exports with a real Time column (Active Trader Pro) bypass all of this.

Option identity comes from `parseFidelityOptionSymbol` first — the Symbol column is structured and exact — then `parseFidelityOptionDesc` as a fallback. Don't route Fidelity symbols through the shared `parseOptionSymbol`: it assumes the padded OCC form and would turn a `585` strike into `0.585`. Verified against a real export, Fidelity symbols take the form `-XYZ270115C160`, `-ABCD270115C7.5` (decimal strike) and `-QQ260918C40` (2-char underlying), always with a leading dash and often a leading space.

Fidelity writes option **descriptions** in two different shapes, and `parseFidelityOptionDesc` handles both: `DKS JAN 15 2027 $160 CALL` (ticker first, 4-digit year, type last — this is the confirmed real one) and `CALL (SPY) SPDR S&P 500 ETF TR NOV 15 24 $585 (100 SHS)` (type first, ticker in parens, 2-digit year). Both regexes are anchored on real month names so a company name can't be misread as an expiration.

Fidelity CSVs are **UTF-8 with a BOM**. `normHeader` strips it, without which the first header cell reads `\ufeffrun date` and the header never matches — the parser would silently find no trades.

Note that `parseDateTime` tries a bare `new Date()` before its `M/D/YYYY` branch, so a 4-digit-year date with a time is read as browser-local rather than ET. `parseFidelity` sidesteps this by building timestamps from its own normalized ISO date plus `getETOffset()`; do the same in new code rather than reordering that shared function, which would shift existing TOS/IBKR imports and invalidate the dedup keys in already-saved journals.

When touching a parser, keep a real export handy — format regressions are the recurring bug class in this repo's history and are invisible until the numbers are wrong.

## AI Coach tab

Calls an OpenAI-style `/chat/completions` endpoint directly from the browser (`callLLM`). The provider is **not** hardcoded: `LLM_PROVIDERS` holds presets (OpenAI, Z.ai/GLM, Zhipu BigModel, and a free-form Custom), and `getLLMConfig()` resolves the endpoint, model and max-tokens parameter actually in force. Anything speaking the OpenAI dialect works.

Two things to preserve when touching it:

- **`tokenParam` is per-provider.** OpenAI's newer models require `max_completion_tokens`; most compatible APIs still expect `max_tokens`. Sending the wrong one is rejected, so the payload key is built from the preset rather than written literally.
- **Reasoning models need headroom and an empty-`content` path.** The per-provider `maxTokens` budget covers thinking *and* the answer, which is why GLM's is 4x OpenAI's; at the original flat 1024 a GLM model spent the whole budget mid-thought and returned `finish_reason: "length"` with empty `content`. Never fall back to rendering `reasoning_content` — it surfaces raw chain-of-thought that reads like a reply truncated mid-sentence. Thinking is left enabled deliberately; the budget is the knob, overridable per user in the settings modal (`tj_llm_maxtok`).
- **`fetch` only rejects on network/CORS failures** — an HTTP 401/404 still resolves. Both paths are handled separately because a browser CORS block and a bad key look nothing alike to the user. All three bundled providers do send `Access-Control-Allow-Origin` (verified, including for `Origin: null`, which is what a `file://` page sends), so a local page can call them directly.

Settings live in `localStorage` under `tj_llm_provider` / `tj_llm_model` / `tj_llm_base`; the key stays at `tj_openai_key` (name kept for backwards compatibility) plus an in-memory `_openAIKey` fallback. The key is only ever sent to the configured endpoint, which the settings modal displays verbatim before saving. `buildCoachContext()` serializes filtered trade stats plus up to 200 trade rows (with notes) into the first user message — the model only ever sees that summary, never raw fills.

`COACH_SYSTEM_PROMPT` constrains the coach to process/psychology and forbids financial advice. Preserve those guardrails when editing it.

## Persistence keys

`localStorage`: `tj_trades`, `tj_groups`, `tj_notes`, `tj_coach_messages`, `tj_coach_timeframe`, `tj_coach_range`, `tj_openai_key`. IndexedDB `tradelog_v1` stores the `fileHandle` for Quick Save across sessions. All reads/writes are wrapped in `try/catch` because storage can be blocked — keep that.

## Rendering conventions

Renderers build HTML strings and assign `innerHTML`; handlers are inline `onclick`/`oninput` attributes calling globals. There is no framework or reactivity — after mutating state, call `save()` and the relevant `renderX()`. Chart.js instances are kept in the `charts` map and must be `.destroy()`ed before re-creating (see `buildCharts`) or they leak and stack tooltips. `showTab()` re-renders on tab switch, and the tab id list there is positional against `.nav-tab` order — update both together when adding a tab.
