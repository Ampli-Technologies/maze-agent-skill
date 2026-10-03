---
name: ampli-composer
description: Build strategy compositions for Ampli Maze Studio and send buy, sell or exit signals to a deployed Ampli agent's webhook. Use when the user asks to design, generate or change an Ampli treasury or agent strategy, or to trade a symbol through an Ampli agent, a TradingView alert or an Ampli webhook URL.
license: Proprietary. See LICENSE.
metadata:
  catalogVersion: "2026.10.3.3"
---

# Ampli Composer

Ampli runs treasury agents. Each agent follows a strategy composition: a
graph of blocks drawn in Ampli Maze Studio (triggers, capital,
intelligence, guards and actions), bounded by the organization's treasury
policy. This skill does two jobs:

1. **Build a strategy**: write a composition the user imports into Maze
   Studio.
2. **Send a signal**: tell a deployed agent to buy, sell or exit a symbol
   by posting to its webhook URL.

Tell them apart by the request. "Make a strategy that...", "change this
graph" and "how would I set up..." are job 1. "Buy ETH", "close my TSLA
short", "send this alert" and "test the webhook" are job 2. A strategy
that trades on outside signals needs both: build it with a `webhook`
trigger, then send it signals once the user has deployed it and created
its URL.

This skill was generated from catalog `2026.10.3.3`. Maze Studio
rejects kinds and config keys it does not know, so if the user's Studio
reports a different catalog version, ask them to download the skill again
(Maze Studio -> AI Skill).

## Job 1: build a strategy

Your deliverable is one JSON code block, which the user imports through
Maze Studio -> AI Skill -> Import composition.

1. Confirm the strategy class with the user: `treasury-yield`,
   `directional`, `market-neutral` or `hedged-carry`.
2. Ask what the committed treasury policy permits, above all whether
   Hyperliquid is enabled as a trading venue, before designing anything
   that trades there. A composition may only narrow the policy.
3. Start from the closest composition in `references/examples.md`. Map
   the user's wording to blocks with `references/phrases.md`.
4. Take every kind, config key and option from the node catalog in
   `references/structures.md`. Never invent one.
5. Wire triggers, then capital, intelligence, guards and actions. Name
   `sourceHandle` on every edge out of a block with several outputs, and
   send every guard's `blocked` output to one shared `journal`.
6. Output the JSON. Tell the user to read the validation report before
   accepting the import, then Rehearse. For a webhook strategy, rehearse
   buy, sell and exit with the Webhook signal scenario; each action
   should end on would-send.

### Output contract

- Exactly one JSON code block, no prose inside it.
- `version` is `2`.
- `catalogVersion` is `2026.10.3.3`. Unknown kinds or config keys
  are errors; out-of-range numbers are warnings the user must accept.
- `strategyClass` is required.
- Node `id` values are unique short slugs.
- `position` is optional; the Studio lays out nodes without one.
- Partial `config` is fine; omitted keys take the catalog defaults.
- The full schema, with `defs`, `group`, `positionTag` and edge
  `payload`, is in `references/structures.md`.

### Rules most often missed

- Trigger nodes have no input; `payout` and `journal` have no output.
- Put a guard directly before every action, and a `kill-switch` in any
  graph that trades.
- In the treasury policy an empty trading-venue list enables none: every
  Hyperliquid block is refused until Hyperliquid is switched on.
- A block reads the webhook signal's symbol only when its field is set to
  "Webhook signal": `symbolFrom` on `perp-position` and `spot-position`,
  `asset` on `twap`, `from` or `to` on `swap`. Maze Studio shows these as
  "Received from Webhook Signal". Set it on every block after the trigger
  that should act on the signalled symbol.
- Never filter tickers in the graph to stand in for the policy. The
  policy's Hyperliquid universe and token whitelist decide what trades.

## Job 2: send a signal

### What you need

