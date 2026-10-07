# Changelog

Versions follow the Maze Studio catalog the skill was generated from.

## 2026.10.5.1 - 2026-10-05

- Added the `execute` webhook action: a signal can carry prepared calls (`to`, `data`, `value`) and Hyperliquid actions for the agent to run.
- Added the Raw Calls block, which runs those calls and actions after checking every one against the treasury policy, refusing the whole signal if any fails.
- Added the Passthrough Agent template, for an agent that plans outside Ampli and only needs Ampli to hold the keys and enforce the policy.
- A signal body may now be up to 64 KB, to fit calldata.

## 2026.10.3.3 - 2026-10-03

- Packaged as a skill folder: SKILL.md with references, README, changelog and licence.
- Added sending buy, sell and exit signals to a deployed agent's webhook, beside building strategies.
- Added Hyperliquid HIP-3 markets (`xyz:TSLA`) to Perp Position and to webhook signals.
- Added the Base Signal Accumulator template.
- The node catalog marks the fields that receive the webhook signal's symbol.
- The webhook trigger takes a signal for any symbol; the treasury policy decides what trades.
- Signals go to the agent's address on the agent composer API and carry its API key.

## Earlier

- A single authoring file, `ampli-composer-authoring.skill.md`.
