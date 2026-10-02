# TradingPal for Claude Code, Codex and Cursor

TradingPal finds breakout patterns on stocks & crypto: the wedges, pennants and triangles price squeezes into before a big move. It scans every night and keeps a twenty-year record of how often each pattern, and each ticker, paid off. This repository packages the [TradingPal MCP server](https://tradingpal.io/developers/mcp) and its skill as a plugin, so an agent can answer "what is setting up on my tickers?" with the setups, the two lines to draw, the trigger, stop and target, the track record behind them and a chart image.

No key to copy: the plugin connects to `https://api.tradingpal.io/mcp`, and the first time it is used your agent opens the browser so you can sign in to TradingPal and approve the connection. The approval becomes a key named after the app on your [keys page](https://tradingpal.io/developers/keys), where you can revoke it any time. Docs for everything: [tradingpal.io/developers](https://tradingpal.io/developers).

## Claude Code

```bash
claude plugin marketplace add rendude/tradingpal-plugins
claude plugin install tradingpal@tradingpal
```

Then run `/mcp`, pick `tradingpal` and sign in. Ask: "What is setting up on my tickers?"

## Codex

```bash
codex plugin marketplace add rendude/tradingpal-plugins
```

Open the Plugins directory, choose the TradingPal marketplace and install the plugin; sign in when it asks. Without the plugin, the server alone works too:

```bash
codex mcp add tradingpal --url https://api.tradingpal.io/mcp
codex mcp login tradingpal
```

## Cursor

Add the server in Cursor's MCP settings with the URL `https://api.tradingpal.io/mcp` and no headers, then click **Needs login** next to it. The [docs](https://tradingpal.io/developers/mcp) have a one-click "Add to Cursor" button.

## What is in the plugin

| Path | What it is |
| --- | --- |
| `plugins/tradingpal/.mcp.json` and `mcp.json` | The MCP server, for Claude Code and for Codex. |
| `plugins/tradingpal/skills/tradingpal/SKILL.md` | The agent reference: routes, tools, shapes and the presentation rules (same text as `https://api.tradingpal.io/api/v1/skill.md`). |
| `plugins/tradingpal/.claude-plugin/plugin.json` and `plugin.json` | The manifests. |
| `.claude-plugin/marketplace.json` and `.agents/plugins/marketplace.json` | The marketplaces the install commands above add. |

## Try it without an account

`https://api.tradingpal.io/mcp/demo` connects with nothing and answers the `demo` and `get_family_track_record` tools on a fixed set of large caps. The REST equivalent is `https://api.tradingpal.io/api/v1/demo/NVDA`.

## Rules for presenting the data

Never present a win rate without its sample size and average loss; every track record carries a `summary` sentence to quote whole. Say which session the data describes. Lines are two dated endpoints on a log-scale chart. This is educational chart analysis, not advice. The skill carries the full rules.
