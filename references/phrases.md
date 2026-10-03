# Phrases

What users say, and what it means here. Check every key and option
against the node catalog in `structures.md` before using it.

## Building a strategy

| The user says | Build with |
| --- | --- |
| "put idle cash to work", "earn yield on our USDC" | `idle-capital` or `deposit-received` -> `liquidity-buffer` -> `yield-optimizer` -> `max-allocation` -> `lend` (Idle Treasury Optimizer) |
| "keep enough liquid for payroll", "a buffer for withdrawals" | `liquidity-buffer` before anything that deploys capital |
| "fund withdrawals from what is deployed" | `withdrawal-approved` -> `withdraw` -> `payout` |
| "when deposits arrive" | `deposit-received` trigger |
| "buy ETH every day", "DCA", "accumulate slowly" | `schedule` -> `max-allocation` -> `slippage-cap` -> `twap` (TWAP Accumulator) |
| "rebalance to 60/40", "keep target weights" | `schedule` -> `rebalance` with `weights` |
| "earn funding", "delta-neutral carry", "basis trade" | Funding Carry (Hyperliquid): `funding-feed`, `ev-gate`, `spot-position` long + `perp-position` short, `position-health`, `leg-check` |
| "pairs trade", "stat arb", "mean reversion between two coins" | Pair Trade (Hyperliquid): `pair-screener` -> `pair-stats` -> guards -> `paired-execution` |
| "loop a stablecoin", "lever up a yield spread" | Stablecoin Loop: `loop-check` -> `vault`, `supply-collateral`, `borrow`; or the single `loop` block |
| "provide liquidity on Uniswap", "concentrated LP" | `lp-stake` (LP Staking Ladder); watch and close it with `lp-monitor`, `lp-rules`, `lp-exit` |
| "hedged LP", "LP without ETH exposure" | Hedged Concentrated LP: `supply-collateral`, `borrow`, `lp-stake`, `hedge-check`, `debt-check`, `repay` |
| "trade my TradingView alerts", "follow my model's signals" | `webhook` trigger. Perps: Hyperliquid Signal Trader. Base tokens: Base Signal Accumulator |
| "trade Tesla / Nvidia / stocks" | `perp-position` with `symbol` `xyz:TSLA` (HIP-3), or `symbolFrom` "Webhook signal" to trade whatever stock the signal names |
| "whatever ticker the alert says", "the symbol from the signal" | the receiving field set to "Webhook signal" on every block after the `webhook` |
| "only when funding is above X%", "when volatility is low" | `signal-threshold` as the trigger, or `signal-gate` / `market-conditions` mid-flow |
| "stop everything if we lose 10%" | `kill-switch` with `maxDrawdown` 10 |
| "no more than once a day" | `cooldown` with `hours` 24 |
| "hold at least a day before selling" | `hold-timer` with `minHold` 1440 (minutes) before the exit |
| "no more than 20% in any one position" | `max-allocation` with `cap` 20 and `scope` "Single position" |
| "at most 3 positions", "cap gross exposure" | `exposure-guard` |
| "stay far from liquidation" | `margin-health` |
| "tell me when it happens", "page me" | `notify` |

## Sending a signal

The action names an output of the agent's `webhook` trigger; what it does
depends on what is wired to that output. In the Hyperliquid Signal Trader
and the Base Signal Accumulator it means:

| Action | Signal Trader (perps) | Signal Accumulator (Base) |
| --- | --- | --- |
| `buy` | close any short, open a long | buy the token over time (TWAP) |
| `sell` | close any long, open a short | sell the treasury's holding to USDC |
| `exit` | close the long and the short | sell the treasury's holding to USDC |

| The user says | Send |
| --- | --- |
| "buy ETH", "go long ETH" | `{"action":"buy","symbol":"ETH"}` |
| "short BTC", "go short bitcoin" | `{"action":"sell","symbol":"BTC"}` |
| "close my SOL", "flatten SOL", "get out of SOL" | `{"action":"exit","symbol":"SOL"}` |
| "long Tesla" | `{"action":"buy","symbol":"xyz:TSLA"}`, or `TSLA` to take the most-traded listing |
| "half size", "a quarter of the usual" | add `"size_pct":50` or `"size_pct":25` |
| "$500 of ETH" | add `"size_usd":500`; honoured only up to the trigger's `maxSizeUsd`, and never above the block's own size |
| "buy this token 0x..." | `{"action":"buy","symbol":"0x..."}`; the address must be on the policy whitelist |
| "take profit", "trim" | ambiguous: ask whether to close (`exit`) or reduce, since `sell` reverses a perp position |

Always add an `id`. Ask before sending when the agent, action or symbol
is unclear, and never raise a size the user did not give.
