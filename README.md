# company-signal-scan

A Claude Code skill that checks whether a company is showing active growth/buying signals right now — hiring, funding, leadership changes, expansion, tech adoption, or press coverage — using Exa MCP search.

Works in two modes:
- **Single-company**: give a website + LinkedIn URL, get a signal report in chat.
- **Batch**: give a CSV of companies, get a `-signals.csv` output with per-company verdicts and proof links.

## Install

Copy `company-signal-scan/` into your `~/.claude/skills/` directory:

```
cp -r company-signal-scan ~/.claude/skills/
```

Requires the Exa MCP server (`mcp__exa__web_search_exa`, `mcp__exa__web_fetch_exa`) to be configured in Claude Code.