- The agent's webhook URL. The user creates it on the deployed agent:
  Agents -> the agent's journal -> Webhook -> Create URL. It is shown once
  and looks like `https://controlroom.ampli.net/api/composer/webhooks/<token>`. The token in the
  path is the only credential, so treat the URL as a secret: read it from
  an environment variable such as `AMPLI_WEBHOOK_URL`, never write it into
  a file that may be committed, and never print it in full.
- An active agent whose graph has a `webhook` trigger wired for the
  action you send. A sell sent to a graph that ignores sells does nothing.

### Before you send

- A signal trades treasury funds as soon as it lands. State the action,
  symbol and size back to the user and wait for a yes before each send,
  unless they have told you in this conversation to send without asking.
- Know what the action does in this agent's graph. In the Hyperliquid
  Signal Trader, `sell` closes a long and opens a short; in the Base
  Signal Accumulator, `sell` sells the treasury's holding. When unsure,
  ask; `exit` only ever closes.
- Give each signal an `id`. A repeated id is ignored, so reuse the same id
  when you retry a signal and it cannot trade twice.

### Send

```bash
curl -sS -X POST "$AMPLI_WEBHOOK_URL" \
  -H 'Content-Type: application/json' \
  -d '{"action":"buy","symbol":"ETH","size_pct":50,"id":"eth-buy-2026-10-03-1"}'
```

Plain text works too: `buy ETH $500`, `sell BTC 25%`, `exit SOL`. A bare
number is ambiguous and ignored; write `$500`, `500usd` or `25%`.

### Payload

- `action`: `buy`, `sell` or `exit`. `long`, `short`, `close` and `flat`
  are accepted.
- `symbol`: what to trade; see Symbols below.
- `size_pct`: above 0 and at most 100. Scales what the block would do on
  its own.
- `size_usd`: honoured only when the trigger's `maxSizeUsd` is above 0,
  and capped at it and at the block's own size. It never sizes a block up.
- `id`: deduplicates. `price` and `message` are recorded in the journal.

Every accepted alias and the TradingView alert message are in
`references/structures.md`; ready-made senders are in
`references/examples.md`.

### Symbols

- Hyperliquid perps by ticker: `ETH`, `BTC`, `SOL`. Exchange prefixes and
  quote suffixes are dropped, so `BINANCE:ETHUSDT.P` is `ETH`.
- Hyperliquid HIP-3 markets (tokenized stocks and other perps deployed by
  builders) as `dex:TICKER`, such as `xyz:TSLA`. A bare `TSLA` or
  `NASDAQ:TSLA` takes the main Hyperliquid listing when there is one,
  otherwise the most-traded HIP-3 listing the platform reads. Name the
  dex when it matters.
- Base tokens by `0x` address.
- The treasury policy decides what may trade. A symbol outside it is
  refused and journalled (`POLICY_SYMBOL_NOT_IN_UNIVERSE`,
  `POLICY_TOKEN_NOT_APPROVED`). That is not a fault in your request: tell
  the user which policy change it needs instead of retrying.

### Read the response

- 202 `{"accepted":true,"signalId":...,"action":...,"symbol":...,"queued":...}`:
  accepted, and the run starts now. Check `symbol` is what you meant; it
  is the symbol as the agent read it. The response does not say whether
  anything traded. The outcome is in the agent's journal in Maze Studio.
- 200 with `duplicate: true`: that id was already taken; nothing new runs.
- 400: unreadable payload. Fix it; do not resend it unchanged.
- 404: unknown or revoked URL. Ask the user for the current one.
- 409: the agent is not active.
- 413: the body is over 16 KB.
- 422: the agent has no webhook flow.
- 429: more than 20 signals are waiting. Wait, then retry with the same id.
- 503: temporarily unavailable. Retry later with the same id.

## References

- `references/structures.md`: composition schema, graph rules, the
  treasury policy, Hyperliquid symbols, webhook wiring and the full signal
  payload, reason codes, and the node catalog.
- `references/examples.md`: complete compositions for every shipped
  template, and signal senders (curl, TradingView, Python).
- `references/phrases.md`: what users say, and the blocks or payload it
  means.
