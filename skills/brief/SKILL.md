---
name: brief
description: Generate or refresh today's daily brief as a live Cowork artifact titled "Today's Brief". Auto-fires on "/brief", "morning brief", "today's brief", "what's on today", "what am I working on today", "give me my brief", or any phrase asking for today's working surface. Pulls calendar, inbox action items, CRM priority tasks, outreach queue, yesterday's reflection. Read-only across sources — no drafting, no sends.
---

See `commands/brief.md` for the full generation workflow.

## When this skill fires

- User runs `/brief` directly
- User says: "morning brief", "today's brief", "give me my brief", "what's on today", "what am I working on today", "start my day"
- Scheduled task fires (if user opted in during `/setup-brief`)

## What this skill is NOT for

- **Drafting or sending anything.** This is read-only. Drafting happens in `/process-brief` after the user annotates.
- **Multi-day planning.** This is today only. For tomorrow, use `plan-tomorrow` plugin's `/plan-tomorrow`.
- **Weekly summaries.** Use `referral-engine`, `weekly-outreach`, or cortex's `/review` for week-level work.
- **Replacing the dashboard.** Cortex's `DASHBOARD.md` is the always-on memory index. The brief is a daily working surface that includes today's slice of dashboard context.

## Inputs

- `<config-root>/identity.md` — time zone, tool inventory
- `<config-root>/plugins/daily-brief.user-context.md` — section toggles, sort defaults, empty-state behavior, annotation placeholder hints
- `<config-root>/memory/DASHBOARD.md` — active project context (cortex)
- Calendar MCP — today's events
- Gmail MCP — inbox action items (or delegated to inbox-triage if installed)
- HubSpot MCP — priority tasks (owner=you, due today / overdue)
- `<config-root>/plugins/weekly-outreach.*` — outreach queue (if installed)
- `<config-root>/briefs/<yesterday>.md` — yesterday's reflection

## Outputs

- `<config-root>/briefs/<today>.md` — markdown snapshot (canonical record)
- Cowork artifact "Today's Brief" — interactive HTML with textarea annotations (or markdown-only fallback in Claude Code)
- One short chat message confirming the brief is ready, with the snapshot path

## Cost profile

- ~3-5K tokens of synthesis on top of raw connector payloads
- Per-section caps: 12 meetings, 7 inbox items, 12 tasks
- Cheap to run; designed to be invoked daily without breaking the bank

## Failure modes

- No `<config-root>` set → routes user to `/setup-brief`
- No calendar / inbox / CRM MCP available → that section renders as "source not connected"; other sections still populate
- Cowork artifact tools not available (Claude Code) → markdown snapshot only, with a clear notice
- `brief_enabled: false` in user-context → stops cleanly with re-enable instructions
