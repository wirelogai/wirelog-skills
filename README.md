# wirelog-skills

[Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) for [WireLog](https://wirelog.ai) — headless analytics for agents and LLMs.

## Install

```bash
npx @anthropic/claude-code skills add wirelogai/wirelog-skills
```

## Skills

| Skill | Description |
|---|---|
| `wirelog` | Query analytics using the pipe-based DSL |
| `wirelog-instrument` | Add event tracking and user identification to code |
| `wirelog-setup` | Set up a new WireLog project from scratch |

## Usage

Once installed, your coding agent can:

- **Query**: "What are my top events this week?" → agent runs `* | last 7d | count by event_type | top 20`
- **Query (script-tag-only projects)**: "Show traffic and top pages" → agent runs `page_view | last 7d | count by day` and `page_view | last 30d | count by _path | top 20`
- **Instrument**: "Add signup tracking" → agent adds `POST /track` calls with proper event schema
- **Setup**: "Set up WireLog for my project" → agent walks through key creation, SDK setup, verification

## What is WireLog?

Headless analytics for AI agents. Track events with HTTP, query with pipes, get Markdown back. No dashboards — your agent is the dashboard.

- 10M events free, $5/million after
- Pipe DSL: `signup | last 7d | count by day`
- Markdown output (LLM-native)
- Identity stitching (anonymous → known user)
- B2B/B2C segmentation via user profiles

Learn more at [wirelog.ai](https://wirelog.ai).
