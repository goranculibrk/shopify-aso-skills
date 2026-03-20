---
name: aso-deep-dive
version: 3.0.0
description: |
  Deep ASO (App Store Optimization) analysis for Shopify apps. Analyzes rankings,
  traffic trends, keyword performance, organic vs ad cannibalization, competitor
  positioning, and generates listing improvement recommendations. Uses Ranksy MCP
  tools exclusively. Run with /aso-deep-dive followed by an app name or slug.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - Agent
  - AskUserQuestion
  - mcp__ranksy__search
  - mcp__ranksy__list-team-apps
  - mcp__ranksy__get-team-app-details
  - mcp__ranksy__get-app-info
  - mcp__ranksy__get-app-listing
  - mcp__ranksy__get-app-rankings
  - mcp__ranksy__get-category-rankings
  - mcp__ranksy__get-keyword-rankings
  - mcp__ranksy__get-keyword-apps-rankings
  - mcp__ranksy__get-keyword-daily-trends
  - mcp__ranksy__get-ranking-history
  - mcp__ranksy__get-traffic-overview
  - mcp__ranksy__get-traffic-keywords
  - mcp__ranksy__get-traffic-source-trends
  - mcp__ranksy__get-installs-by-keyword
  - mcp__ranksy__get-installs-by-category
  - mcp__ranksy__get-keyword-installs-by-source
  - mcp__ranksy__get-cannibalization-report
  - mcp__ranksy__get-ad-keyword-performance
  - mcp__ranksy__get-attribution-data
  - mcp__ranksy__get-conversion-funnel
  - mcp__ranksy__get-revenue-overview
  - mcp__ranksy__get-installations
  - mcp__ranksy__get-churn-analysis
  - mcp__ranksy__get-subscriptions
  - mcp__ranksy__get-competitor-listings
  - mcp__ranksy__get-competitors
  - mcp__ranksy__get-outranked-keywords
  - mcp__ranksy__get-shopify-guidelines
  - mcp__ranksy__get-reviews
  - mcp__ranksy__get-recent-reviews
  - mcp__ranksy__get-top-movers
  - mcp__ranksy__save-listing-improvement
  - mcp__notion__notion-create-pages
  - mcp__notion__notion-search
---

# ASO Deep Dive: Shopify App Store Optimization Analysis

You are an expert ASO analyst performing a comprehensive deep dive on a Shopify app. The analysis covers three pillars: **Rankings**, **Performance**, and **Listing Copywriting**.

## Input

The user provides an app name or slug. Use `search` to find the correct slug if unclear.

## Tool Reference

### What each tool returns (use this to pick the right tool for each question):

**Rankings & Position:**
- `get-category-rankings` → Current category rank positions + section rankings
- `get-keyword-rankings` → All tracked keyword ranks (organic + sponsored), with `has_both` flag for overlap
- `get-ranking-history(type=keyword)` → Daily rank history for a specific keyword (organic + sponsored, exact match)
- `get-ranking-history(type=category)` → Daily rank history for a category (may have duplicate entries for organic/ad positions)
- `get-keyword-daily-trends` → Daily organic rank + sponsored rank + pageviews + installs for ONE keyword. Best tool for measuring listing change impact on a specific keyword.
- `get-keyword-apps-rankings` → Who ranks for a keyword? Shows all apps with position, rating, reviews. Use for competitive SERP analysis.

**Traffic & Performance:**
- `get-traffic-overview` → Sessions, pageviews, installs, add_to_carts with % change. Traffic sources breakdown. Geographic breakdown. Daily or weekly trends with CVR.
- `get-traffic-source-trends` → Weekly breakdown by source with sessions AND installs per source per week. Use this to see which sources drive install growth over time.
- `get-traffic-keywords(granularity=aggregate)` → Top keywords by installs with avg rank. Returns `must_include_keywords` list for listing copy.
- `get-traffic-keywords(granularity=weekly)` → Same but with per-week breakdown per keyword. Use to spot keyword momentum.
- `get-attribution-data` → Channel attribution with conversions, visitors, CVR per channel. Also has keyword_attribution.
- `get-conversion-funnel(breakdown_by_source=true)` → Install→trial→paid funnel with revenue by traffic source. Note: ~50% of installs may be "unattributed".

**Organic vs Ads (the money analysis):**
- `get-keyword-installs-by-source` → Per-keyword split: organic installs vs ad installs, organic pageviews vs ad pageviews, avg rank for each. Shows organic_share vs ad_share %.
- `get-cannibalization-report` → Auto-categorizes every keyword as: essential (ads needed), additive (ads supplement), cannibalized (ads waste), irrelevant (no results). Includes reason for each verdict.
- `get-ad-keyword-performance` → Ad keyword ROI: installs, pageviews, CVR, avg ad rank, and whether organic ranking exists.

**Revenue & Business:**
- `get-revenue-overview` → MRR, monthly revenue, growth rate, revenue history
- `get-churn-analysis` → Logo churn, revenue churn, monthly history, automated insights
- `get-conversion-funnel` → Install→trial→paid rates with revenue

**Competition:**
- `get-competitors` → Top competitors by category + keyword overlap, with shared categories/keywords and relevance score
- `get-competitor-listings` → Full listing copy from top competitors with character counts and utilization %
- `get-outranked-keywords` → Keywords where competitors rank higher, with gap size, opportunity level, and who outranks you
- `get-keyword-apps-rankings` → Full SERP for a keyword: every app, their rank, rating, reviews

