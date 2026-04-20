# <Project Name> — <YYYY-MM-DD>

<!--
This template is for per-project, per-date execution logs.
Filename convention: <project_slug>_YYYY-MM-DD.md
Location: genesis-config/memory/project-logs/<project>/
Purpose: Record what was asked, what was done, how it was tested, and the outcome.
Used by: the writing agent (for continuity), other agents (coordination), Fareed (review), Sentinel (anomaly detection).
-->

## Agent

<Which agent wrote this entry. One of: isaac | oracle | iris | sentinel>

## Session window

<Start and end timestamps of the work session this log covers. Asia/Riyadh timezone.>

Example: 2026-04-19 14:32 to 17:08 AST

## Asked

<Exactly what Fareed asked, or the ticket/task that triggered this work. Verbatim if possible — preserve his words, don't paraphrase.>

<If delegated by another agent (e.g., Iris tasked Oracle), note that with reference to the bus trace ID.>

## Plan

<Brief statement of approach before starting. What I intended to do, in what order.>

## Actions taken

<Chronological list of what was actually done. Be specific. Include:
- Commands run (the important ones, not every `ls`)
- Files touched (paths)
- Services restarted
- PRs opened
- Queries executed
- APIs called

Format as a bulleted list or numbered steps.>

Example:
- 14:35 — Read `models/staging/stg_orders.sql` to understand current structure
- 14:42 — Identified 4 downstream models depending on it
- 14:50 — Created branch `refactor/staging-prefix-consistency`
- 15:05 — Renamed `stg_orders` to `stg_tasker__orders`
- 15:10 — Updated imports in 4 downstream models
- 15:30 — Ran `dbt compile` — no errors
- 15:40 — Ran `dbt test --select state:modified` — all passed
- 16:00 — Pushed branch, opened PR #247
- 16:15 — Coordinated with Oracle via bus message (trace_id: abc123)
- 16:45 — PR reviewed and merged after approval
- 17:00 — Verified production dbt run includes new name

## Testing

<How I verified that what I did actually works. Be specific:
- What tests I ran
- What manual verification I performed  
- What I compared against baseline
- What I couldn't verify and why>

## Outcome

<One of: Success | Partial | Failed | Deferred>

<Then a sentence explaining. "Success" gets one line. "Partial" or "Failed" gets more detail.>

## Notes

<Anything worth remembering for next time working on this project:
- Surprising discoveries
- Known issues uncovered
- Decisions made that might be relevant later
- Warnings for future me or other agents>

## Trace IDs

<Links to message bus traces for detailed replay:
- Initial request trace: <uuid>
- Cross-agent coordination traces: <uuid>
- Tool call traces: <uuid>

These point to the messages and tool_calls tables for full debugging detail.>

## References

<Related project logs, related reflections, external docs, PR links, etc.:
- PR: <link>
- Previous log: <link to yesterday's log if continuing>
- Related issue: <link>
- Design doc section: <reference>>

## Memory updates

<Did I write any structured facts during this session? List them:
- `stg_tasker__orders` is the new canonical name (replaces `stg_orders`)
- Renaming scheme confirmed as `stg_<source>__<entity>`

These are also in Layer 5 memory via memory.store() calls.>
