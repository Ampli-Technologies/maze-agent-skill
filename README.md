# Ampli Composer skill

An agent skill for Ampli Maze Studio. Give it to an AI agent (Cursor,
Claude Code, Claude, Codex, or any tool that reads `SKILL.md` skills) and
it can:

- build strategy compositions you import into Maze Studio, and
- send buy, sell and exit signals to a deployed agent's webhook.

## Install

Unzip it to get the `ampli-composer/` folder, then:

- Cursor: put it in `.cursor/skills/` in a project, or `~/.cursor/skills/` for every project.
- Claude Code: put it in `.claude/skills/` in a project, or `~/.claude/skills/`.
- Claude apps: upload the zip as a custom skill.
- Anything else: point the agent at `SKILL.md`.

## Build a strategy

Ask, for example: "Use the ampli-composer skill to build a strategy that
keeps $50k liquid and lends the rest". The agent answers with one JSON
block. In Maze Studio choose AI Skill -> Import composition, paste it,
read the validation report, then Rehearse before deploying.

## Send signals

1. Deploy an agent whose strategy starts with a Webhook Signal trigger,
   such as the Hyperliquid Signal Trader or Base Signal Accumulator
   template.
2. Create its URL: Agents -> the agent's journal -> Webhook -> Create URL.
   It looks like `http://localhost:3005/api/composer/webhooks/<token>` and is shown once.
3. Give it to the agent as an environment variable, `AMPLI_WEBHOOK_URL`.
   Anyone with the URL can send this agent signals, so keep it out of
   files you commit or share. Replacing the URL revokes the old one.
4. Ask, for example: "Send a buy for ETH at half size". The agent repeats
   the signal back and sends it once you confirm.

What actually trades is still bounded by the strategy's guards and the
treasury policy. Each signal's outcome is in the agent's journal.

## Keeping it current

Generated from catalog `2026.10.3.3`. When Maze Studio's catalog
changes, download the skill again (Maze Studio -> AI Skill); compositions
written against an older catalog may fail validation.

## Contents

- `SKILL.md`: what the agent reads first; both jobs in brief.
- `references/structures.md`: schema, rules, policy, webhook payload, reason codes and the node catalog.
- `references/examples.md`: every shipped template as JSON, and signal senders.
- `references/phrases.md`: requests mapped to blocks and payloads.
