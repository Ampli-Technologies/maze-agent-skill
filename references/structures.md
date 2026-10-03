# Structures

Generated from Maze Studio catalog `2026.10.3.3`.

## Composition schema

```json
{
  "version": 2,
  "catalogVersion": "2026.10.3.3",
  "name": "string",
  "strategyClass": "treasury-yield",
  "defs": {
    "examplePair": {
      "type": "instrument-pair"
    }
  },
  "nodes": [
    {
      "id": "string",
      "kind": "string",
      "group": "optional",
      "positionTag": "optional",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "<field>": "value"
      }
    }
  ],
  "edges": [
    {
      "source": "node id",
      "target": "node id",
      "sourceHandle": "one of the source's outputs, e.g. primary | fallback | passed | blocked | buy | sell | exit (required when the source has several)",
      "payload": "flow"
    }
  ]
}
```

- `strategyClass`: `treasury-yield` | `directional` | `market-neutral` |
  `hedged-carry`.
- `defs` holds shared definitions; reference them in config values as
  `{ "$ref": "defs.myKey" }`.
- `group` gathers nodes into a sub-composition; `positionTag` links the
  blocks that open, watch and close one position.
- Edge `payload`: `flow` (default) | `signal` | `order-intent` | `verdict`
  | `data`.

## Graph rules

- Trigger nodes have NO input; every flow should start from at least one.
- `payout` and `journal` are terminal sinks (no output).
- Intelligence nodes have `primary` and `fallback` outputs.
- A node listing several outputs needs `sourceHandle` on every edge out of
  it, named from its `outputs` below. An unnamed edge out of a `webhook` is
  an error (`BAD_SOURCE_HANDLE`).
- Guard nodes have `passed` and `blocked` outputs. Wire `blocked` to a
  `journal` (or other sink) so blocked events are auditable. Unwired
  blocked handles produce a warning.
- One shared journal per graph is the house style: point every `blocked`
  handle at the same sink rather than adding a journal per guard. The
  reason code already says which guard fired.
- Prefer a guard immediately upstream of every action. Unguarded actions
  produce a warning; global treasury policies still apply and may only be
  narrowed by strategy guards, never widened.
- Keep 4-14 nodes per group; use `group` when the canvas grows.
- Entry actions that open a position should set `positionTag`; exit and
  monitor nodes that reference a tag must match an opener.

## The treasury policy comes first

The fund manager commits a risk policy before choosing an agent, so the
policy already exists when you author. A composition runs underneath it and
may only narrow it, never widen it. Rehearsal compares the two and reports
the conflicts, so a strategy that contradicts the policy is not a surprise
at run time -- it simply never trades.

- Lending protocols: an empty allowed list means unrestricted.
- Trading venues: an empty list means **none enabled**. Hyperliquid must be
  switched on in Policies -> Market Allowance and Limits -> Trading venues,
  or every Hyperliquid block is refused with `POLICY_VENUE_NOT_PERMITTED`.
- A guard looser than the policy never fires; the policy binds first. Set
  guard limits at or inside the policy numbers.
- Leaving a venue on "Best available" delegates the choice to the optimiser,
  which only picks from the permitted set, so it cannot breach policy and is
  not counted against venue-diversification minimums.
- A Base token's address must be on the policy whitelist, as USDC is, to be
  bought or sold (`POLICY_TOKEN_NOT_APPROVED` otherwise).

## Hyperliquid symbols

`symbol` is free text, not a fixed list: any listed Hyperliquid perp is
addressable. What may actually trade is decided by the treasury's
Hyperliquid universe in policy, which is one of:

- `rules` -- any listing clearing the quality floors (24h volume, open
  interest, open-interest rank). The listing-age floor defaults to 0
  because Hyperliquid publishes no listing date; a non-zero value refuses
  every symbol until a listing catalog exists.
- `allowlist` -- only the named tickers, floors ignored.
- `allowlist-and-rules` -- named tickers that also clear the floors.

The denylist always wins, and a listing whose metrics are unknown fails
closed under any floor. A hand-pinned `symbol`, a screener pick and a
webhook signal's ticker are all checked against this at commit, so an
illiquid ticker is refused with `POLICY_SYMBOL_NOT_IN_UNIVERSE` rather than
reaching the venue. Prefer `pair-screener`, which filters candidates
through the same universe and binds the winner into a def the rest of the
graph reads via `{ "$ref": "defs.pairA" }`, over naming a ticker by hand.

### HIP-3 markets

HIP-3 markets are perps deployed by builders on their own Hyperliquid
dexes, mostly tokenized stocks. Name one `dex:TICKER`, dex in lower case:
`xyz:TSLA`.

- Only USDC-margined dexes the platform is configured to read trade
  (`xyz` unless the operator adds more). An opening order on any other dex
  ends `PERP_NO_MARKET`.
- They are perps only: `spot-position` refuses one (`SPOT_NO_MARKET`).
- A bare ticker from a signal (`TSLA`, `NASDAQ:TSLA`) takes the main
  Hyperliquid listing when there is one, otherwise the HIP-3 listing with
  the most 24h volume.
- In a policy allowlist, write them as listed (`xyz:TSLA`). Volume, open
  interest and rank floors apply to them as to any listing; open-interest
  rank is counted within each dex.
- Before the first HIP-3 order the agent turns on Hyperliquid's dex
  abstraction for the account, so one USDC balance margins every dex.
- Some HIP-3 markets are isolated margin only; Hyperliquid margins those
  positions in isolation.
- `basis-check` has no spot leg to compare for them, and `pair-screener`
  leaves them out unless the policy allowlist names them.

## Webhook trigger

The `webhook` trigger lets an outside source -- a TradingView alert, the
user's own model -- tell a deployed agent to buy, sell or exit a symbol.
The trigger takes a signal for any symbol; the blocks after it decide what
trades. The graph fixes venue, guards and the largest size; the signal
picks the direction, names the symbol and may shrink the size. The
treasury policy decides which symbols may trade.

### Blocks that receive the signal's symbol

A block reads the symbol only when the field below is set to "Webhook
signal"; the node catalog marks these fields with `receives`. The symbol
passes along the whole path, so every block after the trigger that is set
this way reads the same symbol.

- `perp-position` / `spot-position`: `symbolFrom` "Webhook signal". The
  ticker must clear the policy's Hyperliquid universe, checked when the
  leg is planned and again at commit (`POLICY_SYMBOL_NOT_IN_UNIVERSE`);
  reduce-only closes are let through. Each ticker keeps its own position
  as `positionTag:TICKER` (`hl-long:SOL`, `hl-long:xyz:TSLA`).
- `twap`: `asset` "Webhook signal" buys the token the signal names, by
  ticker or Base address.
- `swap`: `from` or `to` "Webhook signal" (never both). `from` sells a share
  of the treasury's on-chain balance of that token, the exit path back to
  USDC. Use Orbs dTWAP or Uniswap v3 for tokens named by address.
- Requiring a Base token on the whitelist for the buy keeps the agent from
  buying what it could not sell.
- A block that names its own instrument acts only on a signal for that
  instrument or naming none, and ends `SIGNAL_OTHER_INSTRUMENT` (handle
  `none`) otherwise. A block reading the signal ends `SIGNAL_NO_SYMBOL` when
  the signal names none.

### Wiring

- Name `sourceHandle` `buy`, `sell` or `exit` on every edge out of a
  `webhook`. Leave an output unwired to ignore that action.
- Each run takes the oldest waiting signal. A signal not acted on within
  `maxAge` expires (`WEBHOOK_EXPIRED`).
- The payload size is read by `twap` (its budget), `swap` with `sizeFrom`
  "Share of balance", and `perp-position` / `spot-position` opening with
  `sizeFrom` "Account" or "Account, one of two legs". Other blocks ignore it.
- A signal that would open the side already open on that position takes
  `none` (`PERP_NOTHING` / `SPOT_NOTHING`), so repeated alerts do not
  pyramid.
- Put guards between the trigger and every opening action, as for any
  other trigger: `exposure-guard`, `margin-health` for perps,
  `max-allocation` and `slippage-cap` for Base swaps. Add a `kill-switch`.
- The policy binds: Hyperliquid must be enabled as a trading venue, and
  Orbs dTWAP must be allowed if the protocol list is restricted.

### Patterns

- Hyperliquid perps, long and short (Hyperliquid Signal Trader): `buy` ->
  `perp-position` reduce-only close of the short tag -> guards ->
  `perp-position` Long with `sizeFrom` "Account" and its own
  `positionTag`; `sell` mirrors it; `exit` closes both tags reduce-only.
  Every perp block has `symbolFrom` "Webhook signal". Wire the close's
  `default` and `none` outputs both onward, so a flip and a fresh open take
  the same path.
- Accumulate a Base token (Base Signal Accumulator): `buy` ->
  `max-allocation` -> `slippage-cap` -> `twap` with `asset` "Webhook
  signal". `exit` and `sell` -> `slippage-cap` -> `swap` `from` "Webhook
  signal" `to` USDC with `sizeFrom` "Share of balance" on Orbs dTWAP. Tell
  the user to whitelist every token address the source may send.

### Signal payload

`POST` JSON or a plain-text message, at most 16 KB:

```json
{ "action": "buy", "symbol": "ETH", "size_pct": 50, "price": 3400, "id": "tv-1712", "message": "breakout" }
```

