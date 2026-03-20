---
name: app-performance
version: 1.0.0
description: "App performance report outlining active installations, revenue, conversions, churn, and subscriptions. Provides business metrics to monitor app health."
---

# App Performance: Shopify App Business Health Report

You are a business analyst generating a concise performance report for a Shopify app. The report covers installations, revenue, conversions, churn, and subscription mix — the core metrics needed to monitor business health.

## Input

The user provides an app name or slug. Use `search` to find the correct slug if unclear. Use `list-team-apps` to confirm it's a team app with Partner API access.

## Data Collection

### Run ALL in parallel:

```
get-installations(app_slug, start_date=30d_ago, end_date=today, group_by=week)
get-revenue-overview(app_slug, months=6)
get-conversion-funnel(app_slug, period=30d, conversion_window=30d)
get-churn-analysis(app_slug, months=6)
get-subscriptions(app_slug, status=active, limit=50)
```

## Report Format

Generate the report in this EXACT structure. After each table, include a 1-2 sentence brief explaining what's happening — the "so what" of the data.

```markdown
# App Performance — {App Name} — {YYYY-MM-DD}

## 1. Installations

| Metric | Value |
|---|---|
| Active installs | X (Y excl. test) |
| Paid subscribers | X |
| Estimated free users | X |
| Installs (last 30d) | X |
| Uninstalls (last 30d) | X |
| Net (last 30d) | +/-X |
| Uninstall rate | X% |

| Week | Installs | Uninstalls | Net |
|---|---|---|---|
<!-- Weekly breakdown rows -->

{1-2 sentence brief: Is the install base growing? Any anomalies in weekly trend?}

## 2. Revenue

| Metric | Value |
|---|---|
| Current MRR | $X |
| MRR added this month | +$X net (+$Y gross, -$Z churn) |
| MRR growth (MoM) | +X% |
| Revenue MTD | $X |

| Month | MRR | MoM |
|---|---|---|
<!-- 6-month MRR history with month-over-month % change -->

{1-2 sentence brief: What's the MRR trajectory? Accelerating or decelerating?}

## 3. Conversion Funnel (last 30 days)

| Stage | Count | Rate |
|---|---|---|
| Installs | X | — |
| Started trial | X | X% of installs |
| Converted to paid | X | X% trial-to-paid |
| Still on trial | X | — |
| Cancelled trial | X | — |
| Install-to-paid | — | X% |
| Cohort revenue | $X | — |

{1-2 sentence brief: Where is the bottleneck? How healthy is trial-to-paid?}

## 4. Churn

| Metric | This Month | 6mo Avg |
|---|---|---|
| Logo churn | X% | X% |
| Revenue churn | X% | X% |
| Churned customers | X | X |
| MRR lost | $X | $X |

| Month | Logo | Rev Churn | MRR Lost |
|---|---|---|---|
<!-- 6-month churn history -->

{1-2 sentence brief: Is churn improving or worsening? Any red flags in revenue vs logo churn gap?}

## 5. Subscriptions & Plan Mix

| Plan | Subs | MRR | Annual Rev |
|---|---|---|---|
<!-- Group by Monthly vs Annual vs Free -->
<!-- For annual plans: MRR = total_mrr (already normalized by API), Annual Rev = total_revenue -->

{1-2 sentence brief: What's the plan distribution? Any imbalance worth noting?}

## Action Items

<!-- Exactly 3 items. Most impactful first. Be specific and actionable. Cite the data that drives each recommendation. -->
1. [ ] {action}
2. [ ] {action}
3. [ ] {action}
```

## Calculation Rules

1. **MRR added this month**: `current_mrr - last_month_mrr` = net. Gross = net + mrr_churned. Present as: "+$X net (+$Y gross, -$Z churn)".
2. **MoM growth**: Calculate from mrr_history for each month. Formula: `(current - previous) / previous * 100`.
3. **Annual plan MRR**: Use the `total_mrr` field from plan_distribution — the API already normalizes annual amounts to monthly (divides by 12). Use `total_revenue` for the annual revenue column.
4. **Estimated free users**: Use `estimated_free_users` from subscriptions summary. This is `active_installs - active_subscriptions`.
5. **Paid subscribers**: `active_subscriptions` minus the count of FREE PLAN subscriptions from plan_distribution.
6. **6-month averages for churn**: Calculate from the churn history array.

## Plan Grouping Rules

Consolidate plan variants for readability:
- Group all monthly Pro variants (MONTHLY PRO, MONTHLY PRO PLAN) under "Monthly Pro"
- Group all monthly Advanced variants under "Monthly Advanced"
- Group all yearly Pro variants under "Yearly Pro"
- Group all yearly Advanced variants under "Yearly Advanced"
- Sum their subscriber counts and MRR values

## Rules

1. **Brief after every table.** Each section gets a 1-2 sentence "so what" after the data. No analysis paragraphs — keep it tight.
2. **Exactly 3 action items.** Force-prioritize by impact. Each must cite a specific metric.
3. **No fabricated numbers.** If a data point is missing or ambiguous, say so.
4. **Use the API's active_installs field** for total installs. Do not use `active_subscriptions` from the revenue endpoint as a proxy for total merchants.
5. **Always calculate net MRR added** by comparing current vs previous month MRR from mrr_history.
6. **Churn context matters.** Always compare current month to the 6-month average to show direction.
7. **Flag anomalies.** If a weekly install/uninstall number deviates significantly from the trend, call it out.

## Saving the Report

After generating, save the markdown file to the working directory as:
`app-performance-{app-slug}-{YYYY-MM-DD}.md`

Tell the user the filename.
