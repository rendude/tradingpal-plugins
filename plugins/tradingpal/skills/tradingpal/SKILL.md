---
name: tradingpal
description: >-
  Read TradingPal's breakout patterns (wedges, pennants and triangles that form
  right before a large run) on stocks & crypto: what the nightly scan found
  across the whole market, ranked, and the setups on any ticker, each with
  lines, trigger, stop and target, the 20-year track record
  per pattern family and per ticker, the last occurrences with outcomes, and a
  chart image. Use when the user asks what the scan found, what is about to
  break out, what is setting up on a stock or crypto ticker, how a pattern has
  performed, or for a chart of a setup. Needs a TradingPal signed-in MCP
  connection, or an API key (tp_live_…) in TRADINGPAL_API_KEY, on an account
  with Premium+.
---

# TradingPal pattern API

> TradingPal helps you find better breakout patterns on stocks & crypto, faster. It looks at the entire market for wedges, pennants and triangles that form right before a large run. Based on research and backtesting, we sift out poorly formed patterns and ones with poor historical results. Each pattern comes with its historical win rate and a suggested entry, stop and target. This API and MCP server give an agent what the nightly scan found across the whole market, ranked, and the current setups on any ticker, with the lines and levels to draw, the track record behind each one, the last occurrences with outcomes, and a chart image. Nothing is computed per request; every answer is the same stored nightly result the site serves.

Base URL: `https://api.tradingpal.io/api/v1`. MCP endpoint: `https://api.tradingpal.io/mcp` (Streamable HTTP). Human docs and key management: https://tradingpal.io/developers. Machine index: `GET https://api.tradingpal.io/api/v1/` (JSON). OpenAPI 3.1: `https://api.tradingpal.io/api/v1/openapi.json`. This file: `https://api.tradingpal.io/api/v1/llms.txt`. Installable skill: `https://api.tradingpal.io/api/v1/skill.md`.

## Rules for agents

1. **Never present a win rate alone.** Every track record carries a `summary` sentence with the sample size, the win rate, the average win and the average loss. Quote it whole. "54% win rate" without "average loss -4.2%, 1,287 trades" is the misuse this data exists to prevent.
2. **Say which session the data describes.** Every response carries `as_of_session` (the last trading day the nightly covered). Setups change nightly; do not cache them across days.
3. **Lines are two dated endpoints on a log-scale chart.** Draw a straight line between `start` and `end` with the price axis in log scale. There is no slope field on purpose.
4. **This is educational chart analysis, not advice.** Repeat the `disclaimer` when presenting a setup as something the user might act on.
5. **Any ticker can be looked up; tracked tickers are the user's own list.** `GET /scan` (`get_scan`) covers the whole market, and the pattern, history and chart calls work on any ticker the nightly covers, tracked or not. The tracked tickers are the user's My watchlist on the site; change them only when the user asks.

## Plan

API and MCP access requires **Premium+** subscription at $30/month (or $300/year): all of the benefits in Premium plus API and MCP access. Premium alone does not unlock them. A first subscription starts with 14 days free; the card is billed when it ends unless the user cancels. An account without the plan can connect and create keys, but every keyed route and tool answers `forbidden` with a `fix` pointing at https://tradingpal.io/developers/quickstart, where the user gets the plan (or switches an existing Premium subscription to it) in one step.

## Connect

Two ways in, same account, same limits:

- **Sign in (MCP hosts).** Add the server with no key; the server answers 401 with a `WWW-Authenticate` pointer and the host opens the browser for the user to sign in to TradingPal and approve (OAuth 2.1: PKCE, dynamic client registration at `https://api.tradingpal.io/oauth/register`, metadata at `https://api.tradingpal.io/.well-known/oauth-authorization-server`). The approval becomes a key named after the host on the user's keys page, revocable there.
- **A key (REST and scripts).** Every keyed call sends `Authorization: Bearer tp_live_…`. The user creates a key (shown once) at https://tradingpal.io/developers. Optionally send `X-TradingPal-Client: <your agent or script name>` so the account can see which client made which calls.