- `action` (also `side`, `signal`, `order_action`): `buy` | `sell` | `exit`;
  `long`, `short`, `close`, `flat` are accepted. `market_position: "flat"`
  always means exit.
- `symbol` (also `ticker`, `coin`, `asset`, `token`): what to trade.
  Exchange prefixes and quote suffixes are dropped, so `BINANCE:ETHUSDT.P`,
  `ETH-PERP` and `WETH` are all `ETH`. A lower-case or known dex prefix is
  kept as a HIP-3 market (`xyz:TSLA`); a chart exchange such as `NASDAQ:` is
  dropped. A 0x address names a Base token.
- `size_pct` (also `sizePct`, `percent`): above 0 and at most 100; scales
  what the block would do on its own.
- `size_usd` (also `sizeUsd`, `amount_usd`, `notional`): honoured only when
  the trigger's `maxSizeUsd` is above 0, and capped at `maxSizeUsd` and at
  the block's own size. It never sizes a block up.
- `price` (also `close`) and `message` are recorded in the journal.
- `id` (also `alertId`, `signalId`): a repeated id is ignored.

Plain text works too: `buy ETH $500`, `sell BTC 25%`, `exit SOL`,
`buy xyz:TSLA $250`. A bare number is ignored as ambiguous; write `$500`,
`500usd` or `25%`.

The URL is created on the deployed agent (Agents -> the agent's journal ->
Webhook -> Create URL). It is shown once; replacing it revokes the old one.
Signals are only taken while the agent is active, and the run starts as
soon as the signal lands.

Responses: 202 queued, 200 duplicate id, 400 unreadable payload, 404
unknown URL, 409 agent not active, 413 body over 16 KB, 422 the agent has
no webhook flow, 429 more than 20 signals waiting, 503 temporarily
unavailable.

## Reason codes

When describing outcomes, only use these fixed codes:

- `AUTO_LAYOUT_APPLIED`
- `BASIS_DIVERGED`
- `BASIS_OK`
- `BLOCKED_HANDLE_UNWIRED`
- `BORROW_FAILED`
- `BORROW_HEALTH_FAIL`
- `BORROW_LIMITED`
- `BORROW_NONE`
- `BORROW_SUBMITTED`
- `BUFFER_BREACH`
- `BUFFER_TOPPED_UP`
- `CARRY_BASIS_DIVERGED`
- `CARRY_BELOW_COST`
- `CARRY_FUNDING_FLIPPED`
- `CARRY_LEG_MISMATCH`
- `CARRY_MARGIN_STRESS`
- `CARRY_VENUE_INCIDENT`
- `CATALOG_VERSION_MISMATCH`
- `CLASS_BLOCK_DENIED`
- `DEBT_CHECK_UNREADABLE`
- `DEBT_COVERED`
- `DEBT_NONE`
- `DEBT_SHORT`
- `DEBT_SHORT_PARTIAL`
- `DEFAULTS_FILLED`
- `DEPOSIT_RECEIVED`
- `FEED_DEVIATION`
- `FEED_HEALTHY`
- `FEED_SOURCE_REJECTED`
- `FEED_STALE`
- `FUNDING_READ`
- `FUNDING_UNAVAILABLE`
- `GRAPH_CYCLE`
- `GUARD_ALLOCATION_BLOCKED`
- `GUARD_ALLOCATION_PASSED`
- `GUARD_COOLDOWN_BLOCKED`
- `GUARD_COOLDOWN_PASSED`
- `GUARD_EV_BLOCKED`
- `GUARD_EV_PASSED`
- `GUARD_EXPOSURE_BLOCKED`
- `GUARD_EXPOSURE_PASSED`
- `GUARD_HOLD_BLOCKED`
- `GUARD_HOLD_PASSED`
- `GUARD_MARGIN_BLOCKED`
- `GUARD_MARGIN_PASSED`
- `GUARD_QUARANTINE_BLOCKED`
- `GUARD_QUARANTINE_PASSED`
- `GUARD_SLIPPAGE_BLOCKED`
- `GUARD_SLIPPAGE_PASSED`
- `HEDGE_BORROW`
- `HEDGE_CHECK_UNREADABLE`
- `HEDGE_IN_BAND`
- `HEDGE_LIMITED`
- `HEDGE_NO_POSITION`
- `HEDGE_REPAY`
- `IDLE_CAPITAL_DETECTED`
- `IMPORT_ACCEPTED`
- `IMPORT_BLOCKED`
- `INSUFFICIENT_HISTORY`
- `JOURNAL_WRITTEN`
- `KILL_SWITCH_CLEAR`
- `KILL_SWITCH_STAGE1`
- `KILL_SWITCH_STAGE2`
- `LEGS_FLAT`
- `LEGS_MATCHED`
- `LEG_OUT_INCIDENT`
- `LEG_OUT_WAITING`
- `LEND_REVERTED`
- `LEND_SUBMITTED`
- `LOOP_AT_TARGET`
- `LOOP_EXIT`
- `LOOP_FAILED`
- `LOOP_FOLD`
- `LOOP_HEALTH_FAIL`
- `LOOP_HEALTH_LOW`
- `LOOP_NO_CAPITAL`
- `LOOP_NO_ORDER`
- `LOOP_OPEN`
- `LOOP_ORDER_OPEN`
- `LOOP_OVER_TARGET`
- `LOOP_PENDING`
- `LOOP_SPREAD_LOW`
- `LOOP_SPREAD_NEGATIVE`
- `LOOP_STEP`
- `LOOP_STUCK`
- `LOOP_SWAP_STALLED`
- `LOOP_UNREADABLE`
- `LOOP_UNWINDING`
- `LP_EXITED`
- `LP_EXIT_FAILED`
- `LP_EXIT_NONE`
- `LP_IN_RANGE`
- `LP_NEAR_EDGE`
- `LP_NO_POSITION`
- `LP_OPENED`
- `LP_READ_FAILED`
- `LP_RERANGE_DUE`
- `LP_REVERTED`
- `LP_RULES_CARRY`
- `LP_RULES_HEALTH`
- `LP_RULES_HOLD`
- `LP_RULES_LOSS`
- `LP_RULES_NO_POSITION`
- `LP_RULES_OUT_OF_RANGE`
- `LP_RULES_UNREADABLE`
- `MARKET_OK`
- `MARKET_OUT_OF_RANGE`
- `NOTIFY_SENT`
- `ORPHAN_NODE`
- `PAIRED_CLOSE_OK`
- `PAIRED_OPEN_OK`
- `PAIR_BETA_DRIFT`
- `PAIR_CORR_COLLAPSE`
- `PAIR_DOLLAR_STOP`
- `PAIR_EVENT`
- `PAIR_SCREEN_EMPTY`
- `PAIR_SCREEN_RANKED`
- `PAIR_SCREEN_TIMEOUT`
- `PAIR_STATS_FAIL`
- `PAIR_STATS_PASS`
- `PAIR_TIME_STOP`
- `PAIR_Z_STOP`
- `PAYLOAD_INCOMPATIBLE`
- `PAYOUT_REJECTED`
- `PAYOUT_SENT`
- `PERP_CLOSED`
- `PERP_NOTHING`
- `PERP_NO_MARKET`
- `PERP_NO_PAIR`
- `PERP_NO_SPREAD`
- `PERP_NO_UPSTREAM_LEG`
- `PERP_OPENED`
- `PERP_REDUCED`
- `POSITION_DESYNC`
- `POSITION_HEALTHY`
- `POSITION_SYNCED`
- `POSITION_TAG_UNOPENED`
- `REBALANCE_DONE`
- `REBALANCE_SKIPPED`
- `REPAY_FAILED`
- `REPAY_NONE`
- `REPAY_PARTIAL`
- `REPAY_SUBMITTED`
- `RISK_PASS`
- `RISK_TIMEOUT`
- `RISK_VETO`
- `SCHEDULE_TICK`
- `SIGNAL_CEILING_SUPPRESSED`
- `SIGNAL_CROSSED`
- `SIGNAL_GATE_FAIL`
- `SPOT_CLOSED`
- `SPOT_NOTHING`
- `SPOT_NO_MARKET`
- `SPOT_NO_PAIR`
- `SPOT_NO_SPREAD`
- `SPOT_NO_UPSTREAM_LEG`
- `SPOT_OPENED`
- `SUPPLY_FAILED`
- `SUPPLY_SUBMITTED`
- `SWAP_FILLED`
- `SWAP_NOTHING`
- `SWAP_PENDING`
- `SWAP_REVERTED`
- `SWAP_SUBMITTED`
- `TWAP_BAND_BREACH`
- `TWAP_FILLED`
- `TWAP_PENDING`
- `TWAP_REVERTED`
- `TWAP_SUBMITTED`
- `TWOFACTOR_APPROVE_SUBMITTED`
- `TWOFACTOR_CARRY_CLOSED`
- `TWOFACTOR_CARRY_CLOSING`
- `TWOFACTOR_CARRY_OPEN`
- `TWOFACTOR_CARRY_OPENED`
- `TWOFACTOR_CBBTC_RELEASED`
- `TWOFACTOR_EXIT`
- `TWOFACTOR_EXIT_CAPPED`
- `TWOFACTOR_HEDGE_MARGIN_SHORT`
- `TWOFACTOR_IDLE_CBBTC`
- `TWOFACTOR_IN_BAND`
- `TWOFACTOR_MINTED`
- `TWOFACTOR_MINT_PENDING`
- `TWOFACTOR_NOT_OPEN`
- `TWOFACTOR_NO_BALANCE`
- `TWOFACTOR_PAUSED`
- `TWOFACTOR_READ`
- `TWOFACTOR_REBALANCED`
- `TWOFACTOR_REDEEMED`
- `TWOFACTOR_RESIDUAL_SHORT`
- `TWOFACTOR_SIM_REVERTED`
- `TWOFACTOR_SPREAD_BELOW_FLOOR`
- `TWOFACTOR_UNAVAILABLE`
- `TWOFACTOR_VAULT_DEPOSIT_SUBMITTED`
- `TWOFACTOR_VAULT_REDEEM_SUBMITTED`
- `TWOFACTOR_VENUE_FUNDING_MISSING`
- `TWOFACTOR_YIELD_ABOVE_EXIT`
- `TWOFACTOR_YIELD_BELOW_FLOOR`
- `UNGUARDED_ACTION`
- `VALUE_CLAMPED`
- `VAULT_DEPOSITED`
- `VAULT_FAILED`
- `VAULT_READ`
- `VAULT_REDEEMED`
- `WEBHOOK_BUY`
- `WEBHOOK_EXIT`
- `WEBHOOK_EXPIRED`
- `WEBHOOK_SELL`
- `WITHDRAWAL_APPROVED`
- `WITHDRAW_COLLATERAL_FAILED`
- `WITHDRAW_COLLATERAL_HEALTH_FAIL`
- `WITHDRAW_COLLATERAL_LIMITED`
- `WITHDRAW_COLLATERAL_NONE`
- `WITHDRAW_COLLATERAL_SUBMITTED`
- `WITHDRAW_NOTHING_DEPLOYED`
- `WITHDRAW_REVERTED`
- `WITHDRAW_SUBMITTED`
- `YIELD_OPT_ALREADY_OPTIMAL`
- `YIELD_OPT_BUILD_FAILED`
- `YIELD_OPT_CHAIN`
- `YIELD_OPT_COOLDOWN`
- `YIELD_OPT_DUST`
- `YIELD_OPT_NO_CAPITAL`
- `YIELD_OPT_NO_VENUE`
- `YIELD_OPT_PAUSED`
- `YIELD_OPT_POLICY_EMPTY`
- `YIELD_OPT_POLICY_SKIP`
- `YIELD_OPT_RANKED`
- `YIELD_OPT_TIMEOUT`

