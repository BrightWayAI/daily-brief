---
description: Plan your next business day end-to-end. Pulls CRM tasks, working memory context, and inbox action items, then creates time blocks on your calendar with full context baked into each block. The calendar IS the plan — no separate document.
---

# /plan-tomorrow

Calendar-first daily planning.

The output of this command is **calendar events with full context baked in**. You should be able to open your calendar tomorrow morning and just start working — every block tells you what to do, why it matters, and what "done" looks like.

---

## Step 0 — Preflight

Read `<config-root>/plugins/plan-tomorrow.user-context.md`. If missing or unpopulated, route to `/setup-plan` and stop.

Extract:
- Identity (name, company, what you do)
- CRM (tool name, owner ID)
- Working hours (start, end, time zone)
- Calendar conventions (color for task blocks, emoji map, max blocks per day)
- Companion plugins (cortex installed? core-ops for pipeline-analyst? weekly-outreach?)

---

## Critical Rules

- **NEVER edit or update existing calendar events.** Existing events are read-only context. Only CREATE new events. Always use `sendUpdates: "none"`.
- **Present a simple confirmation table** before creating events. Wait for approval.
- **Next business day logic:** Mon–Thu → plan tomorrow. Friday → plan Monday. If the user specifies a day, use that.
- **Don't overpack.** Leave breathing room. A realistic day has 4–6 task blocks around existing meetings, not 10. Include one buffer block.
- **Task events get color.** Use the `task-block-color-id` from user-context (default `"5"` — Banana yellow).

---

## Step 1 — Determine the target day

Mon–Thu → tomorrow. Friday → Monday. User-specified date wins.

Announce briefly: "Planning **[Day, Month Date]**."

---

## Step 2 — Gather everything (in parallel)

Run all data-gathering steps simultaneously.

### 2A — Scan the target day's calendar
Query your calendar for the target day, working-hours window. Capture:
- Existing meetings (time, title, attendees)
- Free slots available for task blocks

### 2B — Pull project tasks from CRM
**If `pipeline-analyst` is available** (core-ops installed): use the Task tool with `subagent_type="pipeline-analyst"` and pass:
- `user-context-path` — path to core-ops's user-context (or this plugin's, if it has CRM details)
- `time-window` — 14 days
- `focus-filter` — `"deals or contacts with action due today or tomorrow"`
- `top-n` — 5

The agent returns a ranked list. Use the top items as candidate task blocks.

