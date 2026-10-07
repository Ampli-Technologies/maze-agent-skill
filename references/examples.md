# Examples

## Compositions

Every template Maze Studio ships, as importable JSON. Start from the
closest one; the first three cover the common cases: a treasury that
earns yield, a Hyperliquid perp trader driven by signals, and a Base
token accumulator driven by signals.

### Idle Treasury Optimizer

`idle-treasury-optimizer`, treasury-yield. Keep a liquidity buffer for payments, put surplus to work via Yield Optimizer (policy + health aware), and pull it back when a withdrawal needs it. The app then sends the withdrawal, and the next run redeploys the rest.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Idle Treasury Optimizer",
  "strategyClass": "treasury-yield",
  "nodes": [
    {
      "id": "deposit",
      "kind": "deposit-received",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "minAmount": 1000,
        "asset": "USDC"
      }
    },
    {
      "id": "idle",
      "kind": "idle-capital",
      "position": {
        "x": 0,
        "y": 220
      },
      "config": {
        "idleAfter": "24 hours",
        "minIdle": 25000
      }
    },
    {
      "id": "buffer",
      "kind": "liquidity-buffer",
      "position": {
        "x": 320,
        "y": 110
      },
      "config": {
        "target": 50000,
        "asset": "USDC"
      }
    },
    {
      "id": "yield",
      "kind": "yield-optimizer",
      "position": {
        "x": 640,
        "y": 110
      },
      "config": {
        "provider": "Ampli",
        "objective": "Risk-adjusted",
        "respectPolicies": true,
        "healthFloor": "Standard"
      }
    },
    {
      "id": "cap",
      "kind": "max-allocation",
      "position": {
        "x": 960,
        "y": 40
      },
      "config": {
        "cap": 40,
        "scope": "Single venue"
      }
    },
    {
      "id": "lend",
      "kind": "lend",
      "positionTag": "yield-pos",
      "position": {
        "x": 1250,
        "y": 40
      },
      "config": {
        "protocol": "Best available",
        "allocation": 40
      }
    },
    {
      "id": "hold",
      "kind": "treasury-vault",
      "position": {
        "x": 960,
        "y": 260
      },
      "config": {
        "asset": "USDC"
      }
    },
    {
      "id": "wd-trigger",
      "kind": "withdrawal-approved",
      "position": {
        "x": 0,
        "y": 480
      },
      "config": {
        "minAmount": 0
      }
    },
    {
      "id": "withdraw",
      "kind": "withdraw",
      "positionTag": "yield-pos",
      "position": {
        "x": 320,
        "y": 480
      },
      "config": {}
    },
    {
      "id": "payout",
      "kind": "payout",
      "position": {
        "x": 640,
        "y": 480
      },
      "config": {}
    },
    {
      "id": "buffer-breach-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buffer under target; nothing deployed"
      }
    },
    {
      "id": "cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the allocation cap; surplus left in the treasury"
      }
    }
  ],
  "edges": [
    {
      "source": "deposit",
      "target": "buffer"
    },
    {
      "source": "idle",
      "target": "buffer"
    },
    {
      "source": "buffer",
      "target": "yield",
      "sourceHandle": "surplus"
    },
    {
      "source": "yield",
      "target": "cap",
      "sourceHandle": "primary"
    },
    {
      "source": "cap",
      "target": "lend",
      "sourceHandle": "passed"
    },
    {
      "source": "yield",
      "target": "hold",
      "sourceHandle": "fallback"
    },
    {
      "source": "wd-trigger",
      "target": "withdraw"
    },
    {
      "source": "withdraw",
      "target": "payout"
    },
    {
      "source": "buffer",
      "target": "buffer-breach-journal",
      "sourceHandle": "breach"
    },
    {
      "source": "cap",
      "target": "cap-blocked-journal",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Hyperliquid Signal Trader

`hyperliquid-signal-trader`, directional. Trade Hyperliquid perps from outside signals. TradingView or any source posts buy, sell or exit with a ticker to this agent's webhook address: buy closes any short and opens a long, sell closes any long and opens a short, exit closes both. The ticker is whatever the signal names; your policy's Hyperliquid universe (named tickers, or volume, open interest and rank floors) decides which may trade, and each ticker keeps its own long and short. Opens are sized from the Hyperliquid account at up to 2x and pass exposure and liquidation-distance guards; a payload may ask for a smaller share. A repeated signal adds nothing, and the kill switch stops entries past 8% drawdown.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Hyperliquid Signal Trader",
  "strategyClass": "directional",
  "nodes": [
    {
      "id": "hook",
      "kind": "webhook",
      "position": {
        "x": 0,
        "y": 260
      },
      "config": {
        "maxAge": "5 min",
        "maxSizeUsd": 0
      }
    },
    {
      "id": "close-short",
      "kind": "perp-position",
      "positionTag": "hl-short",
      "position": {
        "x": 300,
        "y": 60
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "long-exposure",
      "kind": "exposure-guard",
      "position": {
        "x": 600,
        "y": 60
      },
      "config": {
        "maxGross": 300,
        "maxConcurrent": 2
      }
    },
    {
      "id": "long-margin",
      "kind": "margin-health",
      "position": {
        "x": 900,
        "y": 60
      },
      "config": {
        "minLiqDistance": 25
      }
    },
    {
      "id": "open-long",
      "kind": "perp-position",
      "positionTag": "hl-long",
      "position": {
        "x": 1200,
        "y": 60
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Long",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "close-long",
      "kind": "perp-position",
      "positionTag": "hl-long",
      "position": {
        "x": 300,
        "y": 300
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Long",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "short-exposure",
      "kind": "exposure-guard",
      "position": {
        "x": 600,
        "y": 300
      },
      "config": {
        "maxGross": 300,
        "maxConcurrent": 2
      }
    },
    {
      "id": "short-margin",
      "kind": "margin-health",
      "position": {
        "x": 900,
        "y": 300
      },
      "config": {
        "minLiqDistance": 25
      }
    },
    {
      "id": "open-short",
      "kind": "perp-position",
      "positionTag": "hl-short",
      "position": {
        "x": 1200,
        "y": 300
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "exit-long",
      "kind": "perp-position",
      "positionTag": "hl-long",
      "position": {
        "x": 300,
        "y": 540
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Long",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "exit-short",
      "kind": "perp-position",
      "positionTag": "hl-short",
      "position": {
        "x": 600,
        "y": 540
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Webhook signal",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "kill",
      "kind": "kill-switch",
      "position": {
        "x": 0,
        "y": 700
      },
      "config": {
        "maxDrawdown": 8,
        "scope": "Global"
      }
    },
    {
      "id": "long-exposure-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exposure limit reached; long not opened"
      }
    },
    {
      "id": "long-margin-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Liquidation distance too thin; long not opened"
      }
    },
    {
      "id": "open-long-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Opened long on a buy signal"
      }
    },
    {
      "id": "open-long-none-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Already long; repeated buy signal ignored"
      }
    },
    {
      "id": "short-exposure-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exposure limit reached; short not opened"
      }
    },
    {
      "id": "short-margin-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Liquidation distance too thin; short not opened"
      }
    },
    {
      "id": "open-short-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Opened short on a sell signal"
      }
    },
    {
      "id": "open-short-none-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Already short; repeated sell signal ignored"
      }
    },
    {
      "id": "kill-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Kill switch tripped: drawdown over 8%",
        "channel": "Telegram",
        "severity": "Critical"
      }
    }
  ],
  "edges": [
    {
      "source": "hook",
      "target": "close-short",
      "sourceHandle": "buy"
    },
    {
      "source": "close-short",
      "target": "long-exposure",
      "sourceHandle": "default"
    },
    {
      "source": "close-short",
      "target": "long-exposure",
      "sourceHandle": "none"
    },
    {
      "source": "long-exposure",
      "target": "long-margin",
      "sourceHandle": "passed"
    },
    {
      "source": "long-margin",
      "target": "open-long",
      "sourceHandle": "passed"
    },
    {
      "source": "hook",
      "target": "close-long",
      "sourceHandle": "sell"
    },
    {
      "source": "close-long",
      "target": "short-exposure",
      "sourceHandle": "default"
    },
    {
      "source": "close-long",
      "target": "short-exposure",
      "sourceHandle": "none"
    },
    {
      "source": "short-exposure",
      "target": "short-margin",
      "sourceHandle": "passed"
    },
    {
      "source": "short-margin",
      "target": "open-short",
      "sourceHandle": "passed"
    },
    {
      "source": "hook",
      "target": "exit-long",
      "sourceHandle": "exit"
    },
    {
      "source": "exit-long",
      "target": "exit-short",
      "sourceHandle": "default"
    },
    {
      "source": "exit-long",
      "target": "exit-short",
      "sourceHandle": "none"
    },
    {
      "source": "long-exposure",
      "target": "long-exposure-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "long-margin",
      "target": "long-margin-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "open-long",
      "target": "open-long-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "open-long",
      "target": "open-long-none-journal",
      "sourceHandle": "none"
    },
    {
      "source": "short-exposure",
      "target": "short-exposure-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "short-margin",
      "target": "short-margin-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "open-short",
      "target": "open-short-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "open-short",
      "target": "open-short-none-journal",
      "sourceHandle": "none"
    },
    {
      "source": "kill",
      "target": "kill-blocked-alert",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Base Signal Accumulator

`base-signal-accumulator`, directional. Accumulate Base tokens from outside signals. TradingView or any source posts buy, sell or exit with a token to this agent's webhook address, by ticker (AERO, cbBTC) or contract address, such as a tokenized stock issued on Base. A buy spends a fifth of the USDC through an Orbs dTWAP over an hour, inside a 50% allocation cap; a sell or exit swaps the treasury's whole holding of that token back to USDC. The token passes from the signal to both orders; only tokens whose address is on your policy whitelist can be bought or sold. The kill switch stops buys past 8% drawdown.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Base Signal Accumulator",
  "strategyClass": "directional",
  "nodes": [
    {
      "id": "hook",
      "kind": "webhook",
      "position": {
        "x": 0,
        "y": 200
      },
      "config": {
        "maxAge": "5 min",
        "maxSizeUsd": 0
      }
    },
    {
      "id": "buy-cap",
      "kind": "max-allocation",
      "position": {
        "x": 300,
        "y": 60
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "buy-slippage",
      "kind": "slippage-cap",
      "position": {
        "x": 600,
        "y": 60
      },
      "config": {
        "cap": 50
      }
    },
    {
      "id": "accumulate",
      "kind": "twap",
      "position": {
        "x": 900,
        "y": 60
      },
      "config": {
        "asset": "Webhook signal",
        "tokenAddress": "",
        "allocation": 20,
        "duration": "1 hour",
        "slices": 6,
        "limitBand": 50
      }
    },
    {
      "id": "exit-slippage",
      "kind": "slippage-cap",
      "position": {
        "x": 300,
        "y": 340
      },
      "config": {
        "cap": 50
      }
    },
    {
      "id": "sell",
      "kind": "swap",
      "position": {
        "x": 600,
        "y": 340
      },
      "config": {
        "from": "Webhook signal",
        "to": "USDC",
        "venue": "Orbs dTWAP",
        "sizeFrom": "Share of balance",
        "allocation": 100,
        "maxSlippage": 50
      }
    },
    {
      "id": "kill",
      "kind": "kill-switch",
      "position": {
        "x": 0,
        "y": 520
      },
      "config": {
        "maxDrawdown": 8,
        "scope": "Global"
      }
    },
    {
      "id": "buy-cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the allocation cap; nothing bought"
      }
    },
    {
      "id": "buy-slippage-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Price moved past the slippage cap; nothing bought"
      }
    },
    {
      "id": "accumulate-filled-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Bought the token the signal named"
      }
    },
    {
      "id": "accumulate-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buy order did not fill"
      }
    },
    {
      "id": "exit-slippage-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Price moved past the slippage cap; nothing sold"
      }
    },
    {
      "id": "sell-filled-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Sold the token the signal named back to USDC"
      }
    },
    {
      "id": "sell-none-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "None of that token held; exit ignored"
      }
    },
    {
      "id": "sell-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exit order did not fill"
      }
    },
    {
      "id": "kill-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Kill switch tripped: drawdown over 8%",
        "channel": "Telegram",
        "severity": "Critical"
      }
    }
  ],
  "edges": [
    {
      "source": "hook",
      "target": "buy-cap",
      "sourceHandle": "buy"
    },
    {
      "source": "buy-cap",
      "target": "buy-slippage",
      "sourceHandle": "passed"
    },
    {
      "source": "buy-slippage",
      "target": "accumulate",
      "sourceHandle": "passed"
    },
    {
      "source": "hook",
      "target": "exit-slippage",
      "sourceHandle": "sell"
    },
    {
      "source": "hook",
      "target": "exit-slippage",
      "sourceHandle": "exit"
    },
    {
      "source": "exit-slippage",
      "target": "sell",
      "sourceHandle": "passed"
    },
    {
      "source": "buy-cap",
      "target": "buy-cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "buy-slippage",
      "target": "buy-slippage-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "accumulate",
      "target": "accumulate-filled-journal",
      "sourceHandle": "filled"
    },
    {
      "source": "accumulate",
      "target": "accumulate-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "exit-slippage",
      "target": "exit-slippage-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "sell",
      "target": "sell-filled-journal",
      "sourceHandle": "filled"
    },
    {
      "source": "sell",
      "target": "sell-none-journal",
      "sourceHandle": "none"
    },
    {
      "source": "sell",
      "target": "sell-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "kill",
      "target": "kill-blocked-alert",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Stablecoin Loop

`stablecoin-loop`, treasury-yield. Every 15 minutes on Base, fold yoUSD against a USDC borrow on Morpho toward 80% LTV while the AI risk check agrees and the spread pays: Loop Check opens with half the idle USDC (Vault deposit, Supply Collateral), then each run folds once (Borrow, Vault deposit, Supply Collateral). A health breach, a negative spread or a position over target unwinds a step (Withdraw Collateral, Vault redeem, Repay); a risk-off read or an adverse-exit cooldown unwinds it all back to USDC.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Stablecoin Loop",
  "strategyClass": "treasury-yield",
  "nodes": [
    {
      "id": "schedule",
      "kind": "schedule",
      "position": {
        "x": 0,
        "y": 60
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "vault",
      "kind": "treasury-vault",
      "position": {
        "x": 0,
        "y": 300
      },
      "config": {
        "asset": "USDC"
      }
    },
    {
      "id": "risk",
      "kind": "risk-check",
      "position": {
        "x": 300,
        "y": 170
      },
      "config": {
        "model": "Ensemble",
        "maxRisk": "Medium",
        "timeout": 30
      }
    },
    {
      "id": "cooldown",
      "kind": "cooldown",
      "position": {
        "x": 600,
        "y": 60
      },
      "config": {
        "hours": 12,
        "scope": "This path",
        "condition": "After adverse exit only"
      }
    },
    {
      "id": "check",
      "kind": "loop-check",
      "position": {
        "x": 900,
        "y": 60
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "targetLtv": 80,
        "band": 3,
        "minSpread": 1,
        "minHealthFactor": 1.08,
        "maxSlippage": 30
      }
    },
    {
      "id": "exit-check",
      "kind": "loop-check",
      "position": {
        "x": 900,
        "y": 420
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "targetLtv": 0,
        "band": 3,
        "minSpread": 1,
        "minHealthFactor": 1.08,
        "maxSlippage": 30
      }
    },
    {
      "id": "seed",
      "kind": "vault",
      "position": {
        "x": 1200,
        "y": -120
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "action": "Deposit",
        "sizeFrom": "Share of balance",
        "allocation": 50
      }
    },
    {
      "id": "seed-supply",
      "kind": "supply-collateral",
      "positionTag": "loop-pos",
      "position": {
        "x": 1500,
        "y": -120
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "sizeFrom": "Upstream output",
        "allocation": 100
      }
    },
    {
      "id": "borrow",
      "kind": "borrow",
      "position": {
        "x": 1800,
        "y": 60
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "sizeFrom": "Target LTV",
        "targetLtv": 80,
        "minHealthFactor": 1.08
      }
    },
    {
      "id": "fold",
      "kind": "vault",
      "position": {
        "x": 2100,
        "y": 60
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "action": "Deposit",
        "sizeFrom": "Upstream output",
        "allocation": 100
      }
    },
    {
      "id": "fold-supply",
      "kind": "supply-collateral",
      "positionTag": "loop-pos",
      "position": {
        "x": 2400,
        "y": 60
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "sizeFrom": "Upstream output",
        "allocation": 100
      }
    },
    {
      "id": "withdraw",
      "kind": "withdraw-collateral",
      "position": {
        "x": 1200,
        "y": 300
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "sizeFrom": "Upstream output",
        "minHealthFactor": 1.02
      }
    },
    {
      "id": "redeem",
      "kind": "vault",
      "position": {
        "x": 1500,
        "y": 300
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "action": "Redeem",
        "sizeFrom": "Upstream output",
        "allocation": 100
      }
    },
    {
      "id": "repay",
      "kind": "repay",
      "position": {
        "x": 1800,
        "y": 300
      },
      "config": {
        "market": "yoUSD / USDC on Morpho (Base)",
        "sizeFrom": "Upstream output"
      }
    },
    {
      "id": "check-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Loop Check could not read the market"
      }
    },
    {
      "id": "exit-check-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exit Loop Check could not read the market"
      }
    },
    {
      "id": "seed-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Opening vault deposit failed"
      }
    },
    {
      "id": "seed-supply-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Supplying the opening collateral failed"
      }
    },
    {
      "id": "borrow-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Borrow to fold failed"
      }
    },
    {
      "id": "fold-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Folding vault deposit failed"
      }
    },
    {
      "id": "fold-supply-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Supplying the folded collateral failed"
      }
    },
    {
      "id": "withdraw-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Withdrawing collateral to unwind failed"
      }
    },
    {
      "id": "redeem-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Vault redeem during the unwind failed"
      }
    },
    {
      "id": "repay-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Repay during the unwind failed; debt still open"
      }
    }
  ],
  "edges": [
    {
      "source": "schedule",
      "target": "vault"
    },
    {
      "source": "vault",
      "target": "risk"
    },
    {
      "source": "risk",
      "target": "cooldown",
      "sourceHandle": "primary"
    },
    {
      "source": "risk",
      "target": "exit-check",
      "sourceHandle": "fallback"
    },
    {
      "source": "cooldown",
      "target": "check",
      "sourceHandle": "passed"
    },
    {
      "source": "cooldown",
      "target": "exit-check",
      "sourceHandle": "blocked"
    },
    {
      "source": "check",
      "target": "seed",
      "sourceHandle": "open"
    },
    {
      "source": "seed",
      "target": "seed-supply",
      "sourceHandle": "done"
    },
    {
      "source": "seed-supply",
      "target": "borrow",
      "sourceHandle": "done"
    },
    {
      "source": "check",
      "target": "borrow",
      "sourceHandle": "fold"
    },
    {
      "source": "borrow",
      "target": "fold",
      "sourceHandle": "done"
    },
    {
      "source": "fold",
      "target": "fold-supply",
      "sourceHandle": "done"
    },
    {
      "source": "check",
      "target": "withdraw",
      "sourceHandle": "unwind"
    },
    {
      "source": "exit-check",
      "target": "withdraw",
      "sourceHandle": "unwind"
    },
    {
      "source": "withdraw",
      "target": "redeem",
      "sourceHandle": "done"
    },
    {
      "source": "redeem",
      "target": "repay",
      "sourceHandle": "done"
    },
    {
      "source": "check",
      "target": "check-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "exit-check",
      "target": "exit-check-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "seed",
      "target": "seed-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "seed-supply",
      "target": "seed-supply-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "borrow",
      "target": "borrow-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "fold",
      "target": "fold-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "fold-supply",
      "target": "fold-supply-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "withdraw",
      "target": "withdraw-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "redeem",
      "target": "redeem-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "repay",
      "target": "repay-failed-journal",
      "sourceHandle": "failed"
    }
  ]
}
```

### TWAP Accumulator

`twap-accumulator`, treasury-yield. Each day, buy ETH with half the USDC through an Orbs dTWAP order, inside a 50% allocation cap; once every slice has filled, stake ETH and USDC as LP behind a slippage cap.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "TWAP Accumulator",
  "strategyClass": "treasury-yield",
  "nodes": [
    {
      "id": "schedule",
      "kind": "schedule",
      "position": {
        "x": 0,
        "y": 100
      },
      "config": {
        "interval": "Daily"
      }
    },
    {
      "id": "cap",
      "kind": "max-allocation",
      "position": {
        "x": 150,
        "y": 100
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "twap",
      "kind": "twap",
      "position": {
        "x": 300,
        "y": 100
      },
      "config": {
        "asset": "ETH",
        "tokenAddress": "",
        "allocation": 50,
        "duration": "4 hours",
        "slices": 12,
        "limitBand": 40
      }
    },
    {
      "id": "slippage",
      "kind": "slippage-cap",
      "position": {
        "x": 620,
        "y": 100
      },
      "config": {
        "cap": 40
      }
    },
    {
      "id": "stake",
      "kind": "lp-stake",
      "position": {
        "x": 940,
        "y": 100
      },
      "config": {
        "pool": "ETH / USDC",
        "range": "Wide",
        "width": 0,
        "allocation": 50,
        "maxSlippage": 50
      }
    },
    {
      "id": "cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the allocation cap; no order placed"
      }
    },
    {
      "id": "twap-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "TWAP order failed"
      }
    },
    {
      "id": "slippage-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Slippage over the cap; LP not staked"
      }
    },
    {
      "id": "stake-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "LP stake failed; ETH and USDC left in the treasury"
      }
    }
  ],
  "edges": [
    {
      "source": "schedule",
      "target": "cap"
    },
    {
      "source": "cap",
      "target": "twap",
      "sourceHandle": "passed"
    },
    {
      "source": "twap",
      "target": "slippage",
      "sourceHandle": "filled"
    },
    {
      "source": "slippage",
      "target": "stake",
      "sourceHandle": "passed"
    },
    {
      "source": "cap",
      "target": "cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "twap",
      "target": "twap-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "slippage",
      "target": "slippage-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "stake",
      "target": "stake-failed-journal",
      "sourceHandle": "failed"
    }
  ]
}
```