## Node catalog

### Triggers (category: trigger)

- `schedule` -- Schedule: Run on a fixed interval.
  - io: no input, output
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `interval` (select) one of: "Every 5 min", "Every 15 min", "Every hour", "Every 6 hours", "Daily", "Weekly"; default: "Every 6 hours"
  - reasonCodes: SCHEDULE_TICK
- `deposit-received` -- Deposit Received: Fires when funds arrive in the treasury.
  - io: no input, output
  - strategyClasses: treasury-yield
  - config: `minAmount` (number, $, min 0); default: 1000 | `asset` (select) one of: "USDC", "ETH", "Any"; default: "USDC"
  - reasonCodes: DEPOSIT_RECEIVED
- `withdrawal-approved` -- Withdrawal Approved: Fires when a withdrawal requested in the app needs funds this agent has deployed. An owner request fires once it is confirmed. A multi-sign request fires as soon as it is made, and its signatures open once the funds are back; the agent sends nothing else until it finishes. Approval happens in the app, so the path below needs no approval gate of its own.
  - io: no input, output
  - strategyClasses: treasury-yield
  - config: `minAmount` (number, $, min 0); default: 0
  - reasonCodes: WITHDRAWAL_APPROVED
- `signal-threshold` -- Signal Threshold: Fires when a market metric crosses a threshold.
  - io: no input, output
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `metric` (select) one of: "APY spread", "Utilization", "Price move", "Funding rate (annualized)", "Spread z-score", "2Factor vault yield (annualized)", "2Factor vault yield after entry (annualized)", "2Factor perpSr funding (annualized)", "2Factor Jr spread (annualized)", "2Factor Jr spread after entry (annualized)", "2Factor exit capacity (USD)"; default: "APY spread" | `threshold` (number, min 0, max 50000); default: 40 -- bps for spreads; APR % for funding; z units for spread z-score. | `sustained` (number, min 1, max 48); default: 1 -- Consecutive periods the signal must hold. | `confirmation` (select) one of: "None", "Retrace", "Two closes"; default: "None" | `ceiling` (number, min 0, max 50000); default: 500 -- Do not fire when the move exceeds this size. | `instrumentRef` (text); default: "" -- Optional defs key, e.g. btcPerp or pairA.
  - reasonCodes: SIGNAL_CROSSED, SIGNAL_CEILING_SUPPRESSED
- `idle-capital` -- Idle Capital Detected: Fires when treasury funds sit unused.
  - io: no input, output
  - strategyClasses: treasury-yield
  - config: `idleAfter` (select) one of: "12 hours", "24 hours", "3 days"; default: "24 hours" | `minIdle` (number, $, min 0); default: 25000
  - reasonCodes: IDLE_CAPITAL_DETECTED
- `webhook` -- Webhook Signal: Fires when an outside source (a TradingView alert, your own model) posts a signal to this agent's webhook URL. The payload is JSON or a plain message saying buy, sell or exit, with a symbol and an optional size. The action picks the output; the symbol passes on to every block after it set to receive it: a Perp or Spot Position's Symbol, a TWAP's Accumulate, a Swap's From or To. Those trade what the signal names if the treasury policy allows it; a block that names its own instrument acts only on signals for that instrument or for none.
  - io: no input, outputs: buy + sell + exit
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `maxAge` (select) one of: "1 min", "5 min", "15 min", "1 hour"; default: "5 min" -- A signal not acted on within this window is dropped, so a late tick never trades on an old alert. | `maxSizeUsd` (number, $, min 0, max 10000000); default: 0 -- Largest sizeUsd a payload may ask for; a larger one is cut to this. 0 ignores sizeUsd. A sizePct is always honoured, since it can only shrink what the block would do on its own.
  - reasonCodes: WEBHOOK_BUY, WEBHOOK_SELL, WEBHOOK_EXIT, WEBHOOK_EXPIRED

### Capital (category: capital)

- `treasury-vault` -- Treasury Vault: Your main treasury balance.
  - io: input, output
  - strategyClasses: treasury-yield
  - config: `asset` (select) one of: "USDC", "ETH", "Mixed"; default: "USDC"
  - reasonCodes: VAULT_READ
- `liquidity-buffer` -- Liquidity Buffer: Keep a target balance liquid for payments and withdrawals.
  - io: input, outputs: breach + surplus
  - strategyClasses: treasury-yield
  - config: `target` (number, $, min 0); default: 50000 | `asset` (select) one of: "USDC", "ETH"; default: "USDC"
  - reasonCodes: BUFFER_TOPPED_UP, BUFFER_BREACH
- `payout` -- Payout: Hand the freed funds back to the withdrawal request. The app sends it to the destination on the request, through the same approval and signing as any withdrawal; the agent never transfers out of the treasury.
  - io: input, no output
  - strategyClasses: treasury-yield
  - config: none
  - reasonCodes: PAYOUT_SENT, PAYOUT_REJECTED
- `position-sync` -- Position Sync: Read venue truth for a positionTag before health checks or exits (fills, ADL, liquidations).
  - io: input, output
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `positionTag` (text); default: "carry-btc" | `venue` (select) one of: "Hyperliquid"; default: "Hyperliquid"
  - reasonCodes: POSITION_SYNCED, POSITION_DESYNC

### Intelligence (category: intelligence)

- `signal-gate` -- Signal Gate: Mid-flow signal check (funding APR, z-score, spreads). Unlike Schedule triggers, this sits on a data path.
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `metric` (select) one of: "APY spread", "Utilization", "Price move", "Funding rate (annualized)", "Spread z-score", "2Factor vault yield (annualized)", "2Factor vault yield after entry (annualized)", "2Factor perpSr funding (annualized)", "2Factor Jr spread (annualized)", "2Factor Jr spread after entry (annualized)", "2Factor exit capacity (USD)"; default: "Funding rate (annualized)" | `threshold` (number, min 0, max 50000); default: 10 | `sustained` (number, min 1, max 48); default: 8 | `confirmation` (select) one of: "None", "Retrace", "Two closes"; default: "None" | `ceiling` (number, min 0, max 50000); default: 80 | `instrumentRef` (text); default: ""
  - reasonCodes: SIGNAL_CROSSED, SIGNAL_CEILING_SUPPRESSED, SIGNAL_GATE_FAIL
- `yield-optimizer` -- Yield Optimizer: Finds the best yield markets given your global policies, strategy guards, and venue health metrics.
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield
  - config: `provider` (select) one of: "Ampli", "ZyFAI"; default: "Ampli" -- Which engine ranks the venues. ZyFAI calls the external allocator; Ampli ranks in-house. | `objective` (select) one of: "Max yield", "Risk-adjusted", "Stable only"; default: "Risk-adjusted" | `respectPolicies` (toggle); default: true -- Only rank venues/markets already allowed by treasury policies. | `healthFloor` (select) one of: "Strict", "Standard", "Permissive"; default: "Standard"
  - reasonCodes: YIELD_OPT_RANKED, YIELD_OPT_NO_VENUE, YIELD_OPT_POLICY_EMPTY, YIELD_OPT_TIMEOUT, YIELD_OPT_DUST, YIELD_OPT_COOLDOWN, YIELD_OPT_PAUSED, YIELD_OPT_NO_CAPITAL, YIELD_OPT_CHAIN, YIELD_OPT_ALREADY_OPTIMAL, YIELD_OPT_POLICY_SKIP, YIELD_OPT_BUILD_FAILED