MCP (preferred for agents):

- Claude Code: `claude mcp add --transport http tradingpal https://api.tradingpal.io/mcp`, then sign in when it asks (or add `--header "Authorization: Bearer tp_live_…"` to skip the sign-in).
- Codex: `codex mcp add tradingpal --url https://api.tradingpal.io/mcp` then `codex mcp login tradingpal`, or the key in `bearer_token_env_var = "TRADINGPAL_API_KEY"` in `config.toml`.
- Cursor and other hosts: an HTTP MCP server at `https://api.tradingpal.io/mcp`; sign in when prompted, or set an `Authorization: Bearer tp_live_…` header.

Without an account, `https://api.tradingpal.io/mcp/demo` connects with nothing, lists the tools and answers `demo` and `get_family_track_record`; the other tools return an error that says how to connect for real.

Limits: 60 requests a minute and 10,000 a day per key; keyless routes 30 a minute per IP. A `429` carries `details.retry_after_seconds`.

## Try it without a key

`GET https://api.tradingpal.io/api/v1/demo/NVDA` returns the current setups on a demo symbol with their track record, exactly as the keyed route would. `GET https://api.tradingpal.io/api/v1/demo/NVDA/chart.png?setup_id=…` returns its chart. Demo symbols: AAPL, MSFT, NVDA, AMZN, GOOGL, META, TSLA, AMD, NFLX, JPM. `GET https://api.tradingpal.io/api/v1/families` (keyless) returns the track record of every pattern family.

## Routes

Trailing slashes are optional. All responses are JSON except `chart.png`.

| Route | Key | Returns |
|---|---|---|
| `GET /me` | yes | The key, plan, limits and tracked-ticker count |
| `GET /scan?status=forming&family=&direction=&max_distance_pct=&filters=[…]&limit=50&offset=0` | yes | What the nightly scan found across the whole market, ranked, narrowed by the screener's own filters (see **Scan**) |
| `GET /tickers` | yes | Tracked tickers, each with `covered` (true when the nightly covers it) |
| `PATCH /tickers` body `{"add": ["NVDA"], "remove": ["AAPL"]}` | yes | Adds and removes tickers, leaving the rest alone; either list or both |
| `PUT /tickers` body `{"symbols": ["NVDA", "AAPL"]}` | yes | Replaces the whole tracked set, all-or-nothing; `cap` says how many fit |
| `GET /tickers/{symbol}/patterns` | yes | Current setups on one ticker, tracked or not (see **Setup**) |
| `GET /tickers/{symbol}/history?limit=5&outcome=all` | yes | The last occurrences on that ticker with what happened next, newest first (the receipts behind its track record, at most 5); `outcome` is `all`, `resolved` or `open` |
| `GET /tickers/{symbol}/chart.png?setup_id=…` | yes | PNG of one current setup: candles, both lines, trigger, stop, target |
| `GET /families` | no | The five families with their track record |
| `GET /families/{family}/stats` | no | One family's track record |
| `GET /universe` | no | Symbols the nightly covers |
| `GET /demo/{symbol}` and `GET /demo/{symbol}/chart.png` | no | The demo (above) |
| `GET /` , `/llms.txt`, `/skill.md`, `/openapi.json` | no | This documentation |

Families: `rising_wedge`, `falling_wedge`, `bullish_pennant`, `bearish_pennant`, `triangle`. Every tier sees every family.

## MCP tools