**If pipeline-analyst is not available** (or this plugin's user-context says CRM is "none"): query the CRM directly for tasks owned by you (per `crm-owner-id` in user-context), status NOT_STARTED or IN_PROGRESS, due on/before the target day. Sort by priority then due date.

### 2C — Pull project context from working memory
**If `claude-cortex` is installed:** Memory lives at `~/Documents/Claude/memory/`. Read `DASHBOARD.md` for the master index, then read individual node files for active projects. Extract: P0/P1 actions, open threads, deadlines, blockers.

**If cortex is not installed:** skip this step. Note in the final output that memory wasn't available.

### 2D — Scan inbox for action items
Search Gmail (or whatever email connector is available) for:
- Unanswered inbound from CRM contacts/stakeholders
- Threads where you were last sender with no reply (3+ business days)
- Implied action items in recent threads

### 2E — Check for weekly outreach plan (if applicable)
**If `weekly-outreach` is installed:** check whether it produced a current week's outreach queue. If so, pull the remaining contacts (not yet sent) and any HOLDING PATTERN entries. These feed Step 3's outreach block.

**If not installed:** skip this. Outreach blocks become "if there are clearly good outreach signals from CRM, batch them; otherwise skip."

---

## Step 3 — Build the event list

Merge everything from Step 2 into calendar events. Think in blocks, not a report. Each item becomes an event with a title, time slot, and description.

**Prioritization for slot assignment:**
1. **Meeting prep** — 30 min before each external meeting. Gets first claim.
2. **Client deliverables & deep work** — proposals, builds, complex docs. Longest available morning block (60–90 min).
3. **Responses & follow-ups** — emails needing replies, threads to advance. Fits 30-min gaps between meetings.
4. **Outreach** — batched into one 30-min block if 3+ contacts. Skip if fewer than 3 or day is full.
5. **Post-call capture** — 30 min at end of day to commit notes (use `/remember` if cortex is installed). Only if external meetings happened.

**Scheduling principles:**
- Deep work in the morning before meetings fragment the day
- Meeting prep immediately before its tied meeting
- Outreach batched, not scattered
- One 30-min buffer block for overflow
- Don't schedule past your `working-hours-end`
- Don't create more than `max-blocks-per-day` task blocks (default 6)

**Deduplication:** If a CRM task and a memory item refer to the same thing, merge into one event.

---

## Step 4 — Write rich event descriptions

This is where the value lives. Every description should contain everything you need to execute that block without looking anything up. Write as if you're reading the description cold at 9am tomorrow.

**For deep work / deliverable blocks:**
```
GOAL: [One sentence — what you're producing]

CONTEXT: [Why this matters right now, what project it's part of]

KEY INFO:
- [Specific facts, names, numbers needed]
- [Reference docs or prior decisions]
- [Gotchas from memory — things to watch out for]

DONE WHEN: [Clear completion criteria]
```

**For meeting prep blocks:**
```
MEETING: [Time] — [Meeting name] with [Who]
GOAL: [What outcome you want from this meeting]

CONTEXT:
- [Relationship status, last interaction]
- [What they said / committed to last time]
- [Key intel from memory, CRM, or Gmail]

TALKING POINTS:
- [Point 1]
- [Point 2]
- [Point 3]

GOTCHAS:
- [Anything from memory to watch out for]
```

**For email/response blocks:**
```
EMAILS TO SEND:

1. [Name] ([Org]) — [email address]
   Context: [What they wrote, what's needed]
   Action: [What to say / what to send]

2. [Name] ([Org]) — [email address]
   Context: [...]
   Action: [...]
```

**For outreach blocks:**
```
OUTREACH BATCH ([N] contacts):

1. [Name] — [Title], [Org]
   Why now: [trigger]
   Last contact: [date]
   Channel: [email/LinkedIn]

2. ...

Writing rules: lead with value, no "just checking in", one CTA, 3–5 sentences.
```

**For post-call capture:**
```
Capture notes from today's calls while fresh:
- [Meeting 1]: What was decided? What's the next step?
- [Meeting 2]: ...

Run /remember to commit to working memory.
```

---

## Step 5 — Present for confirmation

Show a simple table — not a full document. The user just needs to see the shape of the day.

```
**[Day, Month Date]** — [N] task blocks around [N] meetings

| Time  | Block                          | Duration |
|-------|--------------------------------|----------|
| 9:00  | 🔨 [Deep work title]           | 90 min   |
| 10:30 | 📧 [Emails: who]               | 30 min   |
| 11:30 | 📝 [Prep: meeting name]        | 30 min   |
| 1:00  | 🔨 [Deep work title]           | 60 min   |
| 4:00  | 📋 [Post-call capture]         | 30 min   |
| 4:30  | 🔲 Buffer                      | 30 min   |

(Existing meetings not shown — already on your calendar)

Create these blocks?
```

Keep this tight. No task details, no meeting prep notes, no outreach lists in the confirmation — that's all in the descriptions.

---

## Step 6 — Create calendar events

After approval, create all events. For each:
- **Title format:** emoji + short name (defaults: 🔨 deep work, 📧 emails, 📝 meeting prep, 📋 capture, 🔲 buffer)
- **colorId:** from `task-block-color-id` in user-context (default `"5"`)
- **sendUpdates:** `"none"` always
- **Description:** Full context as written in Step 4

After creating, confirm: "Done — **[N] blocks** on your calendar for **[Day, Month Date]**. Your day is set."

---

## Edge cases

- **Wall-to-wall meetings:** Say so. Propose 1–2 micro-blocks (15 min) in gaps for highest-priority items only. Don't force a full plan.
- **No CRM tasks:** Fine — memory and Gmail usually have plenty.
- **Memory empty or unavailable:** Lean on CRM and Gmail. Mention `/remember` for building memory over time (if cortex is installed).
- **Few outreach candidates:** Skip the outreach block. Don't pad.
- **User wants changes:** Adjust and re-present the table. Don't recreate events until they confirm.
- **Vague memory items:** Enrich from CRM or Gmail before creating the event. Every event should be specific enough to act on.
- **CRM connector unavailable:** Skip Step 2B and proceed with what you have. Note it in the confirmation.