### LP Staking Ladder

`lp-staking-ladder`, treasury-yield. Route deposits through a buffer; when markets allow, open concentrated LP, otherwise Yield Optimizer finds the best lending rate.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "LP Staking Ladder",
  "strategyClass": "treasury-yield",
  "nodes": [
    {
      "id": "deposit",
      "kind": "deposit-received",
      "position": {
        "x": 0,
        "y": 140
      },
      "config": {
        "minAmount": 5000,
        "asset": "USDC"
      }
    },
    {
      "id": "buffer",
      "kind": "liquidity-buffer",
      "position": {
        "x": 300,
        "y": 140
      },
      "config": {
        "target": 25000,
        "asset": "USDC"
      }
    },
    {
      "id": "market",
      "kind": "market-conditions",
      "position": {
        "x": 620,
        "y": 140
      },
      "config": {
        "metric": "Volatility",
        "window": "7 days",
        "multiple": 2
      }
    },
    {
      "id": "cap",
      "kind": "max-allocation",
      "position": {
        "x": 940,
        "y": 40
      },
      "config": {
        "cap": 30,
        "scope": "Single venue"
      }
    },
    {
      "id": "stake",
      "kind": "lp-stake",
      "position": {
        "x": 1240,
        "y": 40
      },
      "config": {
        "pool": "ETH / USDC",
        "range": "Concentrated",
        "width": 0,
        "allocation": 30,
        "maxSlippage": 50
      }
    },
    {
      "id": "yield",
      "kind": "yield-optimizer",
      "position": {
        "x": 940,
        "y": 300
      },
      "config": {
        "provider": "Ampli",
        "objective": "Stable only",
        "respectPolicies": true,
        "healthFloor": "Strict"
      }
    },
    {
      "id": "lend-cap",
      "kind": "max-allocation",
      "position": {
        "x": 1090,
        "y": 300
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "lend",
      "kind": "lend",
      "position": {
        "x": 1240,
        "y": 300
      },
      "config": {
        "protocol": "Best available",
        "allocation": 50
      }
    },
    {
      "id": "buffer-breach-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buffer under target; nothing deployed"
      }
    },
    {
      "id": "cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the LP allocation cap; not staked"
      }
    },
    {
      "id": "stake-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "LP stake failed"
      }
    },
    {
      "id": "yield-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "No lending rate cleared the policy; left in the treasury"
      }
    },
    {
      "id": "lend-cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the lending allocation cap; not lent"
      }
    }
  ],
  "edges": [
    {
      "source": "deposit",
      "target": "buffer"
    },
    {
      "source": "buffer",
      "target": "market",
      "sourceHandle": "surplus"
    },
    {
      "source": "market",
      "target": "cap",
      "sourceHandle": "primary"
    },
    {
      "source": "cap",
      "target": "stake",
      "sourceHandle": "passed"
    },
    {
      "source": "market",
      "target": "yield",
      "sourceHandle": "fallback"
    },
    {
      "source": "yield",
      "target": "lend-cap",
      "sourceHandle": "primary"
    },
    {
      "source": "lend-cap",
      "target": "lend",
      "sourceHandle": "passed"
    },
    {
      "source": "buffer",
      "target": "buffer-breach-journal",
      "sourceHandle": "breach"
    },
    {
      "source": "cap",
      "target": "cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "stake",
      "target": "stake-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "yield",
      "target": "yield-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "lend-cap",
      "target": "lend-cap-blocked-journal",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Hedged Concentrated LP

`hedged-concentrated-lp`, market-neutral. Every 15 minutes: supply a quarter of idle USDC to Aave, borrow ETH at 50% LTV against it and open a concentrated ETH / USDC position 5% either side of the price with that ETH and as much USDC, so the treasury owes as much ETH as the position holds. Behind a 30 bps slippage cap, Hedge Check borrows more and sells it into collateral as ETH falls, and withdraws collateral to buy ETH back and repay as it rises. The range re-centres (Exit LP, then LP Stake inside the 50% allocation cap, then a hedge check) once the price has sat within 20% of an edge for 30 minutes. A 3% loss against holding, a health factor under 1.3, 12 hours out of range or fees that stop covering the borrow close it all back to USDC (Exit LP, Debt Check, Repay, Withdraw Collateral, Swap) and hold off for 12 hours. Every swap here is a Uniswap v3 Swap block, so each path lands as one batch.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Hedged Concentrated LP",
  "strategyClass": "market-neutral",
  "nodes": [
    {
      "id": "schedule",
      "kind": "schedule",
      "position": {
        "x": 0,
        "y": 200
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "rules",
      "kind": "lp-rules",
      "position": {
        "x": 300,
        "y": 200
      },
      "config": {
        "pool": "ETH / USDC",
        "maxLoss": 3,
        "outOfRangeHours": 12,
        "minHealthFactor": 1.3,
        "minCarry": 0,
        "carryAfterHours": 24
      }
    },
    {
      "id": "monitor",
      "kind": "lp-monitor",
      "position": {
        "x": 600,
        "y": 120
      },
      "config": {
        "pool": "ETH / USDC",
        "edgeBuffer": 20,
        "confirmMinutes": 30
      }
    },
    {
      "id": "cooldown",
      "kind": "cooldown",
      "position": {
        "x": 900,
        "y": -180
      },
      "config": {
        "hours": 12,
        "scope": "This path",
        "condition": "After adverse exit only"
      }
    },
    {
      "id": "cap",
      "kind": "max-allocation",
      "position": {
        "x": 1200,
        "y": -180
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "supply",
      "kind": "supply-collateral",
      "position": {
        "x": 1500,
        "y": -180
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Share of balance",
        "allocation": 25
      }
    },
    {
      "id": "borrow",
      "kind": "borrow",
      "position": {
        "x": 1800,
        "y": -180
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Target LTV",
        "targetLtv": 50,
        "minHealthFactor": 1.5
      }
    },
    {
      "id": "stake",
      "kind": "lp-stake",
      "positionTag": "hedged-lp",
      "position": {
        "x": 2100,
        "y": -180
      },
      "config": {
        "pool": "ETH / USDC",
        "range": "Concentrated",
        "width": 5,
        "allocation": 40,
        "maxSlippage": 50
      }
    },
    {
      "id": "hedge-slip",
      "kind": "slippage-cap",
      "position": {
        "x": 900,
        "y": 120
      },
      "config": {
        "cap": 30
      }
    },
    {
      "id": "hedge",
      "kind": "hedge-check",
      "position": {
        "x": 1200,
        "y": 120
      },
      "config": {
        "pool": "ETH / USDC",
        "hedgeRatio": 100,
        "drift": 5,
        "minHealthFactor": 1.5,
        "maxSlippage": 30
      }
    },
    {
      "id": "h-borrow",
      "kind": "borrow",
      "position": {
        "x": 1500,
        "y": 40
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Upstream output",
        "targetLtv": 50,
        "minHealthFactor": 1.1
      }
    },
    {
      "id": "h-sell",
      "kind": "swap",
      "position": {
        "x": 1800,
        "y": 40
      },
      "config": {
        "from": "ETH",
        "to": "USDC",
        "venue": "Uniswap v3",
        "sizeFrom": "Upstream output",
        "allocation": 100,
        "maxSlippage": 30
      }
    },
    {
      "id": "h-supply",
      "kind": "supply-collateral",
      "position": {
        "x": 2100,
        "y": 40
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Upstream output",
        "allocation": 100
      }
    },
    {
      "id": "h-withdraw",
      "kind": "withdraw-collateral",
      "position": {
        "x": 1500,
        "y": 200
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Upstream output",
        "minHealthFactor": 1.1
      }
    },
    {
      "id": "h-buy",
      "kind": "swap",
      "position": {
        "x": 1800,
        "y": 200
      },
      "config": {
        "from": "USDC",
        "to": "ETH",
        "venue": "Uniswap v3",
        "sizeFrom": "Upstream output",
        "allocation": 100,
        "maxSlippage": 30
      }
    },
    {
      "id": "h-repay",
      "kind": "repay",
      "position": {
        "x": 2100,
        "y": 200
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "Upstream output"
      }
    },
    {
      "id": "r-exit",
      "kind": "lp-exit",
      "position": {
        "x": 900,
        "y": 320
      },
      "config": {
        "pool": "ETH / USDC",
        "maxSlippage": 50
      }
    },
    {
      "id": "r-cap",
      "kind": "max-allocation",
      "position": {
        "x": 1050,
        "y": 320
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "r-stake",
      "kind": "lp-stake",
      "positionTag": "hedged-lp",
      "position": {
        "x": 1200,
        "y": 320
      },
      "config": {
        "pool": "ETH / USDC",
        "range": "Concentrated",
        "width": 5,
        "allocation": 40,
        "maxSlippage": 50
      }
    },
    {
      "id": "exit",
      "kind": "lp-exit",
      "position": {
        "x": 600,
        "y": 520
      },
      "config": {
        "pool": "ETH / USDC",
        "maxSlippage": 50
      }
    },
    {
      "id": "debt",
      "kind": "debt-check",
      "position": {
        "x": 900,
        "y": 520
      },
      "config": {
        "asset": "ETH",
        "maxSlippage": 50
      }
    },
    {
      "id": "x-buy",
      "kind": "swap",
      "position": {
        "x": 1200,
        "y": 440
      },
      "config": {
        "from": "USDC",
        "to": "ETH",
        "venue": "Uniswap v3",
        "sizeFrom": "Upstream output",
        "allocation": 100,
        "maxSlippage": 50
      }
    },
    {
      "id": "x-repay",
      "kind": "repay",
      "position": {
        "x": 1500,
        "y": 520
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "All owed"
      }
    },
    {
      "id": "x-withdraw",
      "kind": "withdraw-collateral",
      "position": {
        "x": 1800,
        "y": 520
      },
      "config": {
        "market": "ETH against USDC on Aave",
        "sizeFrom": "All of this strategy's",
        "minHealthFactor": 1.1
      }
    },
    {
      "id": "x-sell",
      "kind": "swap",
      "position": {
        "x": 2100,
        "y": 520
      },
      "config": {
        "from": "ETH",
        "to": "USDC",
        "venue": "Uniswap v3",
        "sizeFrom": "Upstream output",
        "allocation": 100,
        "maxSlippage": 50
      }
    },
    {
      "id": "monitor-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "LP Monitor could not read the position"
      }
    },
    {
      "id": "cooldown-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Cooling off after an adverse exit; not reopening"
      }
    },
    {
      "id": "cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the allocation cap; not opening"
      }
    },
    {
      "id": "supply-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Supplying USDC collateral failed"
      }
    },
    {
      "id": "borrow-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Borrowing ETH for the LP failed"
      }
    },
    {
      "id": "stake-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Opening the LP position failed"
      }
    },
    {
      "id": "hedge-slip-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Hedge held: swap slippage over the cap"
      }
    },
    {
      "id": "hedge-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Hedge Check could not read the position"
      }
    },
    {
      "id": "h-borrow-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Borrowing more ETH to hedge failed"
      }
    },
    {
      "id": "h-sell-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Selling borrowed ETH failed"
      }
    },
    {
      "id": "h-supply-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Supplying the sale as collateral failed"
      }
    },
    {
      "id": "h-withdraw-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Withdrawing collateral to buy ETH back failed"
      }
    },
    {
      "id": "h-buy-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buying ETH back failed"
      }
    },
    {
      "id": "h-repay-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Repaying ETH from the hedge failed"
      }
    },
    {
      "id": "r-exit-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exiting the LP to re-range failed"
      }
    },
    {
      "id": "r-cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Re-range held: over the allocation cap; capital left in the treasury"
      }
    },
    {
      "id": "r-stake-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Re-opening the LP at the new range failed"
      }
    },
    {
      "id": "exit-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exiting the LP to close failed"
      }
    },
    {
      "id": "x-buy-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buying ETH to cover the debt failed"
      }
    },
    {
      "id": "x-repay-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Repaying the ETH debt failed; position still levered"
      }
    },
    {
      "id": "x-withdraw-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Withdrawing collateral after repay failed"
      }
    },
    {
      "id": "x-sell-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Selling leftover ETH to USDC failed"
      }
    }
  ],
  "edges": [
    {
      "source": "schedule",
      "target": "rules"
    },
    {
      "source": "rules",
      "target": "monitor",
      "sourceHandle": "primary"
    },
    {
      "source": "rules",
      "target": "exit",
      "sourceHandle": "fallback"
    },
    {
      "source": "monitor",
      "target": "hedge-slip",
      "sourceHandle": "manage"
    },
    {
      "source": "monitor",
      "target": "r-exit",
      "sourceHandle": "rerange"
    },
    {
      "source": "monitor",
      "target": "cooldown",
      "sourceHandle": "none"
    },
    {
      "source": "cooldown",
      "target": "cap",
      "sourceHandle": "passed"
    },
    {
      "source": "cap",
      "target": "supply",
      "sourceHandle": "passed"
    },
    {
      "source": "supply",
      "target": "borrow",
      "sourceHandle": "done"
    },
    {
      "source": "borrow",
      "target": "stake",
      "sourceHandle": "done"
    },
    {
      "source": "hedge-slip",
      "target": "hedge",
      "sourceHandle": "passed"
    },
    {
      "source": "hedge",
      "target": "h-borrow",
      "sourceHandle": "borrow"
    },
    {
      "source": "h-borrow",
      "target": "h-sell",
      "sourceHandle": "done"
    },
    {
      "source": "h-sell",
      "target": "h-supply",
      "sourceHandle": "filled"
    },
    {
      "source": "hedge",
      "target": "h-withdraw",
      "sourceHandle": "repay"
    },
    {
      "source": "h-withdraw",
      "target": "h-buy",
      "sourceHandle": "done"
    },
    {
      "source": "h-buy",
      "target": "h-repay",
      "sourceHandle": "filled"
    },
    {
      "source": "r-exit",
      "target": "r-cap",
      "sourceHandle": "done"
    },
    {
      "source": "r-cap",
      "target": "r-stake",
      "sourceHandle": "passed"
    },
    {
      "source": "r-stake",
      "target": "hedge-slip",
      "sourceHandle": "default"
    },
    {
      "source": "exit",
      "target": "debt",
      "sourceHandle": "done"
    },
    {
      "source": "debt",
      "target": "x-repay",
      "sourceHandle": "covered"
    },
    {
      "source": "debt",
      "target": "x-buy",
      "sourceHandle": "short"
    },
    {
      "source": "debt",
      "target": "x-repay",
      "sourceHandle": "error"
    },
    {
      "source": "x-buy",
      "target": "x-repay",
      "sourceHandle": "filled"
    },
    {
      "source": "x-repay",
      "target": "x-withdraw",
      "sourceHandle": "done"
    },
    {
      "source": "x-withdraw",
      "target": "x-sell",
      "sourceHandle": "done"
    },
    {
      "source": "monitor",
      "target": "monitor-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "cooldown",
      "target": "cooldown-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "cap",
      "target": "cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "supply",
      "target": "supply-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "borrow",
      "target": "borrow-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "stake",
      "target": "stake-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "hedge-slip",
      "target": "hedge-slip-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "hedge",
      "target": "hedge-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "h-borrow",
      "target": "h-borrow-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "h-sell",
      "target": "h-sell-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "h-supply",
      "target": "h-supply-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "h-withdraw",
      "target": "h-withdraw-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "h-buy",
      "target": "h-buy-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "h-repay",
      "target": "h-repay-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "r-exit",
      "target": "r-exit-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "r-cap",
      "target": "r-cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "r-stake",
      "target": "r-stake-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "exit",
      "target": "exit-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "x-buy",
      "target": "x-buy-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "x-repay",
      "target": "x-repay-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "x-withdraw",
      "target": "x-withdraw-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "x-sell",
      "target": "x-sell-failed-journal",
      "sourceHandle": "failed"
    }
  ]
}
```

### Funding Carry (Hyperliquid)

`funding-carry-hl`, hedged-carry. Delta-neutral long spot / short perp on Hyperliquid. Collect funding while the rate clears costs; unwind on flip, margin stress, basis diverge, or venue incident. Leverage defaults to 2x — the position is P&L-neutral, not margin-neutral.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Funding Carry (Hyperliquid)",
  "strategyClass": "hedged-carry",
  "defs": {
    "btcSpot": {
      "type": "instrument",
      "venue": "hyperliquid",
      "symbol": "BTC",
      "instrumentType": "spot"
    },
    "btcPerp": {
      "type": "instrument",
      "venue": "hyperliquid",
      "symbol": "BTC",
      "instrumentType": "perp"
    }
  },
  "nodes": [
    {
      "id": "entry-sched",
      "kind": "schedule",
      "group": "entry",
      "position": {
        "x": 0,
        "y": 40
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "funding",
      "kind": "funding-feed",
      "group": "entry",
      "position": {
        "x": 280,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "lookback": "7 days"
      }
    },
    {
      "id": "fund-signal",
      "kind": "signal-gate",
      "group": "entry",
      "position": {
        "x": 560,
        "y": 40
      },
      "config": {
        "metric": "Funding rate (annualized)",
        "threshold": 10,
        "sustained": 8,
        "confirmation": "None",
        "ceiling": 80,
        "instrumentRef": "btcPerp"
      }
    },
    {
      "id": "ev",
      "kind": "ev-gate",
      "group": "entry",
      "position": {
        "x": 840,
        "y": 40
      },
      "config": {
        "minEdgeMultiple": 3,
        "expectedHold": "7 days",
        "edgeMetric": "Modelled edge"
      }
    },
    {
      "id": "regime",
      "kind": "market-conditions",
      "group": "entry",
      "position": {
        "x": 1120,
        "y": 40
      },
      "config": {
        "metric": "Realized vol vs 1y median",
        "window": "30 days",
        "multiple": 2
      }
    },
    {
      "id": "exposure",
      "kind": "exposure-guard",
      "group": "entry",
      "position": {
        "x": 1400,
        "y": 40
      },
      "config": {
        "maxGross": 80,
        "maxConcurrent": 2
      }
    },
    {
      "id": "margin",
      "kind": "margin-health",
      "group": "entry",
      "position": {
        "x": 1680,
        "y": 40
      },
      "config": {
        "minLiqDistance": 25
      }
    },
    {
      "id": "open-spot",
      "kind": "spot-position",
      "group": "entry",
      "positionTag": "carry-btc",
      "position": {
        "x": 1960,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "side": "Buy",
        "sizeFrom": "Account, one of two legs"
      }
    },
    {
      "id": "open-perp",
      "kind": "perp-position",
      "group": "entry",
      "positionTag": "carry-btc",
      "position": {
        "x": 2240,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "mon-sched",
      "kind": "schedule",
      "group": "monitoring",
      "position": {
        "x": 0,
        "y": 480
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "legs",
      "kind": "leg-check",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 280,
        "y": 480
      },
      "config": {
        "positionTag": "carry-btc",
        "maxOneLegOpen": 5
      }
    },
    {
      "id": "flat-perp",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 560,
        "y": 340
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Leg Check",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "flat-spot",
      "kind": "spot-position",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 560,
        "y": 220
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "side": "Sell",
        "sizeFrom": "Leg Check"
      }
    },
    {
      "id": "sync",
      "kind": "position-sync",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 560,
        "y": 480
      },
      "config": {
        "positionTag": "carry-btc",
        "venue": "Hyperliquid"
      }
    },
    {
      "id": "health",
      "kind": "position-health",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 840,
        "y": 480
      },
      "config": {
        "positionTag": "carry-btc",
        "profile": "Carry"
      }
    },
    {
      "id": "close-perp",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 840,
        "y": 620
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "close-spot",
      "kind": "spot-position",
      "group": "monitoring",
      "positionTag": "carry-btc",
      "position": {
        "x": 1120,
        "y": 620
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "side": "Sell",
        "sizeFrom": "Account"
      }
    },
    {
      "id": "exit-cd",
      "kind": "cooldown",
      "group": "monitoring",
      "position": {
        "x": 1400,
        "y": 620
      },
      "config": {
        "hours": 48,
        "scope": "Per instrument",
        "condition": "After adverse exit only"
      }
    },
    {
      "id": "kill",
      "kind": "kill-switch",
      "group": "monitoring",
      "position": {
        "x": 0,
        "y": 780
      },
      "config": {
        "maxDrawdown": 8,
        "scope": "Global"
      }
    },
    {
      "id": "funding-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Funding feed unavailable; not entering"
      }
    },
    {
      "id": "fund-signal-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Funding under the entry threshold"
      }
    },
    {
      "id": "ev-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Edge does not clear 3x round-trip cost"
      }
    },
    {
      "id": "regime-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Volatility over 2x its 1y median; not entering"
      }
    },
    {
      "id": "exposure-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exposure limit reached"
      }
    },
    {
      "id": "margin-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Liquidation distance too thin to open"
      }
    },
    {
      "id": "open-perp-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Opened: entry funding and cost snapshot"
      }
    },
    {
      "id": "flat-perp-default-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Carry left with one leg; perp flattened",
        "channel": "Telegram",
        "severity": "Critical"
      }
    },
    {
      "id": "flat-spot-default-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Carry left with one leg; spot sold",
        "channel": "Telegram",
        "severity": "Critical"
      }
    },
    {
      "id": "close-spot-default-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Carry closed on a health trip",
        "channel": "Telegram",
        "severity": "Critical"
      }
    },
    {
      "id": "exit-cd-passed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Closed: realised vs modelled funding"
      }
    },
    {
      "id": "exit-cd-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Closed during a cooldown: realised vs modelled funding"
      }
    },
    {
      "id": "kill-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Kill switch tripped: drawdown over 8%",
        "channel": "Telegram",
        "severity": "Critical"
      }
    }
  ],
  "edges": [
    {
      "source": "entry-sched",
      "target": "funding"
    },
    {
      "source": "funding",
      "target": "fund-signal",
      "sourceHandle": "data"
    },
    {
      "source": "fund-signal",
      "target": "ev",
      "sourceHandle": "primary"
    },
    {
      "source": "ev",
      "target": "regime",
      "sourceHandle": "passed"
    },
    {
      "source": "regime",
      "target": "exposure",
      "sourceHandle": "primary"
    },
    {
      "source": "exposure",
      "target": "margin",
      "sourceHandle": "passed"
    },
    {
      "source": "margin",
      "target": "open-spot",
      "sourceHandle": "passed"
    },
    {
      "source": "open-spot",
      "target": "open-perp",
      "sourceHandle": "default"
    },
    {
      "source": "mon-sched",
      "target": "legs"
    },
    {
      "source": "legs",
      "target": "sync",
      "sourceHandle": "passed"
    },
    {
      "source": "legs",
      "target": "flat-perp",
      "sourceHandle": "flatten"
    },
    {
      "source": "legs",
      "target": "flat-spot",
      "sourceHandle": "flatten-spot"
    },
    {
      "source": "sync",
      "target": "health"
    },
    {
      "source": "health",
      "target": "close-perp",
      "sourceHandle": "fallback"
    },
    {
      "source": "close-perp",
      "target": "close-spot",
      "sourceHandle": "default"
    },
    {
      "source": "close-perp",
      "target": "close-spot",
      "sourceHandle": "none"
    },
    {
      "source": "close-spot",
      "target": "exit-cd",
      "sourceHandle": "default"
    },
    {
      "source": "funding",
      "target": "funding-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "fund-signal",
      "target": "fund-signal-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "ev",
      "target": "ev-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "regime",
      "target": "regime-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "exposure",
      "target": "exposure-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "margin",
      "target": "margin-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "open-perp",
      "target": "open-perp-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "flat-perp",
      "target": "flat-perp-default-alert",
      "sourceHandle": "default"
    },
    {
      "source": "flat-spot",
      "target": "flat-spot-default-alert",
      "sourceHandle": "default"
    },
    {
      "source": "close-spot",
      "target": "close-spot-default-alert",
      "sourceHandle": "default"
    },
    {
      "source": "exit-cd",
      "target": "exit-cd-passed-journal",
      "sourceHandle": "passed"
    },
    {
      "source": "exit-cd",
      "target": "exit-cd-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "kill",
      "target": "kill-blocked-alert",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Pair Trade (Hyperliquid)

`pair-trade-hl`, market-neutral. Scan Hyperliquid perps for a correlated pair, then trade the stretch: short the rich, long the cheap in a hedge ratio; close on convergence or tripwires. Entry statistics are frozen at fill.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Pair Trade (Hyperliquid)",
  "strategyClass": "market-neutral",
  "defs": {
    "pairA": {
      "type": "instrument-pair",
      "legA": {
        "venue": "hyperliquid",
        "symbol": "X",
        "instrumentType": "perp"
      },
      "legB": {
        "venue": "hyperliquid",
        "symbol": "Y",
        "instrumentType": "perp"
      },
      "hedgeRatioMode": "Rolling OLS 60d",
      "sizingMode": "Vol-balanced"
    }
  },
  "nodes": [
    {
      "id": "screen-sched",
      "kind": "schedule",
      "group": "screening",
      "position": {
        "x": 0,
        "y": 40
      },
      "config": {
        "interval": "Weekly"
      }
    },
    {
      "id": "screener",
      "kind": "pair-screener",
      "group": "screening",
      "position": {
        "x": 280,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "minCorrelation": 0.7,
        "maxHalfLifeDays": 30,
        "minHistoryDays": 90,
        "maxZ": 3.5,
        "topK": 5,
        "bindTo": "pairA"
      }
    },
    {
      "id": "stats",
      "kind": "pair-stats",
      "group": "screening",
      "position": {
        "x": 560,
        "y": 40
      },
      "config": {
        "pairRef": "pairA",
        "minHistoryDays": 90
      }
    },
    {
      "id": "z-signal",
      "kind": "signal-gate",
      "group": "entry",
      "position": {
        "x": 0,
        "y": 320
      },
      "config": {
        "metric": "Spread z-score",
        "threshold": 2.25,
        "sustained": 2,
        "confirmation": "Retrace",
        "ceiling": 3.5,
        "instrumentRef": "pairA"
      }
    },
    {
      "id": "quarantine",
      "kind": "quarantine-guard",
      "group": "entry",
      "position": {
        "x": 280,
        "y": 320
      },
      "config": {
        "pairRef": "pairA"
      }
    },
    {
      "id": "ev",
      "kind": "ev-gate",
      "group": "entry",
      "position": {
        "x": 560,
        "y": 320
      },
      "config": {
        "minEdgeMultiple": 2.5,
        "expectedHold": "14 days",
        "edgeMetric": "Modelled edge"
      }
    },
    {
      "id": "exposure",
      "kind": "exposure-guard",
      "group": "entry",
      "position": {
        "x": 840,
        "y": 320
      },
      "config": {
        "maxGross": 60,
        "maxConcurrent": 4
      }
    },
    {
      "id": "margin",
      "kind": "margin-health",
      "group": "entry",
      "position": {
        "x": 1120,
        "y": 320
      },
      "config": {
        "minLiqDistance": 30
      }
    },
    {
      "id": "leg-a",
      "kind": "perp-position",
      "group": "entry",
      "positionTag": "pair-xy",
      "position": {
        "x": 1400,
        "y": 320
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "Leg A",
        "side": "Against the spread",
        "sizeFrom": "Account, one of two legs",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "leg-b",
      "kind": "perp-position",
      "group": "entry",
      "positionTag": "pair-xy",
      "position": {
        "x": 1680,
        "y": 320
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "Leg B",
        "side": "Against the spread",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "mon-sched",
      "kind": "schedule",
      "group": "monitoring",
      "position": {
        "x": 0,
        "y": 640
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "legs",
      "kind": "leg-check",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 280,
        "y": 640
      },
      "config": {
        "positionTag": "pair-xy",
        "maxOneLegOpen": 5
      }
    },
    {
      "id": "flatten",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 560,
        "y": 500
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Leg Check",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "sync",
      "kind": "position-sync",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 560,
        "y": 640
      },
      "config": {
        "positionTag": "pair-xy",
        "venue": "Hyperliquid"
      }
    },
    {
      "id": "health",
      "kind": "position-health",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 840,
        "y": 640
      },
      "config": {
        "positionTag": "pair-xy",
        "profile": "Pair"
      }
    },
    {
      "id": "close-a",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 840,
        "y": 780
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "Leg A",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "close-b",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "pair-xy",
      "position": {
        "x": 1120,
        "y": 780
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "Leg B",
        "side": "Short",
        "sizeFrom": "Account",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "exit-cd",
      "kind": "cooldown",
      "group": "monitoring",
      "position": {
        "x": 1400,
        "y": 780
      },
      "config": {
        "hours": 720,
        "scope": "Per position tag",
        "condition": "After adverse exit only"
      }
    },
    {
      "id": "screener-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "No pair cleared the screen"
      }
    },
    {
      "id": "stats-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Pair statistics no longer hold"
      }
    },
    {
      "id": "z-signal-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Spread not stretched far enough to enter"
      }
    },
    {
      "id": "quarantine-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Pair cooling off after an adverse exit"
      }
    },
    {
      "id": "ev-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Edge does not clear 2.5x round-trip cost"
      }
    },
    {
      "id": "exposure-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exposure limit reached"
      }
    },
    {
      "id": "margin-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Liquidation distance too thin to open"
      }
    },
    {
      "id": "leg-b-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Entry z, beta and sigma snapshot frozen"
      }
    },
    {
      "id": "flatten-default-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Pair left with one leg; flattened",
        "channel": "Telegram",
        "severity": "Critical"
      }
    },
    {
      "id": "exit-cd-passed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Pair closed"
      }
    },
    {
      "id": "exit-cd-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Pair closed during a cooldown"
      }
    }
  ],
  "edges": [
    {
      "source": "screen-sched",
      "target": "screener"
    },
    {
      "source": "screener",
      "target": "stats",
      "sourceHandle": "primary"
    },
    {
      "source": "stats",
      "target": "z-signal",
      "sourceHandle": "primary"
    },
    {
      "source": "z-signal",
      "target": "quarantine",
      "sourceHandle": "primary"
    },
    {
      "source": "quarantine",
      "target": "ev",
      "sourceHandle": "passed"
    },
    {
      "source": "ev",
      "target": "exposure",
      "sourceHandle": "passed"
    },
    {
      "source": "exposure",
      "target": "margin",
      "sourceHandle": "passed"
    },
    {
      "source": "margin",
      "target": "leg-a",
      "sourceHandle": "passed"
    },
    {
      "source": "leg-a",
      "target": "leg-b",
      "sourceHandle": "default"
    },
    {
      "source": "mon-sched",
      "target": "legs"
    },
    {
      "source": "legs",
      "target": "sync",
      "sourceHandle": "passed"
    },
    {
      "source": "legs",
      "target": "flatten",
      "sourceHandle": "flatten"
    },
    {
      "source": "sync",
      "target": "health"
    },
    {
      "source": "health",
      "target": "close-a",
      "sourceHandle": "fallback"
    },
    {
      "source": "close-a",
      "target": "close-b",
      "sourceHandle": "default"
    },
    {
      "source": "close-a",
      "target": "close-b",
      "sourceHandle": "none"
    },
    {
      "source": "close-b",
      "target": "exit-cd",
      "sourceHandle": "default"
    },
    {
      "source": "screener",
      "target": "screener-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "stats",
      "target": "stats-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "z-signal",
      "target": "z-signal-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "quarantine",
      "target": "quarantine-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "ev",
      "target": "ev-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "exposure",
      "target": "exposure-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "margin",
      "target": "margin-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "leg-b",
      "target": "leg-b-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "flatten",
      "target": "flatten-default-alert",
      "sourceHandle": "default"
    },
    {
      "source": "exit-cd",
      "target": "exit-cd-passed-journal",
      "sourceHandle": "passed"
    },
    {
      "source": "exit-cd",
      "target": "exit-cd-blocked-journal",
      "sourceHandle": "blocked"
    }
  ]
}
```

