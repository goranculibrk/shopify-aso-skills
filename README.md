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

### `/app-performance` — Business Health Report

A quick business health check. Generates a performance report covering:

- **Installations** — active installs, weekly install/uninstall trends, free vs paid breakdown
- **Revenue** — MRR, MoM growth, net MRR added (gross minus churn)
- **Conversion funnel** — install-to-trial, trial-to-paid rates, cohort revenue
- **Churn** — logo and revenue churn with 6-month trend and averages
- **Plan mix** — subscription distribution, monthly vs annual, MRR by plan
- **3 action items** — data-driven, prioritized by impact

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

### Option 1: CLI Install (Recommended)

Use `npx skills` to install skills directly:

```bash
# Install all skills
npx skills add goranculibrk/ranksy-skills

# Install specific skills
npx skills add goranculibrk/ranksy-skills --skill aso-daily

# List available skills
npx skills add goranculibrk/ranksy-skills --list
```

This automatically installs to your `.agents/skills/` directory (and symlinks into `.claude/skills/` for Claude Code compatibility).

### Option 2: Claude Code Plugin

```
/install-plugin goranculibrk/ranksy-skills
```

## Usage

```
/aso-daily my-app-name
/aso-deep-dive my-app-name
/app-performance my-app-name
```

You can use either the app name or Shopify app slug. The skill will search for the correct slug if needed.

## License

MIT
