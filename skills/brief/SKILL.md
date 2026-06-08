---
name: brief
description: Generate or refresh today's daily brief as the persistent Cowork artifact "Today's Brief" (stable id `todays-brief`). Auto-fires on "/brief", "morning brief", "today's brief", "what's on today", "what am I working on today", "give me my brief", or any phrase asking for today's working surface. Renders 5 fixed sections — Center of Gravity, Calendar Block (visual timeline + written list), Priority Tasks (richer actions), Outreach Queue (actions + category tags), Yesterday's Reflection. Filters everything against `memory/surfacing-prefs.md`. Read-only across sources — no drafting, no sends.
---

See `commands/brief.md` for the full generation workflow.

## When this skill fires

- User runs `/brief` directly
- User says: "morning brief", "today's brief", "give me my brief", "what's on today", "what am I working on today", "start my day"
- Scheduled task fires (if user opted in during `/setup-brief`)

## What this skill is NOT for

- **Drafting or sending anything.** This is read-only. Drafting happens in `/process-brief` after the user annotates.
- **Multi-day planning.** This is today only. For tomorrow, use this plugin's `/plan-tomorrow`.
- **Weekly summaries.** Use `relationships`, `referral-engine`, or cortex's `/review` for week-level work.
- **Replacing the dashboard.** Cortex's `DASHBOARD.md` is the always-on memory index. The brief is a daily working surface that includes today's slice of dashboard context.

## Inputs

- `<config-root>/identity.md` — time zone, tool inventory
- `<config-root>/plugins/daily-brief.user-context.md` — section toggles, sort defaults, empty-state behavior, annotation placeholder hints
- `<config-root>/memory/surfacing-prefs.md` — **required filter**: do-not-resurface list + noise rules applied to tasks/outreach before render
- `<config-root>/memory/DASHBOARD.md` — active project context (cortex)
- Calendar MCP — today's events (timeline strip + written list with per-meeting context)
- HubSpot MCP — priority tasks (owner=you, due today / overdue; P0/P1 only)
- relationships pipeline (`<config-root>/relationships/today.json`) / lead-engine — outreach queue (legacy weekly-outreach fallback)
- `<config-root>/briefs/<yesterday>.md` — yesterday's reflection (`## Reflection`)

## Outputs

- `<config-root>/briefs/<today>.md` — markdown twin (canonical record)
- Cowork artifact "Today's Brief" (stable id `todays-brief`) — 5 fixed sections, richer task/outreach actions persisting to localStorage `brief-YYYY-MM-DD` (schema_version 0.5.0); mined by `/end-day` Step 2c
- One short chat message confirming the brief is ready, with the twin path + filtered-item count

## Cost profile

- ~3-5K tokens of synthesis on top of raw connector payloads
- Per-section caps: 12 meetings, 12 tasks
- Cheap to run; designed to be invoked daily without breaking the bank

## Failure modes

- No `<config-root>` set → routes user to `/setup-brief`
- No calendar / inbox / CRM MCP available → that section renders as "source not connected"; other sections still populate
- Cowork artifact tools not available (Claude Code) → markdown snapshot only, with a clear notice
- `brief_enabled: false` in user-context → stops cleanly with re-enable instructions