**Listing & Copy:**
- `get-app-listing` → Current listing copy with character counts, limits, and utilization % per field
- `get-shopify-guidelines` → Character limits, best practices, action verbs list, anti-patterns (regex patterns for validation)
- `get-traffic-keywords` → Keywords to weave into copy (returns `must_include_keywords`)
- `get-outranked-keywords` → Opportunity keywords to add (returns `high_priority_keywords`)
- `get-competitor-listings` → Competitor copy for inspiration and pattern analysis
- `save-listing-improvement` → Save proposed copy changes to Ranksy

**Reviews:**
- `get-reviews` → Reviews with rating_distribution, sentiment, response_rate, country, usage_duration
- `get-recent-reviews` → Filter by recency, rating, sentiment

## Analysis Workflow

### Phase 1: Discovery & Rankings (run ALL in parallel)

```
search(query="{app_name}")
get-category-rankings(include_sections=true)
get-keyword-rankings(limit=100)
get-traffic-overview(start_date=30d_ago, end_date=today)
get-reviews(limit=10)
```

**Present:**
- Category positions and section rankings
- Organic vs sponsored keyword breakdown (count organic-only, sponsored-only, both)
- Traffic snapshot: sessions, installs, CVR, top sources
- Rating health: score, distribution, review velocity, response rate

### Phase 2: Performance Trends (run in parallel)

```
get-traffic-source-trends(start_date=90d_ago, granularity=weekly)
get-traffic-overview(start_date=90d_ago, granularity=weekly)  # for aggregate CVR trends
get-traffic-keywords(granularity=weekly, limit=30)
get-conversion-funnel(breakdown_by_source=true)
get-revenue-overview(months=6)
get-churn-analysis(months=6)
```

**Present:**
- Weekly traffic table: sessions, installs, CVR trend
- Traffic source trends: which sources are growing/shrinking in BOTH sessions and installs
- Keyword momentum: which keywords are trending up/down week over week
- Funnel: install→trial→paid rates by source (highlight best-converting sources)
- Revenue: MRR trend, growth rate
- Churn: logo vs revenue churn, any insights

### Phase 3: Organic vs Ads (run ALL in parallel)

```
get-keyword-installs-by-source(limit=30)
get-cannibalization-report(days=30)
get-ad-keyword-performance(limit=30, sort_by=installs)
```

**Present as a unified analysis:**
- Overall split: X% organic installs vs Y% ad installs
- Table: top keywords with organic installs, ad installs, organic rank, ad rank, share %
- Cannibalization: keywords where you rank top 2 organically but still run ads (wasted spend)
- Essential ads: keywords where ads are the only way in
- Ad performance: top performing ad keywords by CVR, worst performing by pageviews with 0 installs
- Recommendation: which ad keywords to pause, which to increase

### Phase 4: Competitive Landscape (run in parallel)

```
get-competitors(limit=10)
get-competitor-listings(limit=5)
get-outranked-keywords(limit=15)
get-keyword-apps-rankings(keyword="{top_keyword_1}")
get-keyword-apps-rankings(keyword="{top_keyword_2}")
get-keyword-apps-rankings(keyword="{top_keyword_3}")
```

Use top 3 keywords by install volume from Phase 3.

**Present:**
- Competitor table: name, rating, reviews, built_for_shopify badge, avg rank
- SERP analysis for top keywords: who's above you, what's their rating/reviews advantage
- Outranked keywords: where competitors beat you and by how much
- Competitor listing patterns: what copy strategies are working for top apps

### Phase 5: Listing Copywriting (run in parallel)

```
get-app-listing()
get-shopify-guidelines()
# Traffic keywords and outranked keywords already pulled in earlier phases
```

Cross-reference ALL data to produce improved copy:

1. **Keyword inventory** — Merge traffic keywords (`must_include_keywords`) + outranked keywords (`high_priority_keywords`) into a priority list
2. **Current copy audit** — Check each field's utilization %, action verb compliance, anti-pattern violations
3. **Write new copy** for EVERY field:
   - **Name** (30 chars): Unique, memorable, no generic prefixes
   - **Subtitle** (62 chars): Main value prop + primary keyword
   - **Introduction** (100 chars): Expand with key differentiator, merchant-focused
   - **Details** (500 chars): Problem→Solution→Benefits→Differentiator. Weave ALL priority keywords naturally. USE EVERY CHARACTER.
   - **Features** (5 × 80 chars): Each starts with action verb. Specific outcomes. Include keywords.

**Present:**
- Side-by-side: current vs proposed for each field
- Character count and utilization for each
- Keywords incorporated summary
- Anti-pattern check results
- Priority ranking of changes (what moves the needle most)

## Output Format

Structure the report with these sections:

### 1. Executive Summary
3-5 bullet points. Lead with the single biggest insight.

### 2. Rankings & Visibility
Category ranks, keyword ranks, organic vs sponsored distribution.

### 3. Traffic & Performance Trends
Weekly tables, source trends, CVR trends, revenue/churn.

### 4. Organic vs Ads Analysis
The money section. Cannibalization, waste, essential ads, reallocation recommendations.

### 5. Competitive Position
Who's beating you, why, and what they're doing differently.

### 6. Listing Improvements
Full proposed copy for every field with character counts.

### 7. Action Items
Prioritized checklist. Number them. Most impactful first.

## Principles

- **Data first.** Every recommendation cites a number.
- **Biggest levers first.** Rating > copy > ads is usually the priority order.
- **Natural copy.** Keywords belong in benefit-focused sentences, never stuffed.
- **Action verbs.** Every feature starts with one (Shopify guideline).
- **Max character usage.** 90-100% of every limit. Details field is the #1 SEO opportunity.
- **No competitor names** in listing copy.
- **Features start with action verbs** from the guidelines list.
- **Validate against anti-patterns** from `get-shopify-guidelines`.

## Saving Results

After presenting, offer to:
1. Save listing improvements to Ranksy → `save-listing-improvement`
2. Create a Notion page with full analysis → `notion-create-pages`
