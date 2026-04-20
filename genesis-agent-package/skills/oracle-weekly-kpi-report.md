---
name: Weekly KPI Report
description: Generate weekly KPI report for MUVI cinema operations every Sunday at 10 AM AST. Pulls data from BigQuery across defined KPIs (revenue, ticket volume, F&B attach rate, occupancy), formats as markdown with context, and delivers to Iris for inclusion in Monday briefing.
---

# Weekly KPI Report

## Purpose

Every Sunday at 10 AM AST, produce a concise weekly KPI report covering MUVI's core cinema operations metrics. This report lands in Fareed's Monday morning briefing so he starts the week with operational awareness.

The goal is NOT exhaustive analytics. It's the handful of numbers Fareed needs to know every week, with enough context to spot issues.

## Trigger

Cron-driven at `0 10 * * 0` in Asia/Riyadh timezone (Sunday 10 AM). Also on-demand if Fareed asks "Oracle, this week's KPIs?"

## Steps

### 1. Verify schema before anything

This is my non-negotiable first step. I read the latest state of these models in `muvi-dbt-live`:

- `fct_ticket_sales`
- `fct_food_orders`
- `dim_cinema`
- `dim_movie`

Confirm no schema changes since last week that would affect my queries. If changes exist, adapt queries before proceeding.

### 2. Define the week

- Reporting week: Monday 00:00 to Sunday 23:59 of the week just completed (Asia/Riyadh)
- Comparison week: same period one week prior
- YoY comparison: same calendar week prior year

All times in Asia/Riyadh. Never UTC in this report.

### 3. Core KPIs to compute

**Revenue metrics:**
- Total ticket revenue
- Total F&B revenue
- Total revenue
- WoW % change
- YoY % change

**Volume metrics:**
- Total tickets sold
- Unique shows played
- Unique movies screened
- Average tickets per show

**Engagement metrics:**
- F&B attach rate (orders per ticket)
- Average ticket price
- Average F&B basket size

**Operational metrics:**
- Occupancy rate (tickets / available seats)
- Cinema-level breakdown (top 5, bottom 5)
- Movie-level breakdown (top 5 by revenue)

### 4. Run queries

Standard SQL pattern for each metric:

```sql
WITH this_week AS (
  SELECT ...
  FROM `muvi-prod.marts.fct_ticket_sales`
  WHERE show_date_riyadh BETWEEN @week_start AND @week_end
),
prior_week AS (
  SELECT ...
  FROM `muvi-prod.marts.fct_ticket_sales`
  WHERE show_date_riyadh BETWEEN @prior_start AND @prior_end
),
prior_year AS (
  SELECT ...
  FROM `muvi-prod.marts.fct_ticket_sales`
  WHERE show_date_riyadh BETWEEN @yoy_start AND @yoy_end
)
SELECT
  tw.total AS this_week,
  pw.total AS prior_week,
  py.total AS prior_year,
  SAFE_DIVIDE(tw.total - pw.total, pw.total) AS wow_pct,
  SAFE_DIVIDE(tw.total - py.total, py.total) AS yoy_pct
FROM this_week tw
CROSS JOIN prior_week pw
CROSS JOIN prior_year py;
```

Use parameterized dates. Always `SAFE_DIVIDE` for percentage changes (guards against zero denominators).

### 5. Reconcile before reporting

Before publishing, verify consistency:

- Total ticket revenue should equal sum of cinema-level revenue (within rounding)
- Total tickets should equal sum of per-movie tickets
- If variance exceeds 0.1%, investigate before reporting — usually a date range or timezone issue

If reconciliation fails, report with explicit note rather than hiding the discrepancy.

### 6. Format the report

Output location: `genesis-config/memory/project-logs/muvi-kpi/kpi_YYYY-WW.md`

```markdown
# MUVI Weekly KPIs — Week YYYY-WW (Monday <date> to Sunday <date>)

## Revenue
| Metric | This Week | WoW | YoY |
|--------|-----------|-----|-----|
| Ticket revenue | SAR X.X M | ±X.X% | ±X.X% |
| F&B revenue | SAR X.X M | ±X.X% | ±X.X% |
| Total revenue | SAR X.X M | ±X.X% | ±X.X% |

## Volume
| Metric | This Week | WoW | YoY |
|--------|-----------|-----|-----|
| Tickets sold | XXX,XXX | ±X.X% | ±X.X% |
| Shows played | X,XXX | ±X.X% | ±X.X% |
| Avg tickets/show | XX.X | ±X.X% | ±X.X% |

## Engagement
- F&B attach rate: X.XX (prior week: X.XX)
- Avg ticket price: SAR XX.XX (prior week: SAR XX.XX)
- Occupancy rate: XX.X% (prior week: XX.X%)

## Top 5 cinemas by revenue
1. <Cinema Name>: SAR X.X M (X.X% of total)
2. ...

## Top 5 movies by revenue
1. <Movie Name>: SAR X.X M (X,XXX tickets)
2. ...

## Notable observations
- <One or two sentences on anything standing out>
- <Flag any reconciliation issues or data quality concerns>

## Queries used
- See `kpi_YYYY-WW.sql` alongside this file for full query set
```

### 7. Deliver

- **Memory:** commit the markdown report + the SQL file to `genesis-config`
- **Iris:** send message on the bus — "Weekly KPI report ready for Monday briefing"
- **Portal:** report viewable in portal's report browser

Iris picks it up automatically for the Monday 7:30 AM briefing.

## Tone and style

- Numbers rounded sensibly (SAR millions with one decimal, ticket counts with thousands separator)
- Percentages with one decimal point
- "Notable observations" is where I add value — don't just repeat numbers, flag what's interesting
- Don't interpret business implications — I report "ticket revenue up 12% while ticket volume up 3%," I don't say "this is due to strategic pricing changes"

## Failure modes

- **BigQuery unavailable:** retry with exponential backoff, alert Iris if still failing after 15 minutes
- **Schema changed, query broke:** fix the query, note the schema change in my reflection, update memory facts
- **Reconciliation mismatch:** report with explicit variance callout, investigate in follow-up
- **Missing data (backfill delayed):** report what's available, flag gaps

## Quality calibration

After Fareed reads this report, I watch for:
- Questions he asks about it (suggests something I should have included)
- Numbers he doesn't engage with (maybe I don't need to include next time)
- Corrections to my observations (calibrate my noticing)

Update memory facts and reflections accordingly.

## Related skills

- `bigquery-schema-verify.md` — the verification step I do first
- `reconciliation-investigation.md` — when numbers don't tie, how I diagnose
- `muvi-dbt-model-reading.md` — how I navigate muvi-dbt-live for schema understanding

## Related memory

- `genesis-config/memory/project-logs/muvi-kpi/` — historical reports for trend comparison
- `genesis-config/memory/shared-learnings/patterns/reconciliation-rules.md` — accumulated knowledge of known data quirks
