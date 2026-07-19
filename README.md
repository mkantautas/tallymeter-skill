# Tallymeter skill

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that teaches Claude (and any skills-compatible agent) to track billable time in [Tallymeter](https://tallymeter.com) — start and stop timers, log completed work, and check unbilled totals, so tracked time can become a client invoice.

Works with Claude Code, Claude Desktop, and anything else that reads `SKILL.md` skills. Pairs with Tallymeter's [MCP server](https://tallymeter.com/llms.txt) for full capability, but the REST fallback needs nothing beyond `curl`.

## Install

**Claude Code** — clone into your personal skills folder:

```bash
git clone https://github.com/mkantautas/tallymeter-skill ~/.claude/skills/tallymeter
```

(or copy just `SKILL.md` into `~/.claude/skills/tallymeter/SKILL.md`).

**Claude.ai / Claude Desktop** — zip this folder and upload it under Settings → Capabilities → Skills.

## Setup

1. Create a Tallymeter account (30-day free trial, no card): <https://tallymeter.com/register>
2. Create an API token at Settings → API tokens: <https://tallymeter.com/settings>
3. Export it where your agent runs:

```bash
export TALLYMETER_API_TOKEN=your-token-here
```

Optionally connect the MCP server for the full toolset (after-the-fact logging, unbilled summaries):

```bash
claude mcp add --transport http tallymeter https://tallymeter.com/mcp \
  --header "Authorization: Bearer $TALLYMETER_API_TOKEN"
```

## What Claude can do with it

- "Start a timer for ABC-123 code review" / "stop the timer"
- "Log 2 hours on the migration work from yesterday"
- "How much unbilled time do I have this month?"
- "How much time went into ticket ABC-123, and is it billed?"

Invoices themselves are generated in the Tallymeter web app from whatever the agent tracked.

## Links

- [Tallymeter](https://tallymeter.com) · [API docs](https://tallymeter.com/docs/api) · [llms.txt](https://tallymeter.com/llms.txt) · [OpenAPI spec](https://tallymeter.com/openapi.json)

## License

MIT