### Passthrough Agent

`passthrough-agent`, directional. Run your own agent's plan through ampli. Your agent posts prepared calls (to, data, value) and Hyperliquid actions to this deployment's webhook address with its key; ampli checks every call and action against the treasury policy, refuses the whole signal if any one fails, and signs the rest from the wallet. Use it when your agent already decides what to do and only needs ampli to hold the keys and enforce the policy. The kill switch stops everything past 15% drawdown.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "Passthrough Agent",
  "strategyClass": "directional",
  "nodes": [
    {
      "id": "hook",
      "kind": "webhook",
      "position": {
        "x": 0,
        "y": 200
      },
      "config": {
        "maxAge": "5 min",
        "maxSizeUsd": 0
      }
    },
    {
      "id": "run",
      "kind": "raw-calls",
      "position": {
        "x": 300,
        "y": 200
      },
      "config": {
        "maxCalls": 16,
        "allowHyperliquid": true
      }
    },
    {
      "id": "kill",
      "kind": "kill-switch",
      "position": {
        "x": 0,
        "y": 420
      },
      "config": {
        "maxDrawdown": 15,
        "scope": "Global"
      }
    },
    {
      "id": "run-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Ran the agent's calls"
      }
    },
    {
      "id": "run-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "The agent's signal was refused by the policy",
        "channel": "Telegram",
        "severity": "Warning"
      }
    },
    {
      "id": "run-none-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Signal carried nothing to run"
      }
    },
    {
      "id": "kill-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Kill switch tripped: drawdown over 15%",
        "channel": "Telegram",
        "severity": "Critical"
      }
    }
  ],
  "edges": [
    {
      "source": "hook",
      "target": "run",
      "sourceHandle": "execute"
    },
    {
      "source": "run",
      "target": "run-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "run",
      "target": "run-blocked-alert",
      "sourceHandle": "blocked"
    },
    {
      "source": "run",
      "target": "run-none-journal",
      "sourceHandle": "none"
    },
    {
      "source": "kill",
      "target": "kill-blocked-alert",
      "sourceHandle": "blocked"
    }
  ]
}
```

### 2Factor Senior Yield

`twofactor-senior-yield`, treasury-yield. Deposit idle USDC into the 2Factor USD vault (perpSr on Base) while its yield clears 5%. The deposit is sized down until the yield after it still clears the floor. Redeem when the yield falls below 3% or when an approved withdrawal needs the cash. Phase 1: USDC treasuries only.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "2Factor Senior Yield",
  "strategyClass": "treasury-yield",
  "nodes": [
    {
      "id": "entry-sched",
      "kind": "schedule",
      "group": "entry",
      "position": {
        "x": 0,
        "y": 40
      },
      "config": {
        "interval": "Every hour"
      }
    },
    {
      "id": "feed",
      "kind": "twofactor-feed",
      "group": "entry",
      "position": {
        "x": 280,
        "y": 40
      },
      "config": {
        "view": "Senior",
        "entrySize": 50,
        "maxExitFee": 0.5,
        "hedgeSymbol": "BTC"
      }
    },
    {
      "id": "yield-signal",
      "kind": "signal-gate",
      "group": "entry",
      "position": {
        "x": 560,
        "y": 40
      },
      "config": {
        "metric": "2Factor vault yield (annualized)",
        "threshold": 5,
        "sustained": 3,
        "confirmation": "None",
        "ceiling": 80,
        "instrumentRef": ""
      }
    },
    {
      "id": "buffer",
      "kind": "liquidity-buffer",
      "group": "entry",
      "position": {
        "x": 840,
        "y": 40
      },
      "config": {
        "target": 1000,
        "asset": "USDC"
      }
    },
    {
      "id": "cap",
      "kind": "max-allocation",
      "group": "entry",
      "position": {
        "x": 1120,
        "y": 40
      },
      "config": {
        "cap": 50,
        "scope": "Single venue"
      }
    },
    {
      "id": "slip",
      "kind": "slippage-cap",
      "group": "entry",
      "position": {
        "x": 1400,
        "y": 40
      },
      "config": {
        "cap": 50
      }
    },
    {
      "id": "entry-cd",
      "kind": "cooldown",
      "group": "entry",
      "position": {
        "x": 1680,
        "y": 40
      },
      "config": {
        "hours": 6,
        "scope": "This path",
        "condition": "Always"
      }
    },
    {
      "id": "deposit",
      "kind": "twofactor-vault",
      "group": "entry",
      "position": {
        "x": 1960,
        "y": 40
      },
      "config": {
        "mode": "Deposit",
        "allocation": 50,
        "enterAbove": 5,
        "exitBelow": 3,
        "slippageBps": 50
      }
    },
    {
      "id": "exit-sched",
      "kind": "schedule",
      "group": "exit",
      "position": {
        "x": 0,
        "y": 480
      },
      "config": {
        "interval": "Every 6 hours"
      }
    },
    {
      "id": "exit-feed",
      "kind": "twofactor-feed",
      "group": "exit",
      "position": {
        "x": 280,
        "y": 480
      },
      "config": {
        "view": "Senior",
        "entrySize": 25,
        "maxExitFee": 0.5,
        "hedgeSymbol": "BTC"
      }
    },
    {
      "id": "exit-slip",
      "kind": "slippage-cap",
      "group": "exit",
      "position": {
        "x": 560,
        "y": 480
      },
      "config": {
        "cap": 50
      }
    },
    {
      "id": "redeem",
      "kind": "twofactor-vault",
      "group": "exit",
      "position": {
        "x": 840,
        "y": 480
      },
      "config": {
        "mode": "Redeem",
        "allocation": 100,
        "enterAbove": 5,
        "exitBelow": 3,
        "slippageBps": 50
      }
    },
    {
      "id": "wd-trigger",
      "kind": "withdrawal-approved",
      "group": "withdrawal",
      "position": {
        "x": 0,
        "y": 720
      },
      "config": {
        "minAmount": 0
      }
    },
    {
      "id": "wd-redeem",
      "kind": "twofactor-vault",
      "group": "withdrawal",
      "position": {
        "x": 280,
        "y": 720
      },
      "config": {
        "mode": "Redeem",
        "allocation": 100,
        "enterAbove": 5,
        "exitBelow": 0,
        "slippageBps": 50
      }
    },
    {
      "id": "payout",
      "kind": "payout",
      "group": "withdrawal",
      "position": {
        "x": 560,
        "y": 720
      },
      "config": {}
    },
    {
      "id": "feed-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "2Factor feed unavailable; not depositing"
      }
    },
    {
      "id": "yield-signal-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Vault yield under 5%; not depositing"
      }
    },
    {
      "id": "buffer-breach-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buffer under target; nothing deposited"
      }
    },
    {
      "id": "cap-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Over the single-venue cap; not depositing"
      }
    },
    {
      "id": "slip-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Deposit slippage over the cap"
      }
    },
    {
      "id": "entry-cd-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Deposited within the last 6 hours"
      }
    },
    {
      "id": "deposit-out-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Deposited into the 2Factor vault"
      }
    },
    {
      "id": "exit-feed-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "2Factor feed unavailable; not redeeming"
      }
    },
    {
      "id": "exit-slip-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Redeem slippage over the cap; holding"
      }
    },
    {
      "id": "redeem-out-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Redeemed from the 2Factor vault"
      }
    }
  ],
  "edges": [
    {
      "source": "entry-sched",
      "target": "feed"
    },
    {
      "source": "feed",
      "target": "yield-signal",
      "sourceHandle": "data"
    },
    {
      "source": "yield-signal",
      "target": "buffer",
      "sourceHandle": "primary"
    },
    {
      "source": "buffer",
      "target": "cap",
      "sourceHandle": "surplus"
    },
    {
      "source": "cap",
      "target": "slip",
      "sourceHandle": "passed"
    },
    {
      "source": "slip",
      "target": "entry-cd",
      "sourceHandle": "passed"
    },
    {
      "source": "entry-cd",
      "target": "deposit",
      "sourceHandle": "passed"
    },
    {
      "source": "exit-sched",
      "target": "exit-feed"
    },
    {
      "source": "exit-feed",
      "target": "exit-slip",
      "sourceHandle": "data"
    },
    {
      "source": "exit-slip",
      "target": "redeem",
      "sourceHandle": "passed"
    },
    {
      "source": "wd-trigger",
      "target": "wd-redeem"
    },
    {
      "source": "wd-redeem",
      "target": "payout"
    },
    {
      "source": "feed",
      "target": "feed-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "yield-signal",
      "target": "yield-signal-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "buffer",
      "target": "buffer-breach-journal",
      "sourceHandle": "breach"
    },
    {
      "source": "cap",
      "target": "cap-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "slip",
      "target": "slip-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "entry-cd",
      "target": "entry-cd-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "deposit",
      "target": "deposit-out-journal"
    },
    {
      "source": "exit-feed",
      "target": "exit-feed-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "exit-slip",
      "target": "exit-slip-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "redeem",
      "target": "redeem-out-journal"
    }
  ]
}
```

