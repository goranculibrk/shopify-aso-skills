# Ranksy Skills

Claude Code plugin for Shopify app analytics. Powered by [Ranksy](https://ranksyapp.com) MCP tools.

## Skills Included

### `/aso-daily` — Daily Dashboard

A 60-second scan of what changed. Generates a concise daily report with:

- **Pulse metrics** — sessions, installs, CVR, reviews, category rank (week-over-week)
- **Keyword movements** — rising, falling, new appearances, lost rankings
- **Ad snapshot** — ad installs, CVR, top/waste keywords
- **Competitor watch** — event-driven changes only
- **Flags** — threshold-based alerts
- **3 action items** — force-prioritized, most impactful first

### `/aso-deep-dive` — Comprehensive Analysis

A full ASO audit covering three pillars — Rankings, Performance, and Listing Copywriting:

- **Rankings & visibility** — category ranks, keyword distribution, organic vs sponsored
- **Traffic & performance trends** — weekly tables, source trends, CVR, revenue, churn
- **Organic vs ads analysis** — cannibalization, wasted spend, essential ads, reallocation recs
- **Competitive position** — SERP analysis, outranked keywords, competitor listing patterns
- **Listing improvements** — full proposed copy for every field with character counts
- **Action items** — prioritized checklist

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [Ranksy](https://ranksyapp.com) MCP server configured in your Claude Code settings

## Installation

```
/plugin marketplace add goranculibrk/ranksy-skills
/plugin install ranksy-skills
```

## Usage

```
/aso-daily my-app-name
/aso-deep-dive my-app-name
```

You can use either the app name or Shopify app slug. The skill will search for the correct slug if needed.

## Manual Installation

If you prefer not to use the plugin system:

```bash
git clone https://github.com/goranculibrk/ranksy-skills.git
ln -s "$(pwd)/ranksy-skills/skills/aso-daily" ~/.claude/skills/aso-daily
ln -s "$(pwd)/ranksy-skills/skills/aso-deep-dive" ~/.claude/skills/aso-deep-dive
```

## License

MIT
