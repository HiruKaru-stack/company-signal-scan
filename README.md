# company-signal-scan

A Claude Code skill that checks whether a company is showing active growth/buying signals right now — hiring, funding, leadership changes, expansion, tech adoption, or press coverage — using Exa MCP search.

Built by [Hiruni Jayasinghe](https://www.linkedin.com/in/hiruni-jayasinghe-b47600251).

Works in two modes:
- **Single-company**: give a website + LinkedIn URL, get a signal report in chat.
- **Batch**: give a CSV of companies, get a `-signals.csv` output with per-company verdicts and proof links.

## Install

Copy `company-signal-scan/` into your `~/.claude/skills/` directory:

```
cp -r company-signal-scan ~/.claude/skills/
```

Requires the Exa MCP server (`mcp__exa__web_search_exa`, `mcp__exa__web_fetch_exa`) to be configured in Claude Code.

## Usage

Single company:

```
/company-signal-scan acme.io linkedin.com/company/acme
```

Example output (illustrative):

```
Company: Acme (acme.io)
Overall: 2 active signal(s) found (of 6 checked)

Hiring — YES: 3 open roles (SDR, AE, Engineer) — live today — https://acme.io/careers
Funding — NO: none found
Leadership change — NO: none found
Expansion — NO: none found
Tech/tool adoption — YES: "Built on Clay" on homepage — live today — https://acme.io
Press/launch — NO: none found

Stale mentions (>3 months, not counted toward verdict):
- ...
```

Batch (CSV with domain + LinkedIn URL columns):

```
/company-signal-scan leads.csv — check hiring and funding
```

Runs a 5-row sample first for you to check, then processes the rest in waves and writes `leads-signals.csv` with a verdict and proof link per signal.

## Rules it follows

- Only signals from the last 3 months count (live careers pages always count).
- Every YES needs a clickable proof link; otherwise it's reported as UNKNOWN.
- Results are checked against the right company to avoid name collisions.
