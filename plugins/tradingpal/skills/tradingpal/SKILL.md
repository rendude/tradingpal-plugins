---
name: tradingpal
description: >-
  Read TradingPal's breakout patterns (the wedges, pennants and triangles price
  squeezes into before a big move) on the stocks & crypto the user tracks:
  current setups with lines, trigger, stop and target, the 20-year track
  record per pattern family and per ticker, past occurrences with outcomes,
  and a chart image. Use when the user asks what is setting up on a stock or crypto ticker,
  how a pattern has performed, or for a chart of a setup. Needs a TradingPal
  signed-in MCP connection, or an API key (tp_live_…) in TRADINGPAL_API_KEY.
---

# TradingPal pattern API

> TradingPal finds breakout patterns on stocks & crypto: the wedges, pennants and triangles price squeezes into before a big move. It scans every night and keeps a 20-year record of how often each pattern family, and each ticker, paid off. This API and MCP server give an agent the current setups on the tickers its user tracks, the lines and levels to draw, the track record behind them, past occurrences with outcomes, and a chart image. Nothing is computed per request; every answer is the same stored nightly result the site serves.

Base URL: `https://api.tradingpal.io/api/v1`. MCP endpoint: `https://api.tradingpal.io/mcp` (Streamable HTTP). Human docs and key management: https://tradingpal.io/developers. Machine index: `GET https://api.tradingpal.io/api/v1/` (JSON). OpenAPI 3.1: `https://api.tradingpal.io/api/v1/openapi.json`. This file: `https://api.tradingpal.io/api/v1/llms.txt`. Installable skill: `https://api.tradingpal.io/api/v1/skill.md`.

## Rules for agents

1. **Never present a win rate alone.** Every track record carries a `summary` sentence with the sample size, the win rate, the average win and the average loss. Quote it whole. "54% win rate" without "average loss -4.2%, 1,287 trades" is the misuse this data exists to prevent.
2. **Say which session the data describes.** Every response carries `as_of_session` (the last trading day the nightly covered). Setups change nightly; do not cache them across days.
3. **Lines are two dated endpoints on a log-scale chart.** Draw a straight line between `start` and `end` with the price axis in log scale. There is no slope field on purpose.
4. **This is educational chart analysis, not advice.** Repeat the `disclaimer` when presenting a setup as something the user might act on.
5. **Tracked tickers are the unit.** Pattern, history and chart calls work only on tickers the account tracks. When a call fails with `ticker_not_tracked`, add the ticker (the error's `fix` says how) and retry.

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
| `GET /tickers` | yes | Tracked tickers, each with `covered` (true when the nightly covers it) |
| `PUT /tickers` body `{"symbols": ["NVDA", "AAPL"]}` | yes | Replaces the whole tracked set, all-or-nothing; `cap` says how many fit |
| `GET /tickers/{symbol}/patterns` | yes | Current setups on one tracked ticker (see **Setup**) |
| `GET /tickers/{symbol}/history?limit=50&outcome=all` | yes | Past occurrences on that ticker with what happened next, newest first; `outcome` is `all`, `resolved` or `open`; `limit` up to 100 |
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
| `get_ticker_patterns(symbol)` | yes | `GET /tickers/{symbol}/patterns`. The one to call for "what is setting up on X". |
| `get_pattern_history(symbol, limit?, outcome?)` | yes | `GET /tickers/{symbol}/history` |
| `get_pattern_chart(symbol, setup_id)` | yes | The PNG as an image content block, so the model and the user both see it |
| `list_tracked_tickers()` | yes | `GET /tickers` |
| `set_tracked_tickers(symbols)` | yes | `PUT /tickers`: replaces the list, so include the tickers to keep |
| `get_family_track_record(family?)` | no | `GET /families` or `GET /families/{family}/stats` |
| `demo(symbol)` | no | The keyless demo, with the first setup's chart as an image |

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
{"error": "ticker_not_tracked",
 "message": "NVDA is not one of the tickers this account tracks",
 "fix": "Add it to the tracked tickers (3 of 50 slots used): PUT /api/v1/tickers, or the set_tracked_tickers MCP tool.",
 "docs_url": "https://tradingpal.io/developers"}
```

| Code | HTTP | Meaning |
|---|---|---|
| `missing_api_key` | 401 | No `Authorization: Bearer tp_live_…` header |
| `invalid_api_key` | 401 | Unknown or revoked key |
| `forbidden` | 403 | The account is not on a plan with API access |
| `ticker_not_tracked` | 403 | Add the ticker, then retry |
| `rate_limited` | 429 | Wait `details.retry_after_seconds` |
| `bad_request`, `invalid_symbol`, `unknown_symbol`, `ticker_cap_exceeded` | 400 | The message says which input |
| `unknown_family`, `setup_not_found`, `chart_unavailable`, `not_a_demo_symbol` | 404 | The fix says what to call instead |

Tickers use the exchange symbol in upper case (`BRK.B`, not `BRK-B`). A ticker the nightly does not cover answers `available: false` with `reason: not_covered`, not an error.

## Example: what is setting up on my tickers

1. `GET /tickers` → the tracked list.
2. For each ticker, `GET /tickers/{symbol}/patterns` → skip empty `setups`.
3. For each setup, present: the family and direction, the trigger price and rule, the stop and target, the family `summary` sentence, the ticker `summary` when present, and the chart from `/chart.png`.
4. End with `as_of_session` and the disclaimer.

Stable contract: field names in v1 do not change; v1 stays available at least 12 months after a v2 ships.