### 2Factor Junior Funding Arb

`twofactor-junior-arb`, hedged-carry. A Swap buys cbBTC with USDC on Orbs dTWAP; once it fills, 2Factor Mint mints perpJr and a Perp Position shorts the BTC exposure on Hyperliquid after the mint is mined. Earns the Hyperliquid funding on the levered short minus the perpJr funding. While the carry is open, a 2Factor Hedge Check resizes the short inside a band. 2Factor Carry Check exits when the spread falls below 3%: 2Factor Redeem returns the cbBTC, a reduce-only Perp Position closes the matching short, and a second dTWAP Swap sells the cbBTC for USDC. A 2Factor Cleanup closes any short left behind and hands on cbBTC that was never minted. Phase 1: USDC treasuries only.

```json
{
  "version": 2,
  "catalogVersion": "2026.10.5.1",
  "name": "2Factor Junior Funding Arb",
  "strategyClass": "hedged-carry",
  "defs": {
    "btcSpot": {
      "type": "instrument",
      "venue": "hyperliquid",
      "symbol": "BTC",
      "instrumentType": "spot"
    },
    "btcPerp": {
      "type": "instrument",
      "venue": "hyperliquid",
      "symbol": "BTC",
      "instrumentType": "perp"
    }
  },
  "nodes": [
    {
      "id": "entry-sched",
      "kind": "schedule",
      "group": "entry",
      "position": {
        "x": 0,
        "y": 40
      },
      "config": {
        "interval": "Every hour"
      }
    },
    {
      "id": "hl-funding",
      "kind": "funding-feed",
      "group": "entry",
      "position": {
        "x": 280,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "lookback": "7 days"
      }
    },
    {
      "id": "feed",
      "kind": "twofactor-feed",
      "group": "entry",
      "position": {
        "x": 560,
        "y": 40
      },
      "config": {
        "view": "Junior",
        "entrySize": 10,
        "maxExitFee": 0.5,
        "hedgeSymbol": "BTC"
      }
    },
    {
      "id": "spread-signal",
      "kind": "signal-gate",
      "group": "entry",
      "position": {
        "x": 840,
        "y": 40
      },
      "config": {
        "metric": "2Factor Jr spread (annualized)",
        "threshold": 5,
        "sustained": 3,
        "confirmation": "None",
        "ceiling": 80,
        "instrumentRef": "btcPerp"
      }
    },
    {
      "id": "ev",
      "kind": "ev-gate",
      "group": "entry",
      "position": {
        "x": 1120,
        "y": 40
      },
      "config": {
        "minEdgeMultiple": 2,
        "expectedHold": "30 days",
        "edgeMetric": "2Factor Jr spread after entry (annualized)"
      }
    },
    {
      "id": "exposure",
      "kind": "exposure-guard",
      "group": "entry",
      "position": {
        "x": 1400,
        "y": 40
      },
      "config": {
        "maxGross": 80,
        "maxConcurrent": 1
      }
    },
    {
      "id": "margin",
      "kind": "margin-health",
      "group": "entry",
      "position": {
        "x": 1680,
        "y": 40
      },
      "config": {
        "minLiqDistance": 25
      }
    },
    {
      "id": "buy",
      "kind": "swap",
      "group": "entry",
      "position": {
        "x": 1960,
        "y": 40
      },
      "config": {
        "from": "USDC",
        "to": "cbBTC",
        "venue": "Orbs dTWAP",
        "sizeFrom": "Share of balance",
        "allocation": 10,
        "maxSlippage": 50
      }
    },
    {
      "id": "mint",
      "kind": "twofactor-mint",
      "group": "entry",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 2240,
        "y": 40
      },
      "config": {
        "positionTag": "2f-jr-btc",
        "allocation": 100,
        "enterAbove": 5,
        "hedgeLeverage": 2,
        "slippageBps": 10
      }
    },
    {
      "id": "short",
      "kind": "perp-position",
      "group": "entry",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 2520,
        "y": 40
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "mon-sched",
      "kind": "schedule",
      "group": "monitoring",
      "position": {
        "x": 0,
        "y": 480
      },
      "config": {
        "interval": "Every 15 min"
      }
    },
    {
      "id": "mon-funding",
      "kind": "funding-feed",
      "group": "monitoring",
      "position": {
        "x": 280,
        "y": 480
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "lookback": "7 days"
      }
    },
    {
      "id": "mon-feed",
      "kind": "twofactor-feed",
      "group": "monitoring",
      "position": {
        "x": 560,
        "y": 480
      },
      "config": {
        "view": "Junior",
        "entrySize": 25,
        "maxExitFee": 0.5,
        "hedgeSymbol": "BTC"
      }
    },
    {
      "id": "mon-guard",
      "kind": "slippage-cap",
      "group": "monitoring",
      "position": {
        "x": 840,
        "y": 480
      },
      "config": {
        "cap": 10
      }
    },
    {
      "id": "check",
      "kind": "twofactor-check",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1120,
        "y": 480
      },
      "config": {
        "positionTag": "2f-jr-btc",
        "exitBelow": 3,
        "allocation": 100,
        "releaseIdleAfter": 60
      }
    },
    {
      "id": "hedge",
      "kind": "twofactor-hedge-check",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1260,
        "y": 300
      },
      "config": {
        "positionTag": "2f-jr-btc",
        "band": 2,
        "hedgeLeverage": 2
      }
    },
    {
      "id": "cleanup",
      "kind": "twofactor-cleanup",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1260,
        "y": 780
      },
      "config": {
        "positionTag": "2f-jr-btc"
      }
    },
    {
      "id": "add",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1400,
        "y": 240
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": false
      }
    },
    {
      "id": "trim",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1400,
        "y": 360
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "redeem",
      "kind": "twofactor-redeem",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1400,
        "y": 620
      },
      "config": {
        "positionTag": "2f-jr-btc",
        "maxExitFee": 0.5,
        "slippageBps": 10
      }
    },
    {
      "id": "close-short",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1680,
        "y": 620
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "close-rest",
      "kind": "perp-position",
      "group": "monitoring",
      "positionTag": "2f-jr-btc",
      "position": {
        "x": 1680,
        "y": 780
      },
      "config": {
        "venue": "Hyperliquid",
        "symbol": "BTC",
        "symbolFrom": "Symbol",
        "pairLeg": "None",
        "side": "Short",
        "sizeFrom": "Match upstream leg",
        "leverageCap": 2,
        "marginBuffer": 50,
        "reduceOnly": true
      }
    },
    {
      "id": "sell",
      "kind": "swap",
      "group": "monitoring",
      "position": {
        "x": 1960,
        "y": 620
      },
      "config": {
        "from": "cbBTC",
        "to": "USDC",
        "venue": "Orbs dTWAP",
        "sizeFrom": "Upstream output",
        "allocation": 100,
        "maxSlippage": 50
      }
    },
    {
      "id": "kill",
      "kind": "kill-switch",
      "group": "monitoring",
      "position": {
        "x": 0,
        "y": 1100
      },
      "config": {
        "maxDrawdown": 8,
        "scope": "Global"
      }
    },
    {
      "id": "hl-funding-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Hyperliquid funding feed unavailable; not entering"
      }
    },
    {
      "id": "feed-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "2Factor feed unavailable; not entering"
      }
    },
    {
      "id": "spread-signal-fallback-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Junior spread under 5%; not entering"
      }
    },
    {
      "id": "ev-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Edge does not clear 2x round-trip cost"
      }
    },
    {
      "id": "exposure-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Exposure limit reached"
      }
    },
    {
      "id": "margin-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Liquidation distance too thin to open"
      }
    },
    {
      "id": "buy-failed-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Buying cbBTC failed; nothing minted"
      }
    },
    {
      "id": "short-default-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Carry opened"
      }
    },
    {
      "id": "mon-funding-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Hyperliquid funding feed unavailable; not managing"
      }
    },
    {
      "id": "mon-feed-error-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "2Factor feed unavailable; not managing"
      }
    },
    {
      "id": "mon-guard-blocked-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Hedge slippage over the cap; holding"
      }
    },
    {
      "id": "sell-pending-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Selling cbBTC for USDC: order still filling"
      }
    },
    {
      "id": "sell-filled-journal",
      "kind": "journal",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "label": "Carry closed; cbBTC sold for USDC"
      }
    },
    {
      "id": "sell-failed-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Selling cbBTC for USDC failed after closing the short",
        "channel": "Telegram",
        "severity": "Critical"
      }
    },
    {
      "id": "kill-blocked-alert",
      "kind": "notify",
      "position": {
        "x": 0,
        "y": 0
      },
      "config": {
        "message": "Kill switch tripped: drawdown over 8%",
        "channel": "Telegram",
        "severity": "Critical"
      }
    }
  ],
  "edges": [
    {
      "source": "entry-sched",
      "target": "hl-funding"
    },
    {
      "source": "hl-funding",
      "target": "feed",
      "sourceHandle": "data"
    },
    {
      "source": "feed",
      "target": "spread-signal",
      "sourceHandle": "data"
    },
    {
      "source": "spread-signal",
      "target": "ev",
      "sourceHandle": "primary"
    },
    {
      "source": "ev",
      "target": "exposure",
      "sourceHandle": "passed"
    },
    {
      "source": "exposure",
      "target": "margin",
      "sourceHandle": "passed"
    },
    {
      "source": "margin",
      "target": "buy",
      "sourceHandle": "passed"
    },
    {
      "source": "buy",
      "target": "mint",
      "sourceHandle": "filled"
    },
    {
      "source": "mint",
      "target": "short",
      "sourceHandle": "default"
    },
    {
      "source": "mon-sched",
      "target": "mon-funding"
    },
    {
      "source": "mon-funding",
      "target": "mon-feed",
      "sourceHandle": "data"
    },
    {
      "source": "mon-feed",
      "target": "mon-guard",
      "sourceHandle": "data"
    },
    {
      "source": "mon-guard",
      "target": "check",
      "sourceHandle": "passed"
    },
    {
      "source": "check",
      "target": "hedge",
      "sourceHandle": "passed"
    },
    {
      "source": "check",
      "target": "redeem",
      "sourceHandle": "exit"
    },
    {
      "source": "check",
      "target": "cleanup",
      "sourceHandle": "cleanup"
    },
    {
      "source": "hedge",
      "target": "add",
      "sourceHandle": "resize"
    },
    {
      "source": "hedge",
      "target": "trim",
      "sourceHandle": "reduce"
    },
    {
      "source": "cleanup",
      "target": "close-rest",
      "sourceHandle": "flatten"
    },
    {
      "source": "cleanup",
      "target": "sell",
      "sourceHandle": "release"
    },
    {
      "source": "redeem",
      "target": "close-short",
      "sourceHandle": "default"
    },
    {
      "source": "close-short",
      "target": "sell",
      "sourceHandle": "default"
    },
    {
      "source": "close-short",
      "target": "sell",
      "sourceHandle": "none"
    },
    {
      "source": "close-rest",
      "target": "sell",
      "sourceHandle": "default"
    },
    {
      "source": "close-rest",
      "target": "sell",
      "sourceHandle": "none"
    },
    {
      "source": "hl-funding",
      "target": "hl-funding-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "feed",
      "target": "feed-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "spread-signal",
      "target": "spread-signal-fallback-journal",
      "sourceHandle": "fallback"
    },
    {
      "source": "ev",
      "target": "ev-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "exposure",
      "target": "exposure-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "margin",
      "target": "margin-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "buy",
      "target": "buy-failed-journal",
      "sourceHandle": "failed"
    },
    {
      "source": "short",
      "target": "short-default-journal",
      "sourceHandle": "default"
    },
    {
      "source": "mon-funding",
      "target": "mon-funding-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "mon-feed",
      "target": "mon-feed-error-journal",
      "sourceHandle": "error"
    },
    {
      "source": "mon-guard",
      "target": "mon-guard-blocked-journal",
      "sourceHandle": "blocked"
    },
    {
      "source": "sell",
      "target": "sell-pending-journal",
      "sourceHandle": "pending"
    },
    {
      "source": "sell",
      "target": "sell-filled-journal",
      "sourceHandle": "filled"
    },
    {
      "source": "sell",
      "target": "sell-failed-alert",
      "sourceHandle": "failed"
    },
    {
      "source": "kill",
      "target": "kill-blocked-alert",
      "sourceHandle": "blocked"
    }
  ]
}
```