- `price-feed` -- Price Feed: Oracle price data; falls back when the feed is stale or deviates.
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `pair` (select) one of: "ETH / USD", "BTC / USD", "wstETH / ETH"; default: "ETH / USD" | `source` (select) one of: "Alchemy", "DefiLlama"; default: "Alchemy" | `maxDeviation` (number, %, min 0, max 50); default: 0.5 | `staleAfter` (number, s, min 1, max 3600); default: 120
  - reasonCodes: FEED_HEALTHY, FEED_STALE, FEED_DEVIATION, FEED_SOURCE_REJECTED
- `market-conditions` -- Market Conditions: Volatility, liquidity and funding checks before capital moves.
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `metric` (select) one of: "Volatility", "Liquidity depth", "Funding rate (per venue)", "Funding rate (net across legs)", "Realized vol vs 1y median"; default: "Volatility" | `window` (select) one of: "24 hours", "7 days", "30 days"; default: "7 days" | `multiple` (number, min 0.1, max 20); default: 2 -- Used when metric is Realized vol vs 1y median.
  - reasonCodes: MARKET_OK, MARKET_OUT_OF_RANGE
- `risk-check` -- Risk Check: Cheap deterministic pre-gate. Expensive judgment belongs on an agent pack (P1).
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `model` (select) one of: "Ensemble", "Single model"; default: "Ensemble" | `maxRisk` (select) one of: "Low", "Medium", "High"; default: "Medium" | `timeout` (number, s, min 1, max 300); default: 30
  - reasonCodes: RISK_PASS, RISK_VETO, RISK_TIMEOUT
- `loop-orders` -- Loop Orders: Before a Loop Check on a market that folds or exits through a swap: while a dTWAP order buying the collateral is out, take Fold order open; while one selling it is out, take Exit order open, so the Swap that placed it can settle it. With no order out, pass on to the Loop Check.
  - io: input, outputs: passed + settle-fold + settle-exit + error
  - strategyClasses: treasury-yield, hedged-carry
  - config: `market` (select) one of: "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "yoUSD / USDC on Morpho (Base)"
  - reasonCodes: LOOP_NO_ORDER, LOOP_ORDER_OPEN, LOOP_UNREADABLE
- `loop-check` -- Loop Check: Read a stablecoin loop and say what it needs. Open when there is no position, Fold when the LTV is under the target, Unwind when it is over, the spread has turned or the health factor is low. Unwind hands the collateral to withdraw and a floor for selling it. While a dTWAP order from a fold or an exit is still out it holds; put a Loop Orders block in front to settle that order.
  - io: input, outputs: open + fold + unwind + none + error
  - strategyClasses: treasury-yield, hedged-carry
  - config: `market` (select) one of: "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "yoUSD / USDC on Morpho (Base)" | `targetLtv` (number, %, min 0, max 93); default: 80 -- Debt over collateral value. 80% is 5x leverage. 0 unwinds the whole loop back to the borrow token. | `band` (number, pts, min 0.5, max 20); default: 3 -- Fold once the LTV is this far under the target; unwind once it is this far over. | `minSpread` (number, %, min -5, max 20); default: 1 -- Collateral yield (its share price over the last week) less the borrow rate. Under it the loop does not open or fold; under zero it unwinds. | `minHealthFactor` (number, min 1.01, max 3); default: 1.08 -- Under it the loop unwinds. An unwind through a dTWAP order also keeps the health factor above this while the order is out. | `maxSlippage` (number, bps, min 1, max 300); default: 30 -- Floor on selling withdrawn collateral, from the lending market's own oracle price.
  - reasonCodes: LOOP_OPEN, LOOP_FOLD, LOOP_AT_TARGET, LOOP_SPREAD_LOW, LOOP_OVER_TARGET, LOOP_SPREAD_NEGATIVE, LOOP_HEALTH_LOW, LOOP_EXIT, LOOP_STUCK, LOOP_ORDER_OPEN, LOOP_UNREADABLE
- `lp-monitor` -- LP Monitor: Read the treasury's Uniswap v3 position in a pool: where the price sits in its range, what it holds and the fees it has earned. Takes Re-range once the price has stayed near an edge long enough; wire it to Exit LP and then LP Stake to re-centre.
  - io: input, outputs: manage + rerange + none + error
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `pool` (select) one of: "ETH / USDC", "USDC / USDT", "wstETH / USDC"; default: "ETH / USDC" | `edgeBuffer` (number, %, min 0, max 45); default: 20 -- Re-centre once the price is within this share of the range's width from an edge. 0 waits until it leaves the range. | `confirmMinutes` (number, min, min 0, max 1440); default: 30 -- How long the price must stay past the buffer before re-centring, so a brief spike does not churn the position.
  - reasonCodes: LP_IN_RANGE, LP_NEAR_EDGE, LP_RERANGE_DUE, LP_NO_POSITION, LP_READ_FAILED
- `lp-rules` -- LP Exit Rules: When to pull out of a Uniswap v3 position: a loss against simply holding what went in, a weak Aave health factor on its hedge, too long out of range, or fees that no longer cover the borrow. Any rule takes Exit.
  - io: input, outputs: primary + fallback
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `pool` (select) one of: "ETH / USDC", "USDC / USDT", "wstETH / USDC"; default: "ETH / USDC" | `maxLoss` (number, %, min 0.5, max 50); default: 3 -- Exit once the position, fees included, is worth this much less than holding the tokens that went in. This is impermanent loss net of fees. | `outOfRangeHours` (number, h, min 0, max 168); default: 12 -- Exit once the price has stayed outside the range this long, earning nothing. 0 turns the rule off. | `minHealthFactor` (number, min 1.05, max 3); default: 1.3 -- Exit once the Aave health factor of the hedge falls under this. | `minCarry` (number, %, min -50, max 100); default: 0 -- Fees a year plus what the collateral earns, less the borrow cost, over the position's value. Exit below it. | `carryAfterHours` (number, h, min 0, max 720); default: 24 -- Judge the carry only once the position is this old, so a quiet first day does not close it.
  - reasonCodes: LP_RULES_HOLD, LP_RULES_NO_POSITION, LP_RULES_LOSS, LP_RULES_HEALTH, LP_RULES_OUT_OF_RANGE, LP_RULES_CARRY, LP_RULES_UNREADABLE
- `hedge-check` -- Hedge Check: Compare the ETH owed on Aave with the ETH a Uniswap position holds. As ETH falls the pool buys ETH, so it takes Borrow and asks for the ETH to borrow; as ETH rises the pool sells ETH, so it takes Repay and asks for the USDC collateral to withdraw, with the least ETH it must buy back.
  - io: input, outputs: borrow + repay + none + error
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `pool` (select) one of: "ETH / USDC", "USDC / USDT", "wstETH / USDC"; default: "ETH / USDC" | `hedgeRatio` (number, %, min 0, max 150); default: 100 -- Share of the position's ETH to owe. 100% is delta neutral: the ETH price alone neither gains nor loses the position anything. | `drift` (number, %, min 0.5, max 50); default: 5 -- Rebalance once the ETH owed is off target by more than this share of the position's value. | `minHealthFactor` (number, min 1.05, max 3); default: 1.5 -- Asks to borrow more only while the health factor, after the USDC is supplied, stays above this. | `maxSlippage` (number, bps, min 1, max 300); default: 30 -- Sizes the withdrawal so the swap still buys back enough ETH, after the pool fee.
  - reasonCodes: HEDGE_IN_BAND, HEDGE_BORROW, HEDGE_REPAY, HEDGE_LIMITED, HEDGE_NO_POSITION, HEDGE_CHECK_UNREADABLE