| Tool | Key | Does |
|---|---|---|
| `get_scan(family?, direction?, status?, max_distance_pct?, filters?, limit?, offset?)` | yes | `GET /scan`. The one to call for "what did the scan find", "best setups tonight", "what is about to break out", or any screen the user describes (`filters`). |
| `get_ticker_patterns(symbol)` | yes | `GET /tickers/{symbol}/patterns`. The one to call for "what is setting up on X". |
| `get_pattern_history(symbol, limit?, outcome?)` | yes | `GET /tickers/{symbol}/history`: the last five occurrences |
| `get_pattern_chart(symbol, setup_id)` | yes | The PNG as an image content block, plus `page_url` (the setup on the TradingPal site, the link to give the user); the result's `_meta` carries `io.tradingpal/chart_url`, a keyless link to the same PNG that is good for a week, for code and app cards; in ChatGPT and Claude the setup shows as a card with the image |
| `list_tracked_tickers()` | yes | `GET /tickers`: the account's My watchlist on the site, with the account's email |
| `add_tickers(symbols)` | yes | `PATCH /tickers` with `add`: starts tracking, leaves the rest alone. Call it when the user says "track NVDA" |
| `remove_tickers(symbols)` | yes | `PATCH /tickers` with `remove`: stops tracking, leaves the rest alone |
| `set_tracked_tickers(symbols)` | yes | `PUT /tickers`: replaces the list from scratch, so include the tickers to keep; prefer add_tickers and remove_tickers for changes |
| `get_family_track_record(family?)` | no | `GET /families` or `GET /families/{family}/stats` |
| `demo(symbol)` | no | The keyless demo, with the first setup's chart as an image and its link in `_meta` |

Tool results carry the JSON payload as text and as `structuredContent`; errors come back as tool errors whose text starts with the error code and ends with the fix.

## Envelope

Every JSON response has `api_version` (`v1`), `as_of_session` (ISO date or null), `data_version` (the artifact the answer came from), the route's own fields, `source` (`{"name": "TradingPal", "url": "https://tradingpal.io"}`) and `disclaimer`.

## Setup

One current pattern on a ticker, from `/patterns` (field values illustrative):

```json
{
  "setup_id": "NVDA:D:falling_wedge:1",
  "lineage_id": "…",
  "symbol": "NVDA", "interval": "D",
  "family": "falling_wedge", "family_label": "Falling wedge",
  "pattern_type": "falling_wedge", "direction": "bullish",
  "status": "forming",
  "pattern_start": "2026-08-08", "pattern_end": "2026-09-07",
  "as_of_session": "2026-09-25",
  "last_close": 101.5,
  "price_scale": "log",
  "lines": {
    "upper": {"start": {"date": "2026-08-08", "price": 110.0},
              "end": {"date": "2026-09-07", "price": 104.0},
              "touch_count": 3,
              "touches": [{"date": "2026-08-08", "price": 111.2}, {"date": "2026-08-23", "price": 106.9}]},
    "lower": {"start": {"date": "2026-08-08", "price": 90.0},
              "end": {"date": "2026-09-07", "price": 98.0},
              "touch_count": 2, "touches": []}
  },
  "trigger_rule": "A daily close above the upper line; the trigger price moves with the line.",
  "plans": [{"direction": "bullish", "trigger_price": 105.25, "stop_price": 96.0, "target_price": 120.0}],
  "track_record": {"family": {"…": "stat block"}, "ticker": {"…": "stat block or null"}}
}
```

