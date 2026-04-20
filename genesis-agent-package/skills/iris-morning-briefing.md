---
name: Morning Briefing
description: Generate and deliver Fareed's morning briefing at 7:30 AM AST on weekdays. Synthesizes calendar, urgent emails, overnight Sentinel reports, and one relevant news item into a concise, actionable summary delivered via Telegram and portal.
---

# Morning Briefing

## Purpose

At 7:30 AM AST on weekdays, generate a concise morning briefing for Fareed that tells him everything he needs to know before his day starts, and nothing he doesn't.

The briefing is the primary touchpoint where Fareed engages with me after a night away. It sets the tone for his day. A good briefing saves him 20 minutes of catching up. A bad briefing either overwhelms him or omits something important.

## Trigger

Cron-driven at `30 7 * * 1-5` in Asia/Riyadh timezone. Also on-demand if Fareed asks "Iris, brief me."

## Steps

### 1. Gather sources

Read these in parallel:

- **Calendar:** today's events from Google Calendar (or MS Graph when Phase 5+)
- **Inbox:** emails received since last briefing (yesterday 5:30 PM), filter to:
  - From key contacts (Taqieddin, execs, partners)
  - Flagged important by sender
  - Containing explicit asks ("need your decision", "please review")
  - Marked urgent
- **Overnight events:** Sentinel reports between 10 PM yesterday and now
- **Open items:** tickets I'm tracking where something changed overnight
- **News:** one relevant item (AI, Saudi tech, cinema industry) via web search

### 2. Categorize

**Urgent (needs attention this morning):**
- Meetings in next 2 hours requiring prep
- Emails requiring reply before noon
- Any P0/P1 Sentinel events

**Today (aware, not urgent):**
- Rest of today's calendar
- Emails needing reply today but not morning
- Project milestones reaching today

**Awareness (worth knowing, no action):**
- Team activity overnight
- News items
- Background trends

### 3. Compose

Format (keep under 300 words):

```
Morning, Fareed. <day>, <date>.

📅 TODAY
• 9:00 — Call with Taqieddin re Q2 numbers (prep: <link to doc>)
• 11:00 — Brio strategic review (prep: their last email)
• 3:00 — MUVI ops sync
• 5:00 — [Free — good time for deep work]

✉️ URGENT
• Taqieddin asked for Q2 reconciliation by EOD (email from 11 PM last night)
• isolutions proposal needs sign-off — expires 5 PM today

🌙 OVERNIGHT
• Sentinel: all systems healthy, dbt ran clean at 2 AM
• Isaac completed staging refactor (PR awaiting your review)
• Oracle: no new analyses, ran her scheduled reports

🧠 WORTH NOTING
• [One relevant news item, 2 sentences max]

Want me to prep anything? Draft the reconciliation reply?
```

### 4. Deliver

- **Telegram:** full briefing as a single message to Fareed's chat
- **Portal:** briefing stored as event in message bus, displayed in portal dashboard
- **Memory:** write `iris_morning_briefing_YYYY-MM-DD.md` with briefing content plus what Fareed asked about it later

### 5. Follow through

After delivery, stay ready. If Fareed replies with actions ("yes, draft the reconciliation reply"), execute them. If he replies with questions, answer. If silence, that's fine — he's read it.

## Tone

- Conversational but efficient
- Assume Fareed is smart — no over-explanation
- Use his vocabulary (learned over time from reflections)
- Direct asks like "Draft the reconciliation reply?" at the end are welcome — he can answer yes or ignore

## Quality calibration

After Fareed's response pattern (do he engages with specific sections, skips others), I update my judgment on what to surface. Weekly I reflect on:

- Which sections did he engage with most?
- Which did he skip?
- Did I miss anything he brought up later?
- Was the briefing length right?

Adjust future briefings accordingly. Don't write more than he reads.

## Failure modes

- **MS Graph/IMAP down:** brief with calendar + Sentinel only, note "email unreachable right now, will update when available"
- **Sentinel down:** brief with calendar + email only, note "ops monitoring unavailable"
- **No news available:** skip news section rather than filler
- **Too much urgent stuff:** surface top 3, note "plus 4 more I'll batch to EOD if not acted on"

## Related skills

- `executive-email-draft.md` — when the briefing surfaces an email that needs reply
- `calendar-conflict-resolution.md` — when briefing reveals a conflict

## Related memory

- `genesis-config/memory/agent-reflections/iris_*.md` — my daily reflections inform what to surface
- `genesis-config/memory/shared-learnings/patterns/briefing-preferences.md` — accumulated learnings about Fareed's preferences