- `debt-check` -- Debt Check: On the way out, compare the ETH on hand (handed over by an Exit LP, plus the treasury's) with the ETH owed on Aave. Covered when it is enough; Short asks for the USDC to spend buying the rest, with the least ETH it must buy. Hands every token on.
  - io: input, outputs: covered + short + error
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `asset` (select) one of: "ETH"; default: "ETH" | `maxSlippage` (number, bps, min 1, max 500); default: 50 -- Sizes the USDC to spend so the swap still buys enough ETH, after the pool fee.
  - reasonCodes: DEBT_NONE, DEBT_COVERED, DEBT_SHORT, DEBT_SHORT_PARTIAL, DEBT_CHECK_UNREADABLE
- `funding-feed` -- Funding Feed: Provider: current and trailing funding for a venue/symbol, annualized from the venue's settlement cadence.
  - io: input, outputs: data + error
  - strategyClasses: hedged-carry
  - config: `venue` (select) one of: "Hyperliquid"; default: "Hyperliquid" | `symbol` (text); default: "BTC" -- Any listed Hyperliquid perp ticker, not just the majors. Reads are ungated; the policy universe gates the action that acts on the signal. | `lookback` (select) one of: "24 hours", "7 days", "30 days"; default: "7 days"
  - reasonCodes: FUNDING_READ, FUNDING_UNAVAILABLE
- `twofactor-feed` -- 2Factor Feed: Provider: 2Factor tranche funding on Base. Senior reads the USD vault yield; Junior reads the perpJr carry spread against venue funding. Also publishes exit capacity, subDR and your position.
  - io: input, outputs: data + error
  - strategyClasses: treasury-yield, hedged-carry
  - config: `view` (select) one of: "Senior", "Junior"; default: "Senior" -- Senior: USD vault (USDC + perpSr). Junior: perpJr hedged with a venue short; needs a funding-feed upstream. | `entrySize` (number, %, min 0, max 100); default: 25 -- Share of idle USDC the after-entry metrics assume. Your entry moves the funding rate; the 2Factor actions size themselves to their floor, so these metrics show the cost of the full size. | `maxExitFee` (number, %, min 0, max 10); default: 0.5 -- Exit capacity is the largest exit whose fee stays within this. | `hedgeSymbol` (select) one of: "BTC"; default: "BTC"
  - reasonCodes: TWOFACTOR_READ, TWOFACTOR_UNAVAILABLE, TWOFACTOR_PAUSED, TWOFACTOR_VENUE_FUNDING_MISSING
- `basis-check` -- Basis Check: Spot vs perp basis against a historical range.
  - io: input, outputs: primary + fallback
  - strategyClasses: hedged-carry
  - config: `symbol` (text); default: "BTC" -- Any listed Hyperliquid perp ticker, not just the majors. Reads are ungated; the policy universe gates the action that acts on the signal. | `maxDeviation` (number, bps, min 1, max 5000); default: 80
  - reasonCodes: BASIS_OK, BASIS_DIVERGED
- `pair-screener` -- Pair Screener: Ranks correlated Hyperliquid perp pairs by how stretched their spread is today and binds the top pick into an instrument-pair def. Pair Stats still has to pass the chosen pair.
  - io: input, outputs: primary + fallback
  - strategyClasses: market-neutral
  - config: `venue` (select) one of: "Hyperliquid"; default: "Hyperliquid" | `minCorrelation` (number, min 0, max 1); default: 0.7 -- Pearson ρ floor. Pairs below this are dropped. | `maxHalfLifeDays` (number, d, min 1, max 90); default: 30 -- Reject relationships that mean-revert too slowly to sit through. | `minHistoryDays` (number, d, min 30, max 365); default: 90 | `maxZ` (number, σ, min 1, max 10); default: 3.5 -- The most stretched spread within this many standard deviations is picked first. Match the entry signal's ceiling; pairs past it rank last. | `topK` (number, min 1, max 20); default: 5 | `bindTo` (text); default: "pairA" -- defs key the rest of the graph reads, e.g. "pairA"
  - reasonCodes: PAIR_SCREEN_RANKED, PAIR_SCREEN_EMPTY, PAIR_SCREEN_TIMEOUT
- `pair-stats` -- Pair Stats: Deterministic screen for an instrument-pair def (β, half-life, σ). Enforces minimum history.
  - io: input, outputs: primary + fallback
  - strategyClasses: market-neutral
  - config: `pairRef` (text); default: "pairA" -- defs key, e.g. "pairA" | `minHistoryDays` (number, d, min 30, max 365); default: 90
  - reasonCodes: PAIR_STATS_PASS, PAIR_STATS_FAIL, INSUFFICIENT_HISTORY
- `position-health` -- Position Health: Tripwire battery for an open positionTag. Entry statistics are frozen at fill.
  - io: input, outputs: primary + fallback
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `positionTag` (text); default: "carry-btc" | `profile` (select) one of: "Carry", "Pair"; default: "Carry"
  - reasonCodes: POSITION_HEALTHY, CARRY_FUNDING_FLIPPED, CARRY_BELOW_COST, CARRY_MARGIN_STRESS, CARRY_BASIS_DIVERGED, CARRY_LEG_MISMATCH, CARRY_VENUE_INCIDENT, PAIR_CORR_COLLAPSE, PAIR_BETA_DRIFT, PAIR_Z_STOP, PAIR_TIME_STOP, PAIR_DOLLAR_STOP, PAIR_EVENT
- `leg-check` -- Leg Check: Finds a two-leg Hyperliquid position left with one leg, because the second leg of an open failed or the first leg of a close went through alone. Past the one-leg window it names the leg for a Perp or Spot Position sized from Leg Check to flatten.
  - io: input, outputs: passed + flatten + flatten-spot + waiting
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `positionTag` (text); default: "carry-btc" | `maxOneLegOpen` (number, s, min 1, max 3600); default: 5
  - reasonCodes: LEGS_MATCHED, LEGS_FLAT, LEG_OUT_WAITING, LEG_OUT_INCIDENT
- `twofactor-check` -- 2Factor Carry Check: Reads where the perpJr carry is: Open while it is held and the Jr spread clears the floor, on to a 2Factor Hedge Check; Exit once the spread falls below the floor; Clean up once the perpJr is gone or unminted cbBTC is due to be handed on, on to a 2Factor Cleanup.
  - io: input, outputs: passed + exit + cleanup + waiting
  - strategyClasses: hedged-carry
  - config: `positionTag` (text); default: "2f-jr-btc" | `exitBelow` (number, %, min -100, max 200); default: 3 -- Takes Exit once the Jr spread falls below this. | `allocation` (number, %, min 0, max 100); default: 100 -- Share of treasury cbBTC that is the carry's, for cbBTC it never recorded. The Cleanup after it uses the same share. | `releaseIdleAfter` (number, min, min 0, max 1440); default: 0 -- Takes Clean up for cbBTC the carry holds but never minted, once it has sat this long, so the Cleanup hands it on. 0 keeps it; use 0 when the treasury holds cbBTC of its own.
  - reasonCodes: TWOFACTOR_CARRY_OPEN, TWOFACTOR_EXIT, TWOFACTOR_RESIDUAL_SHORT, TWOFACTOR_CBBTC_RELEASED, TWOFACTOR_CARRY_CLOSED, TWOFACTOR_IDLE_CBBTC, TWOFACTOR_MINT_PENDING, TWOFACTOR_NOT_OPEN, TWOFACTOR_UNAVAILABLE
- `twofactor-hedge-check` -- 2Factor Hedge Check: After a 2Factor Carry Check's Open: keeps the Hyperliquid short matched to the perpJr's BTC exposure. In band passes; Add to short or Reduce short hands the size to a Perp Position once net BTC drifts past the band.
  - io: input, outputs: passed + resize + reduce + waiting
  - strategyClasses: hedged-carry
  - config: `positionTag` (text); default: "2f-jr-btc" | `band` (number, %, min 0.5, max 20); default: 2 -- Takes Add or Reduce once net BTC drifts past this share of the short. | `hedgeLeverage` (number, x, min 1, max 10); default: 2 -- An add waits until Hyperliquid margin covers its notional over this.
  - reasonCodes: TWOFACTOR_IN_BAND, TWOFACTOR_REBALANCED, TWOFACTOR_HEDGE_MARGIN_SHORT, TWOFACTOR_NOT_OPEN, TWOFACTOR_UNAVAILABLE
- `twofactor-cleanup` -- 2Factor Cleanup: After a 2Factor Carry Check's Clean up: closes a Hyperliquid short left once the perpJr is gone, hands on cbBTC the carry no longer needs, or says the carry is closed. Uses the allocation and idle release of the Carry Check before it.
  - io: input, outputs: flatten + release + done + waiting
  - strategyClasses: hedged-carry
  - config: `positionTag` (text); default: "2f-jr-btc"
  - reasonCodes: TWOFACTOR_RESIDUAL_SHORT, TWOFACTOR_CBBTC_RELEASED, TWOFACTOR_CARRY_CLOSED, TWOFACTOR_NOT_OPEN, TWOFACTOR_UNAVAILABLE

### Actions (category: action)

- `lend` -- Lend / Supply: Supply capital to a lending venue.
  - io: input, output
  - strategyClasses: treasury-yield
  - config: `protocol` (select) one of: "Best available", "Aave v3", "Morpho Blue", "Compound v3"; default: "Best available" | `allocation` (number, %, min 0, max 100); default: 80
  - reasonCodes: LEND_SUBMITTED, LEND_REVERTED
- `withdraw` -- Withdraw: Pull every yield position back into the treasury, wherever the optimizer placed it. The next scheduled run redeploys whatever the open withdrawals do not need.
  - io: input, output
  - strategyClasses: treasury-yield
  - config: none
  - reasonCodes: WITHDRAW_SUBMITTED, WITHDRAW_NOTHING_DEPLOYED, WITHDRAW_REVERTED
- `swap` -- Swap: Exchange one asset for another. On Orbs dTWAP the order is placed on chain and the path continues on Filled once it has filled; while it is open the block takes Pending and places no second order. On Uniswap v3 the swap lands in this run's batch and takes Filled at once, so the next block spends its output in the same transaction.
  - io: input, outputs: filled + pending + failed + none
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `from` (select) one of: "USDC", "ETH", "wstETH", "cbBTC", "USDT", "USDe", "sUSDe", "USDS", "sUSDS", "Webhook signal"; default: "USDC"; receives the webhook signal's symbol when `from` is "Webhook signal" -- Webhook signal sells the token the signal names, by ticker or Base address, such as an exit back to USDC. The treasury policy must approve the token's address. | `to` (select) one of: "USDC", "ETH", "wstETH", "cbBTC", "USDT", "USDe", "sUSDe", "USDS", "sUSDS", "Webhook signal"; default: "ETH"; receives the webhook signal's symbol when `to` is "Webhook signal" -- Webhook signal buys the token the signal names. The treasury policy must approve its address, so whatever is bought can be sold again. | `venue` (select) one of: "Orbs dTWAP", "Uniswap v3", "Orbs Spot", "Best available", "CowSwap"; default: "Orbs dTWAP" -- Orbs dTWAP places the order on chain from the treasury, needs at least $10 on Base or $200 on Ethereum, and fills in a few minutes. Uniswap v3 swaps in the same batch as the blocks around it, for hedges that must land together; it uses the cheapest pool deep enough for the trade. Orbs Spot needs a signature the treasury cannot give yet; Best available and CowSwap route through it. | `sizeFrom` (select) one of: "Share of balance", "Upstream output"; default: "Share of balance" -- Share of balance swaps a share of the From balance, at its live price. Upstream output swaps exactly what the block before hands over, such as the ETH a Borrow takes out or the cbBTC a 2Factor Redeem returns. | `allocation` (number, %, min 0, max 100); default: 100 -- Share of the From balance to swap, when sized from the balance. | `maxSlippage` (number, bps, min 1, max 500); default: 30
  - reasonCodes: SWAP_SUBMITTED, SWAP_PENDING, SWAP_FILLED, SWAP_REVERTED, SWAP_NOTHING
- `twap` -- TWAP Order: Buy an asset with USDC in slices over time through one Orbs dTWAP order, to reduce market impact. The path continues on Filled once every slice has filled; while the order is open the block takes Pending and places no second order. Any Base ERC-20 can be bought by pasting its address under Custom token, or named by a webhook signal.
  - io: input, outputs: filled + pending + failed
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `asset` (select) one of: "ETH", "wstETH", "cbETH", "weETH", "cbBTC", "AERO", "VIRTUAL", "MORPHO", "EURC", "Custom token", "Webhook signal"; default: "ETH"; receives the webhook signal's symbol when `asset` is "Webhook signal" -- Bought with the treasury's USDC. Custom token buys any ERC-20 on Base that Alchemy prices. Webhook signal buys the token the signal names, by ticker or Base address, once the treasury policy approves its address. | `tokenAddress` (text); default: "" -- The ERC-20 contract on Base. Decimals are read on chain and the slice floor uses Alchemy's price. Slices fill only while dTWAP takers find liquidity for it inside the limit band. | `allocation` (number, %, min 1, max 100); default: 100 -- Share of the USDC balance each accumulation spends. | `duration` (select) one of: "1 hour", "4 hours", "24 hours"; default: "4 hours" | `slices` (number, min 2, max 100); default: 12 -- Each slice must be worth at least $10; a smaller budget fills in fewer, larger slices. | `limitBand` (number, bps, min 1, max 1000); default: 50 -- Max distance from the price when the order is placed. A slice that cannot fill inside it waits; the order expires 30 minutes after the duration.
  - reasonCodes: TWAP_SUBMITTED, TWAP_PENDING, TWAP_FILLED, TWAP_BAND_BREACH, TWAP_REVERTED
- `loop` -- Loop: Recursive leverage on a stablecoin yield spread: supply a yield-bearing stablecoin, borrow a cheaper stablecoin against it, turn the borrow into more collateral and supply it again, up to a target LTV. A vault deposit folds several times in one run; a swap folds once through a one-chunk Orbs dTWAP order and takes Pending until it fills.
  - io: input, outputs: done + pending + unwind + failed
  - strategyClasses: treasury-yield, hedged-carry
  - config: `market` (select) one of: "Best on this chain", "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "Best on this chain" -- Collateral / borrow pair. Best on this chain picks the widest net yield at the target among markets on the treasury's chain, and sticks with a position once it is open. | `allocation` (number, %, min 1, max 100); default: 50 -- Share of idle USDC that opens the position. Later runs only fold what is already in it. | `targetLtv` (number, %, min 10, max 93); default: 80 -- Debt over collateral value. 80% is 5x leverage. Capped below each market's borrow limit and so the health factor stays above its floor. | `minSpread` (number, %, min 0, max 20); default: 1 -- Collateral yield (its share price over the last week) less the borrow rate. Under it the loop does not open or fold further; a negative spread unwinds. | `minHealthFactor` (number, min 1.01, max 3); default: 1.08 -- Under it the block takes Unwind. | `maxSlippage` (number, bps, min 1, max 300); default: 30 -- Floor on every dTWAP order, from the lending market's own oracle price. | `iterations` (number, min 1, max 10); default: 4 -- For markets entered by vault deposit. Swap entries fold once per run.
  - reasonCodes: LOOP_STEP, LOOP_PENDING, LOOP_AT_TARGET, LOOP_SPREAD_LOW, LOOP_NO_CAPITAL, LOOP_HEALTH_FAIL, LOOP_SPREAD_NEGATIVE, LOOP_SWAP_STALLED, LOOP_UNWINDING, LOOP_FAILED
- `supply-collateral` -- Supply Collateral: Supply one token as collateral to a lending market: USDC to Aave for the hedge market, or a loop market's yield-bearing stablecoin to Aave or Morpho. Enters the market's e-mode first when the account is not in it.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `market` (select) one of: "ETH against USDC on Aave", "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "ETH against USDC on Aave" -- ETH against USDC on Aave is the hedge market on the treasury's chain. The loop markets name their collateral and borrow. | `sizeFrom` (select) one of: "Upstream output", "Share of balance"; default: "Upstream output" -- Upstream output supplies what the block before hands over, such as a Vault deposit's shares or a Swap's USDC. Share of balance supplies a share of what the treasury holds. | `allocation` (number, %, min 1, max 100); default: 100 -- Share of the balance to supply, when sized from the balance. For USDC this is idle USDC.
  - reasonCodes: SUPPLY_SUBMITTED, SUPPLY_FAILED
- `borrow` -- Borrow: Borrow a market's loan token against the collateral already supplied, including collateral a Supply Collateral just before this puts up in the same batch. Hands the borrowed token to the next block.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `market` (select) one of: "ETH against USDC on Aave", "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "ETH against USDC on Aave" | `sizeFrom` (select) one of: "Target LTV", "Upstream output"; default: "Target LTV" -- Target LTV borrows up to the target against the collateral. Upstream output borrows exactly what a check before this asks for, such as the ETH a Hedge Check finds the hedge short of. | `targetLtv` (number, %, min 5, max 93); default: 50 -- Debt over collateral value. Capped below the market's borrow limit and so the health factor stays above its floor. Aave lends up to 75% against USDC. | `minHealthFactor` (number, min 1.02, max 3); default: 1.5 -- A borrow that would leave the health factor under this is cut back to it.
  - reasonCodes: BORROW_SUBMITTED, BORROW_LIMITED, BORROW_NONE, BORROW_HEALTH_FAIL, BORROW_FAILED
- `withdraw-collateral` -- Withdraw Collateral: Take collateral back out of a lending market, never below the health floor against the debt that is left after any Repay just before this.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `market` (select) one of: "ETH against USDC on Aave", "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "ETH against USDC on Aave" | `sizeFrom` (select) one of: "Upstream output", "All of this strategy's"; default: "Upstream output" -- Upstream output withdraws what a check before this asks for. All of this strategy's withdraws only the collateral this deployment supplied. | `minHealthFactor` (number, min 1.02, max 3); default: 1.1 -- A withdrawal that would leave the health factor under this is cut back to it.
  - reasonCodes: WITHDRAW_COLLATERAL_SUBMITTED, WITHDRAW_COLLATERAL_LIMITED, WITHDRAW_COLLATERAL_NONE, WITHDRAW_COLLATERAL_HEALTH_FAIL, WITHDRAW_COLLATERAL_FAILED
- `repay` -- Repay: Repay a lending market's debt with the loan token: what the block before hands over, or that plus what the treasury holds. Clears the whole debt when there is enough, and hands on what is left.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `market` (select) one of: "ETH against USDC on Aave", "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "ETH against USDC on Aave" | `sizeFrom` (select) one of: "Upstream output", "All owed"; default: "Upstream output" -- Upstream output repays with what the block before hands over. All owed also uses the loan token the treasury already holds.
  - reasonCodes: REPAY_SUBMITTED, REPAY_PARTIAL, REPAY_NONE, REPAY_FAILED
- `vault` -- Vault: Deposit into or redeem from a loop market's ERC-4626 vault: USDC into yoUSD, USDe into sUSDe. Hands on the shares or the assets, less a small margin for the vault price moving before the batch lands.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `market` (select) one of: "yoUSD / USDC on Morpho (Base)", "sUSDe / USDC on Aave e-mode (Ethereum)", "sUSDe / USDe on Aave e-mode (Ethereum)", "sUSDS / USDT on Morpho (Ethereum)"; default: "yoUSD / USDC on Morpho (Base)" -- The vault is the market's collateral. | `action` (select) one of: "Deposit", "Redeem"; default: "Deposit" -- Redeem works only where the vault pays out at once; sUSDe has a cooldown, so sell it with a Swap. | `sizeFrom` (select) one of: "Upstream output", "Share of balance"; default: "Upstream output" | `allocation` (number, %, min 1, max 100); default: 100 -- Share of the balance, when sized from the balance. For USDC this is idle USDC.
  - reasonCodes: VAULT_DEPOSITED, VAULT_REDEEMED, VAULT_FAILED
- `lp-stake` -- LP Stake: Open a Uniswap v3 position from USDC: one batch swaps the share the range needs into the other token, then mints.
  - io: input, outputs: default + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `pool` (select) one of: "ETH / USDC", "USDC / USDT", "wstETH / USDC"; default: "ETH / USDC" -- On Base, wstETH / USDC only has a thin 0.05% pool, so only Concentrated can mint there. | `range` (select) one of: "Wide", "Balanced", "Concentrated"; default: "Balanced" -- Wide uses the 1% fee pool, Balanced 0.3%, Concentrated 0.05%. | `width` (number, %, min 0, max 50); default: 0 -- Either side of the price. 0 uses the range's preset: about 50%, 10% or 2%. | `allocation` (number, %, min 1, max 100); default: 50 -- Share of idle USDC the position takes, swap included. A token handed over alone, such as a Borrow's ETH or a Swap's output, goes in with as much idle USDC as it is worth, up to this share. USDC handed over (an Exit LP before a re-range) goes in as it is. | `maxSlippage` (number, bps, min 1, max 500); default: 50 -- Bounds the swap and the mint against the live price, and refuses a pool priced further than this from the market.
  - reasonCodes: LP_OPENED, LP_REVERTED
- `lp-exit` -- Exit LP: Close every Uniswap v3 position the treasury holds in a pool: remove the liquidity, collect fees and burn the NFT. Hands what comes back to the next block.
  - io: input, outputs: done + failed
  - strategyClasses: treasury-yield, market-neutral, hedged-carry
  - config: `pool` (select) one of: "ETH / USDC", "USDC / USDT", "wstETH / USDC"; default: "ETH / USDC" | `maxSlippage` (number, bps, min 1, max 500); default: 50 -- Bounds the withdrawal against the pool price moving inside the block.
  - reasonCodes: LP_EXITED, LP_EXIT_NONE, LP_EXIT_FAILED
- `rebalance` -- Rebalance: Bring the portfolio back to target weights. Each asset off target by more than the drift is one Orbs dTWAP market order to or from USDC.
  - io: input, output
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `mode` (select) one of: "Proportional", "Threshold"; default: "Proportional" | `drift` (number, %, min 0, max 50); default: 5 | `scope` (select) one of: "Portfolio", "Strategy instance", "Position tag"; default: "Portfolio" | `weights` (text); default: undefined -- Percent by asset, for example USDC:70, ETH:30
  - reasonCodes: REBALANCE_DONE, REBALANCE_SKIPPED
- `perp-position` -- Perp Position: Open or close one perpetual leg on Hyperliquid, including HIP-3 markets such as tokenized stocks (xyz:TSLA). In a two-leg structure each leg is its own block: the second sizes to match the first, and both carry the same position tag so Leg Check can flatten a leg left alone.
  - io: input, outputs: default + none
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `venue` (select) one of: "Hyperliquid"; default: "Hyperliquid" | `symbol` (text); default: "BTC"; receives the webhook signal's symbol when `symbolFrom` is "Webhook signal" -- Any listed Hyperliquid perp ticker, or a HIP-3 market as dex:TICKER, such as xyz:TSLA for Tesla on the xyz dex. Checked against your policy's Hyperliquid universe at commit, so a name that misses the quality floors is refused. | `symbolFrom` (select) one of: "Symbol", "Webhook signal"; default: "Symbol" -- Webhook signal trades the ticker the signal names, if your policy's Hyperliquid universe admits it: ETH, or a HIP-3 market such as xyz:TSLA. A bare ticker listed only on a HIP-3 dex, such as TSLA from a NASDAQ:TSLA alert, trades its most liquid HIP-3 listing. Each ticker keeps its own position under the tag, as tag:TICKER, so a signal for SOL never closes the ETH leg. | `pairLeg` (select) one of: "None", "Leg A", "Leg B"; default: "None" -- Trade a leg of the pair the screener bound instead of the symbol. A close reads the leg frozen on the position at entry. | `side` (select) one of: "Long", "Short", "Against the spread"; default: "Short" -- Against the spread sells the rich leg and buys the cheap one, from the sign of the spread z-score. A close always reverses the leg it closes. | `sizeFrom` (select) one of: "Account", "Account, one of two legs", "Match upstream leg", "Leg Check"; default: "Account" -- Account: the collateral at Hyperliquid at the leverage cap. One of two legs: half of what the account can carry across two perps. Match upstream leg: the notional of the leg before it. Leg Check: flatten the perp leg Leg Check found alone. | `leverageCap` (number, x, min 1, max 20); default: 2 | `marginBuffer` (number, %, min 10, max 200); default: 50 -- Required distance above maintenance margin. | `reduceOnly` (toggle); default: false
  - reasonCodes: PERP_OPENED, PERP_CLOSED, PERP_REDUCED, PERP_NOTHING, PERP_NO_PAIR, PERP_NO_SPREAD, PERP_NO_UPSTREAM_LEG, PERP_NO_MARKET
- `spot-position` -- Spot Position: Buy or sell one spot leg on Hyperliquid, such as the long leg of a carry. Its perp hedge is a separate Perp Position after it that matches its size.
  - io: input, outputs: default + none
  - strategyClasses: hedged-carry
  - config: `venue` (select) one of: "Hyperliquid", "Best available"; default: "Hyperliquid" | `symbol` (text); default: "BTC"; receives the webhook signal's symbol when `symbolFrom` is "Webhook signal" -- Ticker for the spot leg. Checked against your policy's Hyperliquid universe at commit, same as the perp leg. HIP-3 markets are perps only. | `symbolFrom` (select) one of: "Symbol", "Webhook signal"; default: "Symbol" -- Webhook signal trades the ticker the signal names, if your policy's Hyperliquid universe admits it, under its own tag:TICKER position. | `side` (select) one of: "Buy", "Sell"; default: "Buy" -- Sell with a position tag closes that position's spot leg. | `sizeFrom` (select) one of: "Account", "Account, one of two legs", "Match upstream leg", "Leg Check"; default: "Account" -- One of two legs: what the Hyperliquid account can carry with this spot leg and a perp hedge. Leg Check: flatten the spot leg Leg Check found alone.
  - reasonCodes: SPOT_OPENED, SPOT_CLOSED, SPOT_NOTHING, SPOT_NO_PAIR, SPOT_NO_SPREAD, SPOT_NO_UPSTREAM_LEG, SPOT_NO_MARKET
- `paired-execution` -- Paired Execution: Atomic two-leg open/close. If one leg fills and the other cannot within maxOneLegOpen, flatten and raise an incident.
  - io: input, output
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `mode` (select) one of: "OPEN", "CLOSE"; default: "OPEN" | `positionTag` (text); default: "carry-btc" | `maxOneLegOpen` (number, s, min 1, max 60); default: 5 | `structure` (select) one of: "Spot long / perp short", "Perp long / perp short"; default: "Spot long / perp short"
  - reasonCodes: PAIRED_OPEN_OK, PAIRED_CLOSE_OK, LEG_OUT_INCIDENT
- `twofactor-vault` -- 2Factor USD Vault: Deposit idle USDC into the 2Factor USD vault (USDC + perpSr, near USD-stable) or redeem notes back to USDC. Simulated first; the min-out comes from the simulation.
  - io: input, output
  - strategyClasses: treasury-yield
  - config: `mode` (select) one of: "Deposit", "Redeem"; default: "Deposit" | `allocation` (number, %, min 0, max 100); default: 50 -- Deposit: share of idle USDC. Redeem: share of vault notes, unless an approved withdrawal sets the amount. | `enterAbove` (number, %, min 0, max 100); default: 5 -- Deposit only while the vault yield after your entry is at least this; a deposit that would dilute it further is sized down. | `exitBelow` (number, %, min 0, max 100); default: 3 -- Redeem only once the vault yield falls below this. 0 redeems regardless, for withdrawal paths. | `slippageBps` (number, bps, min 1, max 500); default: 50
  - reasonCodes: TWOFACTOR_VAULT_DEPOSIT_SUBMITTED, TWOFACTOR_VAULT_REDEEM_SUBMITTED, TWOFACTOR_YIELD_BELOW_FLOOR, TWOFACTOR_YIELD_ABOVE_EXIT, TWOFACTOR_EXIT_CAPPED, TWOFACTOR_SIM_REVERTED, TWOFACTOR_APPROVE_SUBMITTED, TWOFACTOR_NO_BALANCE, TWOFACTOR_UNAVAILABLE
- `twofactor-carry` -- 2Factor Junior Carry: Hold perpJr (BTC at about 1.33x, no liquidation) and short the same BTC on Hyperliquid, earning venue funding on the full notional. Mints treasury cbBTC through the 2Factor Router with a subDR ceiling and opens the short once the mint is mined. CLOSE redeems perpJr and hands the cbBTC to the next block. A USDC treasury buys and sells the cbBTC with Swap blocks on either side.
  - io: input, outputs: done + waiting
  - strategyClasses: hedged-carry
  - config: `mode` (select) one of: "OPEN", "MAINTAIN", "CLOSE"; default: "OPEN" | `positionTag` (text); default: "2f-jr-btc" | `fundingSource` (select) one of: "Treasury cbBTC"; default: "Treasury cbBTC" -- Mints cbBTC the treasury holds. From USDC, place a Swap block (USDC to cbBTC) before OPEN. | `hedgeRatio` (select) one of: "Full (neutral)"; default: "Full (neutral)" | `allocation` (number, %, min 0, max 100); default: 100 -- Share of treasury cbBTC the carry takes and mints into perpJr. | `enterAbove` (number, %, min 0, max 200); default: 5 -- Minimum Jr spread after entry; an entry that would dilute it further is sized down. Also sets the Router's subDR ceiling, so the mint reverts if the system moved past it. | `exitBelow` (number, %, min -100, max 200); default: 3 -- CLOSE unwinds once the Jr spread falls below this. | `band` (number, %, min 0.5, max 20); default: 2 -- MAINTAIN resizes the short once net BTC drifts past this share of it. | `hedgeLeverage` (number, x, min 1, max 10); default: 2 -- Margin the short needs on Hyperliquid is its notional over this. | `maxExitFee` (number, %, min 0, max 10); default: 0.5 -- CLOSE redeems at most the perpJr whose exit fee stays within this. | `slippageBps` (number, bps, min 1, max 500); default: 10 -- Router min-out tolerance against the simulated amount. | `releaseIdleAfter` (number, min, min 0, max 1440); default: 0 -- CLOSE hands on cbBTC the carry holds but never minted, once it has sat this long. 0 keeps it; use 0 when the treasury holds cbBTC of its own.
  - reasonCodes: TWOFACTOR_MINT_PENDING, TWOFACTOR_IDLE_CBBTC, TWOFACTOR_CBBTC_RELEASED, TWOFACTOR_CARRY_OPENED, TWOFACTOR_SPREAD_BELOW_FLOOR, TWOFACTOR_HEDGE_MARGIN_SHORT, TWOFACTOR_REBALANCED, TWOFACTOR_IN_BAND, TWOFACTOR_CARRY_CLOSING, TWOFACTOR_CARRY_CLOSED, TWOFACTOR_EXIT_CAPPED, TWOFACTOR_SIM_REVERTED, TWOFACTOR_APPROVE_SUBMITTED, TWOFACTOR_NO_BALANCE, TWOFACTOR_NOT_OPEN, TWOFACTOR_UNAVAILABLE
- `twofactor-mint` -- 2Factor Mint: Mint perpJr (BTC at about 1.33x, no liquidation) with cbBTC through the 2Factor Router, under a subDR ceiling and simulated first. Sized down until the Jr spread after entry clears the floor and the Hyperliquid margin covers the hedge. Hands the hedge size to a Perp Position after it, which shorts once the mint is mined.
  - io: input, outputs: default + waiting
  - strategyClasses: hedged-carry
  - config: `positionTag` (text); default: "2f-jr-btc" | `allocation` (number, %, min 0, max 100); default: 100 -- Share of treasury cbBTC the carry takes. cbBTC a Swap hands over is used first, up to what the carry owns. | `enterAbove` (number, %, min 0, max 200); default: 5 -- Minimum Jr spread after entry; an entry that would dilute it further is sized down. Also sets the Router's subDR ceiling, so the mint reverts if the system moved past it. | `hedgeLeverage` (number, x, min 1, max 10); default: 2 -- Margin the short needs on Hyperliquid is its notional over this; the mint is sized to it. | `slippageBps` (number, bps, min 1, max 500); default: 10 -- Router min-out tolerance against the simulated amount.
  - reasonCodes: TWOFACTOR_MINTED, TWOFACTOR_MINT_PENDING, TWOFACTOR_IN_BAND, TWOFACTOR_SPREAD_BELOW_FLOOR, TWOFACTOR_HEDGE_MARGIN_SHORT, TWOFACTOR_SIM_REVERTED, TWOFACTOR_APPROVE_SUBMITTED, TWOFACTOR_NO_BALANCE, TWOFACTOR_UNAVAILABLE
- `twofactor-redeem` -- 2Factor Redeem: Redeem perpJr for cbBTC through the 2Factor Router, cut to what exits within the fee cap and simulated first. Hands the cbBTC to a Swap and the matching share of the short to a reduce-only Perp Position, which closes it once the redeem is mined.
  - io: input, outputs: default + waiting
  - strategyClasses: hedged-carry
  - config: `positionTag` (text); default: "2f-jr-btc" | `maxExitFee` (number, %, min 0, max 10); default: 0.5 -- Redeems at most the perpJr whose exit fee stays within this. | `slippageBps` (number, bps, min 1, max 500); default: 10 -- Router min-out tolerance against the simulated amount.
  - reasonCodes: TWOFACTOR_REDEEMED, TWOFACTOR_EXIT_CAPPED, TWOFACTOR_SIM_REVERTED, TWOFACTOR_APPROVE_SUBMITTED, TWOFACTOR_NOT_OPEN, TWOFACTOR_UNAVAILABLE

### Guards (category: guard)

- `max-allocation` -- Max Allocation: Cap exposure by venue, position, category, or portfolio gross.
  - io: input, outputs: passed + blocked
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `cap` (number, %, min 1, max 100); default: 40 | `scope` (select) one of: "Single venue", "Single position", "Category", "Portfolio gross"; default: "Single venue"
  - reasonCodes: GUARD_ALLOCATION_PASSED, GUARD_ALLOCATION_BLOCKED
- `slippage-cap` -- Slippage Cap: Block execution above a slippage limit.
  - io: input, outputs: passed + blocked
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `cap` (number, bps, min 1, max 1000); default: 50
  - reasonCodes: GUARD_SLIPPAGE_PASSED, GUARD_SLIPPAGE_BLOCKED
- `cooldown` -- Cooldown: Limit how often this path can execute.
  - io: input, outputs: passed + blocked
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `hours` (number, h, min 0, max 720); default: 24 | `scope` (select) one of: "This path", "Per instrument", "Per position tag"; default: "This path" | `condition` (select) one of: "Always", "After adverse exit only"; default: "Always"
  - reasonCodes: GUARD_COOLDOWN_PASSED, GUARD_COOLDOWN_BLOCKED
- `hold-timer` -- Hold Timer: Passes once a matching open position has been held for the minimum time. Gates exits so a position is not closed on the tick that opened it.
  - io: input, outputs: passed + blocked
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `minHold` (number, min, min 1, max 20160); default: 60 | `match` (text); default: "" -- Matched against the position tag, case-insensitive substring: "ETH" matches hl-perp-ETH. Blank matches any open position.
  - reasonCodes: GUARD_HOLD_PASSED, GUARD_HOLD_BLOCKED
- `ev-gate` -- EV Gate: Blocks when modelled edge does not clear round-trip cost by a configured multiple.
  - io: input, outputs: passed + blocked
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `minEdgeMultiple` (number, min 1, max 20); default: 3 | `expectedHold` (select) one of: "1 day", "7 days", "14 days", "30 days"; default: "7 days" | `edgeMetric` (select) one of: "Modelled edge", "2Factor Jr spread after entry (annualized)", "2Factor Jr spread (annualized)", "2Factor vault yield after entry (annualized)", "2Factor vault yield (annualized)"; default: "Modelled edge" -- Modelled edge is the venue funding rate over the expected hold, against the venue's fees and slippage. With a 2Factor block on the canvas, a 2Factor rate can be chosen instead; it is costed with the 2Factor round-trip cost.
  - reasonCodes: GUARD_EV_PASSED, GUARD_EV_BLOCKED
- `exposure-guard` -- Exposure Guard: Evaluates the portfolio after the proposed trade (gross, concurrent positions, net beta).
  - io: input, outputs: passed + blocked
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `maxGross` (number, %, min 1, max 500); default: 100 | `maxConcurrent` (number, min 1, max 50); default: 3
  - reasonCodes: GUARD_EXPOSURE_PASSED, GUARD_EXPOSURE_BLOCKED
- `margin-health` -- Margin Health: Liquidation distance and margin buffer — survival, not size (unlike max-allocation).
  - io: input, outputs: passed + blocked
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `minLiqDistance` (number, %, min 1, max 90); default: 25
  - reasonCodes: GUARD_MARGIN_PASSED, GUARD_MARGIN_BLOCKED
- `quarantine-guard` -- Quarantine Guard: Stateful: blocks entry while a pair/instrument is serving a cooldown after adverse exit.
  - io: input, outputs: passed + blocked
  - strategyClasses: market-neutral
  - config: `pairRef` (text); default: "pairA"
  - reasonCodes: GUARD_QUARANTINE_BLOCKED, GUARD_QUARANTINE_PASSED
- `kill-switch` -- Kill Switch: Global overlay (no inbound required): drawdown / regime. Stage 2 routes every open positionTag to CLOSE.
  - io: no input, outputs: passed + blocked
  - strategyClasses: directional, market-neutral, hedged-carry
  - config: `maxDrawdown` (number, %, min 1, max 50); default: 8 | `scope` (select) one of: "Global"; default: "Global"
  - reasonCodes: KILL_SWITCH_CLEAR, KILL_SWITCH_STAGE1, KILL_SWITCH_STAGE2

### Messages (category: message)

- `journal` -- Journal: Explicit annotation sink for blocked or notable paths. Platform journaling still runs on every node.
  - io: input, no output
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `label` (text); default: "Blocked path" -- What this entry says. Each journal on the graph writes its own.
  - reasonCodes: JOURNAL_WRITTEN
- `notify` -- Notify: Page a human without gating flow. Escalation sink.
  - io: input, no output
  - strategyClasses: treasury-yield, directional, market-neutral, hedged-carry
  - config: `message` (text); default: "" -- What this alert says. Left empty, it lists the reasons that led here. | `severity` (select) one of: "Info", "Warning", "Critical"; default: "Warning" | `channel` (select) one of: "Telegram", "Email", "Pager"; default: "Telegram"
  - reasonCodes: NOTIFY_SENT
