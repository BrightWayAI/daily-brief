---
name: plan-tomorrow
description: Calendar-first daily planning. Auto-fires on "/plan-tomorrow", "plan my day", "plan my tomorrow", "block my day", "set up my day", "what should I do tomorrow", "prep tomorrow", "schedule my tasks", "plan my Monday/Tuesday/etc.", "what's tomorrow look like", "help me figure out tomorrow", "organize my day", or any variation involving planning and time-blocking the upcoming workday. Pulls from CRM, working memory, and inbox; creates calendar events with rich context.
---

See `commands/plan-tomorrow.md` for the full workflow.

## When this skill fires

- User runs `/plan-tomorrow` directly
- User says: "plan my day", "plan my tomorrow", "block my day", "what should I do tomorrow", "prep tomorrow"
- User asks about scheduling tasks for the next business day
- A scheduled task triggers this (when wired up in Cowork)

## Pre-flight check

Confirm `<config-root>/plugins/plan-tomorrow.user-context.md` exists. If missing, route to `/setup-plan` first — the planner needs to know your CRM, working hours, and which companion plugins you have.

## What this skill is *not* for

- Multi-day planning. One day at a time.
- Editing existing calendar events. The skill is strictly read-only on existing events; only creates new ones.
- Replacing a CRM. Tasks come from your CRM; the skill schedules them.
