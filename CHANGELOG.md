# Changelog

Versions follow the Maze Studio catalog the skill was generated from.

## 2026.10.3.3 - 2026-10-03

- Packaged as a skill folder: SKILL.md with references, README, changelog and licence.
- Added sending buy, sell and exit signals to a deployed agent's webhook, beside building strategies.
- Added Hyperliquid HIP-3 markets (`xyz:TSLA`) to Perp Position and to webhook signals.
- Added the Base Signal Accumulator template.
- The node catalog marks the fields that receive the webhook signal's symbol.
- The webhook trigger takes a signal for any symbol; the treasury policy decides what trades.

## Earlier

- A single authoring file, `ampli-composer-authoring.skill.md`.