## Signals

Each example reads the address from `AMPLI_WEBHOOK_URL` and the API key
from `AMPLI_WEBHOOK_KEY`.

### curl

```bash
# Open or flip long ETH at half the block's size
curl -sS -X POST "$AMPLI_WEBHOOK_URL" -H "Authorization: Bearer $AMPLI_WEBHOOK_KEY" -H 'Content-Type: application/json' \
  -d '{"action":"buy","symbol":"ETH","size_pct":50,"id":"eth-buy-1"}'

# Short Tesla on Hyperliquid's xyz dex
curl -sS -X POST "$AMPLI_WEBHOOK_URL" -H "Authorization: Bearer $AMPLI_WEBHOOK_KEY" -H 'Content-Type: application/json' \
  -d '{"action":"sell","symbol":"xyz:TSLA","id":"tsla-sell-1"}'

# Buy a Base token by address (it must be on the policy whitelist)
curl -sS -X POST "$AMPLI_WEBHOOK_URL" -H "Authorization: Bearer $AMPLI_WEBHOOK_KEY" -H 'Content-Type: application/json' \
  -d '{"action":"buy","symbol":"0x532f27101965dd16442e59d40670faf5ebb142e4","id":"brett-buy-1"}'

# Plain text
curl -sS -X POST "$AMPLI_WEBHOOK_URL" -H "Authorization: Bearer $AMPLI_WEBHOOK_KEY" -H 'Content-Type: text/plain' --data-raw 'exit SOL'

# Execute a prepared plan (Passthrough Agent): approve USDC, then a $500 ETH perp long
curl -sS -X POST "$AMPLI_WEBHOOK_URL" -H "Authorization: Bearer $AMPLI_WEBHOOK_KEY" -H 'Content-Type: application/json' \
  -d '{"action":"execute","id":"plan-42","calls":[{"to":"0x833589fcd6edb6e08f4c7c32d4f71b54bda02913","data":"0x095ea7b3...","value":"0"}],"hlActions":[{"type":"order","orders":[{"coin":"ETH","side":"buy","notionalUsd":500,"orderType":"market","reduceOnly":false,"market":"perp"}]}]}'
```

