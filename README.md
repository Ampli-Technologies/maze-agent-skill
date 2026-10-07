# Maze agent skill

An agent skill for Ampli Maze Studio. Give it to an AI agent (Cursor,
Claude Code, Claude, ChatGPT, Grok, Codex, or any tool that reads
`SKILL.md` skills) and it can:

- build strategy compositions you import into Maze Studio, and
- send buy, sell and exit signals to a deployed agent's webhook, or hand
  it calls your own agent prepared to execute under the treasury policy.

## Install

- Cursor: `git clone https://github.com/Ampli-Technologies/maze-agent-skill ~/.cursor/skills/maze-agent-skill`
  (or into `.cursor/skills/` in one project).
- Claude Code: `git clone https://github.com/Ampli-Technologies/maze-agent-skill ~/.claude/skills/maze-agent-skill`
  (or into `.claude/skills/` in one project).
- Claude apps: download the zip from Maze Studio (AI Skill -> Download skill)
  and upload it as a custom skill.
- ChatGPT, Grok or any assistant that can read the web: Maze Studio's AI Skill
  menu opens each one with a prompt pointing at this skill. Or ask it to read
  `https://raw.githubusercontent.com/Ampli-Technologies/maze-agent-skill/main/SKILL.md` and follow it.

## Build a strategy

Ask, for example: "Use the maze-agent-skill to build a strategy that keeps
$50k liquid and lends the rest". The agent answers with one JSON block. In
Maze Studio choose AI Skill -> Import composition, paste it, read the
validation report, then Rehearse before deploying.

## Send signals

1. Deploy an agent whose strategy starts with a Webhook Signal trigger, such
   as the Hyperliquid Signal Trader or Base Signal Accumulator template, or
   the Passthrough Agent to execute prepared calls.
2. Select its Webhook Signal block in Maze Studio and choose Create key. Copy
   the address and the key; the key is shown once.
3. Give them to the agent as environment variables: `AMPLI_WEBHOOK_URL` for
   the address and `AMPLI_WEBHOOK_KEY` for the key. Anyone with the key can
   send this agent signals, so keep it out of files you commit or share.
   Rotating the key stops the old one at once.
4. Ask, for example: "Send a buy for ETH at half size". The agent repeats the
   signal back and sends it once you confirm.

What actually trades is still bounded by the strategy's guards and the
treasury policy. Each signal's outcome is in the agent's journal.

## Keeping it current

Generated from Maze Studio catalog `2026.10.5.1`. Compositions written
against an older catalog may fail validation, so pull this repository again
when Maze Studio's catalog changes.

## Contents

- `SKILL.md`: what the agent reads first; both jobs in brief.
- `references/structures.md`: schema, rules, policy, webhook payload, reason codes and the node catalog.
- `references/examples.md`: every shipped template as JSON, and signal senders.
- `references/phrases.md`: requests mapped to blocks and payloads.