- `status` is `forming` until the trigger rule fires; an `entered` object (entry date and price, stop, target) appears once it has.
- `plans` holds the trade plan per direction: the trigger price as of the last session, the stop and the measured-move target. A symmetrical triangle can carry two plans.
- `touches` are the bars that touched the line (date and the bar's high or low), so a charting tool can reproduce the fit.
- `track_record.ticker` is null when the ticker has no resolved history for that family.

### Stock facts

`/patterns` (`get_ticker_patterns`), the chart tool and the demo also carry `stock_facts`, once per ticker beside `setups`: the ticker's facts as the TradingPal screener's quick view shows them, in its groups (Price, Performance, Momentum, Volume and volatility, Company or Asset), each row a label and a ready value. Keep the labels and values as given. Null when the ticker has no current facts; scan rows leave them out to stay small.

```json
{"as_of": "Facts as of the Sep 25 close.",
 "sections": [{"title": "Momentum", "rows": [{"label": "EMA8 vs EMA21", "value": "+2.5%"}, {"label": "RSI (14)", "value": "59"}]}]}
```

## Scan

`GET /scan` returns the ranked book of the last nightly run: every setup forming across the whole market (`status=forming`, the default), the ones that have triggered and are running (`status=in_progress`), or both (`status=all`). The order is the TradingPal screener's order, and `rank` is the position in it: rank 1 is the best-ranked setup of the night. Ranks are assigned over the whole book before any filter, so narrowing by `family`, `direction` or `max_distance_pct` keeps each row's rank. Outcomes are never served here.

Each row is a **Setup** (above) plus:

```json
{
  "rank": 1,
  "expected_gain_pct": 6.8,
  "win_rate_pct": 54.0,
  "win_rate_trades": 31,
  "distance_to_trigger_pct": 3.69,
  "setup_id": "NVDA:D:falling_wedge:1", "symbol": "NVDA", "…": "the Setup fields"
}
```

- `expected_gain_pct` is **Typical return**, as the TradingPal screener labels it, and the value the ranking sorts by: the typical winning move for this pattern on this ticker, in percent, blended with the family's record (ticker evidence weighted by its trade count). Null when the setup is not ranked; unranked rows sort last.
- `win_rate_pct` is the win rate the screener shows on the setup, and `win_rate_trades` how many past trades it counts. Null without a track record.
- `distance_to_trigger_pct` is how far the last close is from the trigger, in percent of the close, signed (positive: price must rise to reach it). Small means about to break out; `max_distance_pct=2` keeps the forming setups within 2%.
- `count` says how many matched; page with `limit` (up to 200) and `offset`. The answer comes from a short cache, so page within a session.
- `filters` runs the user's own screen: the same filter list the TradingPal screener takes, so "stocks over $10 with RSI under 40" is `filters=[{"kind":"price","op":">","value":10},{"kind":"rsi","op":"<","value":40},{"kind":"asset_type","value":"stock"}]` (URL-encode it on the REST route; the MCP tool takes the array). Every predicate must match, one per kind, and a setup with no evidence for a kind never matches. The response echoes them as `filters.predicates`. A malformed list answers 400 `bad_request` naming the predicate.

Present the top few, not the list: for each, the rank, the symbol, the family and direction, and the three numbers the screener shows, named as it names them: Typical return (`expected_gain_pct`), Win rate (`win_rate_pct`, with `win_rate_trades` trades) and Distance to trigger (`distance_to_trigger_pct`); then the trigger, stop and target and the family `summary` sentence.

### Scan filters

Numeric kinds take `op` (`<`, `<=`, `>`, `>=`, `=`) and `value`; choice and text kinds take `value`; boolean kinds take `value` true or false; the moving-average, performance and high/low kinds also take `period`. Dollar and share counts are plain numbers (market_cap 2000000000 is $2B).

| kind | takes | unit | meaning |
| --- | --- | --- | --- |
| `direction` | value one of bearish, bullish |  | The traded direction. Also a plain query parameter. |
| `phase` | value one of armed, confirmed, forming |  | forming, armed (price near the trigger) or confirmed, as the site's screener labels the setup. |
| `dist_to_entry` | op (<, <=, >, >=, =) and a number | % | Distance between the last price and the trigger, absolute percent. |
| `min_win_rate` | op (<, <=, >, >=, =) and a number | % | Historical win rate of this pattern on this ticker, percent. |
| `min_avg_r` | op (<, <=, >, >=, =) and a number | R | Average R per past trade on this ticker. |
| `min_expected_r` | op (<, <=, >, >=, =) and a number | R | Expected R, TradingPal's ticker-plus-family estimate. |
| `min_rr` | op (<, <=, >, >=, =) and a number | ratio | Reward to risk: distance to the target divided by distance to the stop. |
| `target_move` | op (<, <=, >, >=, =) and a number | % | Trigger to target, percent. |
| `pre_pattern_move` | op (<, <=, >, >=, =) and a number | % | The run-up before the pattern, percent. |
| `price` | op (<, <=, >, >=, =) and a number | $ | Last price. |
| `market_cap` | op (<, <=, >, >=, =) and a number | $ | Market capitalization. |
| `avg_volume` | op (<, <=, >, >=, =) and a number | shares | 20-day average daily volume. |
| `dollar_volume` | op (<, <=, >, >=, =) and a number | $ | 20-day average dollar volume. |
| `dollar_volume_30d` | op (<, <=, >, >=, =) and a number | $ | Price times the 30-day average volume. |
| `current_dollar_volume` | op (<, <=, >, >=, =) and a number | $ | Price times today's volume. |
| `current_volume` | op (<, <=, >, >=, =) and a number | shares | Today's volume; the day may still be in progress. |
| `volume_ratio` | op (<, <=, >, >=, =) and a number | x | The last full day's volume over its 20-day average (1 = average). |
| `volume_percentile` | op (<, <=, >, >=, =) and a number | 0 to 100 | Full-day volume rank over the last three months (100 = highest). |
| `rsi` | op (<, <=, >, >=, =) and a number | 0 to 100 | 14-day RSI. |
| `adr` | op (<, <=, >, >=, =) and a number | % | 14-day average daily range as a percent of price. |
| `atr` | op (<, <=, >, >=, =) and a number | % | 14-day average true range, gaps included, as a percent of price. |
| `volatility` | op (<, <=, >, >=, =) and a number | % | 20-day realized volatility. |
| `ema_distance` | op (<, <=, >, >=, =) and a number; period 8, 21, 60 | % | Percent above (+) or below (-) the exponential moving average; period 8, 21 or 60. |
| `ema_cross` | op (<, <=, >, >=, =) and a number | % | Percent EMA8 is above EMA21; at least 0 includes equality. |
| `sma_distance` | op (<, <=, >, >=, =) and a number; period 20, 50, 200 | % | Percent above (+) or below (-) the simple moving average; period 20, 50 or 200. |
| `price_above_sma` | value true or false; period 20, 50, 200 | true/false | true when price is above the simple moving average; period 20, 50 or 200. |
| `sma_cross` | value true or false | true/false | true when the 50-day average is above the 200-day. |
| `performance` | op (<, <=, >, >=, =) and a number; period 5, 21, 63, 126, 252, ytd | % | Price change over the period: 5, 21, 63, 126 or 252 sessions, or ytd. |
| `daily_change` | op (<, <=, >, >=, =) and a number | % | One-day price change. |
| `change_from_open` | op (<, <=, >, >=, =) and a number | % | Open to last, percent. |
| `gap` | op (<, <=, >, >=, =) and a number | % | Opening gap. |
| `distance_from_high` | op (<, <=, >, >=, =) and a number; period 20, 50, 252, or a calendar window (1w, 2w, 4w, 13w, 26w, 52w, 1m, 2m, 3m, 6m, 12m) | % | Percent below the period high; period 20, 50 or 252 sessions, or 1w to 12m. |
| `distance_from_low` | op (<, <=, >, >=, =) and a number; period 20, 50, 252, or a calendar window (1w, 2w, 4w, 13w, 26w, 52w, 1m, 2m, 3m, 6m, 12m) | % | Percent above the period low; period 20, 50 or 252 sessions, or 1w to 12m. |
| `asset_type` | value one of crypto, etf, stock |  | stock, etf or crypto. |
| `exchange` | value, a label |  | Market identifier code: XNYS, XNAS, ARCX, BATS or XASE. |
| `industry` | value, a label |  | One of: Semiconductors, Software, Hardware & Electronics, Internet & Media, Telecom, Biotech & Pharma, Healthcare Equipment & Services, Banks, Insurance, Capital Markets & Asset Management, Fintech & Payments, Real Estate, Oil & Gas, Utilities & Renewables, Metals & Mining, Chemicals & Materials, Industrials & Construction, Aerospace & Defense, Transport & Logistics, Retail & E-Commerce, Consumer Goods & Services, Food & Beverage, Restaurants, Travel & Leisure, Autos & Mobility, Crypto, Broad Market & Index, Bonds & Fixed Income, Commodities. |
| `country` | value, a label |  | US or GLOBAL. |

## Stat block

A track record, at family scope (`/families`) or ticker scope (inside a setup):

```json
{
  "scope": "family", "family": "falling_wedge", "symbol": null,
  "win_rate_pct": 54.0, "sample_size": 1287, "wins": 695, "losses": 592,
  "avg_win_pct": 8.1, "avg_loss_pct": -4.2, "median_win_pct": 6.3,
  "avg_trade_pct": 2.4, "avg_r": 0.41, "expected_r": 0.41, "profit_factor": 1.61,
  "avg_hold_bars": 11.2,
  "backtest_range": ["2006-09-25", "2026-09-25"],
  "measured": "Every detector-confirmed breakout … open trades are excluded.",
  "summary": "Falling wedge: 1,287 resolved trades from 2006-09-25 to 2026-09-25; 54% reached the target, average win +8.1%, average loss -4.2%, median win +6.3%."
}
```

`measured` states the rule: entry at the breakout bar's close, stop frozen at entry and filled on a touch or an adverse gap open, a bar that reaches both target and stop counts as a loss, bullish converging patterns ride the 10-day average after the target, bearish breaks exit at the target, open trades excluded. Quote `summary` whole.

## History row

From `/history`: `setup_id`, `lineage_id`, `family`, `family_label`, `pattern_type`, `direction`, `outcome` (`win`, `loss`, `open`), `entry_date`, `entry_price`, `stop_price`, `target_price`, `exit_date`, `exit_price` (resolved only), `realized_move_pct`, `target_move_pct`, `r_multiple`, `unrealized_r` (open only), `hold_bars`, `attempt` (`number`, `count`: the n-th breakout attempt of that pattern).

## Errors

```json
{"error": "setup_not_found",
 "message": "no current setup NVDA:D:falling_wedge:500 on NVDA",
 "fix": "Setups change nightly; re-read the current patterns for the symbol.",
 "docs_url": "https://tradingpal.io/developers"}
```

| Code | HTTP | Meaning |
|---|---|---|
| `missing_api_key` | 401 | No `Authorization: Bearer tp_live_…` header |
| `invalid_api_key` | 401 | Unknown or revoked key |
| `forbidden` | 403 | The account is not on Premium+; the `fix` says where to get it |
| `rate_limited` | 429 | Wait `details.retry_after_seconds` |
| `bad_request`, `invalid_symbol`, `unknown_symbol`, `ticker_cap_exceeded` | 400 | The message says which input |
| `unknown_family`, `setup_not_found`, `chart_unavailable`, `not_a_demo_symbol` | 404 | The fix says what to call instead |

Tickers use the exchange symbol in upper case (`BRK.B`, not `BRK-B`). A ticker the nightly does not cover answers `available: false` with `reason: not_covered`, not an error.

## Example: what did the scan find last night

1. `GET /scan?limit=10` → the ten best-ranked forming setups across the market (`get_scan` with `limit: 10`). For "about to break out", add `max_distance_pct=2`; for one family, `family=falling_wedge`; for the user's own screen, `filters` (see **Scan filters**).
2. Present each: rank, symbol, family and direction, Typical return (`expected_gain_pct`), Win rate (`win_rate_pct`, with `win_rate_trades` trades), Distance to trigger (`distance_to_trigger_pct`), the trigger price and rule, the stop and target, and the family `summary` sentence. Offer the chart (`get_pattern_chart`, or `/chart.png`).
3. End with `as_of_session` and the disclaimer.

## Example: what is setting up on my tickers

1. `GET /tickers` → the tracked list.
2. For each ticker, `GET /tickers/{symbol}/patterns` → skip empty `setups`.
3. For each setup, present: the family and direction, the trigger price and rule, the stop and target, the family `summary` sentence, the ticker `summary` when present, and the chart from `/chart.png`.
4. End with `as_of_session` and the disclaimer.

Stable contract: field names in v1 do not change; v1 stays available at least 12 months after a v2 ships.