### TradingView

TradingView cannot set headers, so the key goes in the message. Paste the
address into the alert's Webhook URL field. A strategy alert carries its
own direction; use this message:

```json
{"key":"<your API key>","action":"{{strategy.order.action}}","market_position":"{{strategy.market_position}}","ticker":"{{ticker}}","price":{{close}},"id":"{{timenow}}"}
```

An indicator alert has no direction, so make one alert per action with
the action written in:

```json
{"key":"<your API key>","action":"buy","ticker":"{{ticker}}","price":{{close}},"id":"{{ticker}}-{{timenow}}"}
```

A chart on `NASDAQ:TSLA` sends `TSLA`, which trades the most-traded HIP-3
listing. To pin one dex, write the symbol in: `"symbol":"xyz:TSLA"`.

### Python

Retries only what is safe to retry, with the same id each time:

```python
import json, os, time, urllib.error, urllib.request

def send_signal(signal: dict) -> tuple[int, str]:
    url = os.environ["AMPLI_WEBHOOK_URL"]
    headers = {
        "Authorization": f"Bearer {os.environ['AMPLI_WEBHOOK_KEY']}",
        "Content-Type": "application/json",
    }
    body = json.dumps(signal).encode()
    for attempt in range(5):
        request = urllib.request.Request(url, data=body, method="POST", headers=headers)
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return response.status, response.read().decode()
        except urllib.error.HTTPError as error:
            if error.code != 429:
                return error.code, error.read().decode()
        except urllib.error.URLError:
            pass
        time.sleep(2 ** attempt)
    raise RuntimeError("webhook unavailable after 5 attempts")

print(send_signal({"action": "buy", "symbol": "ETH", "size_pct": 50, "id": "eth-breakout-2026-10-03"}))
```
