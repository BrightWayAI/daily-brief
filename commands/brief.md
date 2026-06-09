---
description: Generate or refresh today's daily brief as the persistent Cowork artifact "Today's Brief" (stable id `todays-brief`). Renders 5 fixed sections — Center of Gravity, Calendar Block (visual timeline + written list), Priority Tasks (richer actions), Outreach Queue (actions + category tags), Yesterday's Reflection. Filters everything against `memory/surfacing-prefs.md` before render. Writes a markdown twin to `<config-root>/briefs/YYYY-MM-DD.md`. Run again any time to refresh; state persists in localStorage `brief-YYYY-MM-DD`.
---

# /brief

Builds today's working surface. The output is two coordinated things:

1. A **markdown twin** for audit at `<config-root>/briefs/YYYY-MM-DD.md` (the canonical text record).
2. A live Cowork **artifact** with stable id `todays-brief` — the working surface, with richer per-item actions that persist to localStorage and get mined by `/end-day`.

This command is **read-only across all sources**. It does not draft replies, modify CRM tasks, or send anything. Acting on annotations happens in `/process-brief`; mining the actions happens in `/end-day`.

**The brief is a working surface, not a read-only snapshot (v0.5.0).** Every interactive action writes to one localStorage blob keyed `brief-YYYY-MM-DD` and is mined by `/end-day` Step 2c.

---

## Step 0 — Resolve plugin config root and load context

### A — Resolve `<config-root>`

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.

- **Pointer exists**: read line 1 → that's `<config-root>`. Ensure access to `<config-root>` (in Cowork, `request_cowork_directory(<config-root>)` if not already mounted). Continue to section B.
- **Pointer missing**: prompt the user to run `/setup-brief` (or any other plugin's setup command) first, then stop. The brief cannot run without a config root.

### B — Load plugin config

Read `<config-root>/plugins/daily-brief.user-context.md`.

- **Exists** → parse `sections_enabled`, sort defaults, empty-state behavior, annotation placeholders.
- **Missing** → ask: "I don't see your daily-brief config. Want to run `/setup-brief` first, or use defaults?" Default to "use defaults" if the user is in a hurry.

If `brief_enabled` is `false`, stop with: "Daily brief is disabled. Re-enable in `<config-root>/plugins/daily-brief.user-context.md`."

### C — Load shared identity

Read `<config-root>/identity.md` for time zone (defines "today") and tool inventory (decides which sections will have data).

### D — Load surfacing preferences (v0.5.0 — REQUIRED filter)

Read `<config-root>/memory/surfacing-prefs.md`. This is the canonical suppression store (written by `/end-day` from `not_important` actions and the repeat-ignore rule). Parse:

- **Do-not-resurface list** — explicit per-item suppressions. An item whose title/source matches a suppressed entry MUST NOT be rendered as a priority task or outreach item, unless its linked re-surface condition has flipped.
- **Surfacing rules** — noise classes (admin/finance dunning, vendor cert/onboarding nudges). Items matching a noise class are demoted (never P0/P1); route to a low-priority "admin" mention at most, or drop.

If `surfacing-prefs.md` is missing, proceed without filtering but note it once: "No surfacing-prefs.md found — brief is unfiltered. `/end-day` will create it the first time you mark something 'not important.'"

### E — Resolve dates

`today_local = current date in user's time zone from identity.md` (fallback: system local date). Format `YYYY-MM-DD`.
`yesterday_local = today_local − 1 calendar day`.

When `/end-day` invokes this command to pre-stage, it passes `target_date = tomorrow_local`; use that in place of `today_local` everywhere below (and `target_date − 1` for the reflection read).

---

## Step 1 — Pull source data (in parallel where possible), then filter

Pull each section's source, then **apply the Step 0D filter before anything is rendered**. Suppressed/noise items are dropped silently; log the count of filtered items for the markdown twin's footer ("N items filtered by surfacing-prefs").

### Center of Gravity (section 1)

The single most important thing for the day, one or two sentences. Derive from, in order of preference:
1. Yesterday's reflection "the one thing tomorrow has to move" (read from `briefs/<yesterday_local>.md` `## Reflection`).
2. The top P0 priority task after filtering.
Phrase it as a directive sentence. Add upstream context if it sharpens urgency ("X start ~July 1 creates pull"). Not interactive.

### Calendar Block (section 2)

Pull today's calendar via the connected calendar MCP. For each event, capture: title, start, end, location/video link, attendees (excluding you). Cap at 12.

For each external attendee, attempt to find a cortex node and pull one line of context:
- `<config-root>/memory/person/<firstname>-<lastname>.md` → most recent interaction, open commitments, "why this matters / last touch."
- Fall back to `<config-root>/memory/client/<slug>.md` if the company appears in a client node, or the event description.

Two coordinated views are rendered (Step 3):
- **Visual timeline strip** — 8a–6p horizontal scale, one block per event positioned by time, color-coded `meeting` / `focus` / `personal` / `open`.
- **Written block list** — time, title, attendees, and **notes where available** (the one-line context above; "why this matters / last touch" for known person/client nodes).

Meetings are **read-only context cards** (no actions). If zero meetings, hide the whole calendar card.

### Priority Tasks (section 3)

Query CRM (HubSpot MCP if available; otherwise render "CRM not connected" placeholder):
- Tasks where `owner = current user`, `due_date <= today_local`, `status != completed`.
- **Keep P0 and P1 only.** P2 and lower do NOT appear — reduces clutter (spec A.1 §3).
- **Filter against surfacing-prefs** (Step 0D): drop suppressed items; demote noise-class items out of the priority card.
- Sort: priority desc, then due_date asc. Cap at 12.

For each surviving task capture: stable `task-<id>` id, title, due date, related contact/deal, priority (P0/P1). Each row renders the richer action set (Step A.2: done / delegate / skip / not_important / annotate). Hide the card if zero tasks survive.

### Outreach Queue (section 4)

Source from the relationships/lead-engine pipeline (look in `<config-root>/relationships/today.json` if present; else the relationships plugin's current queue; legacy `<config-root>/plugins/weekly-outreach.*` fallback only if relationships isn't installed).

- Tier each contact: **Today** (scheduled/flagged today), **This week** (`due_date <= today + 7d` or weekly-cadence overdue), **Backlog** (flagged, not date-scoped).
- For each contact capture: stable `outreach-<id>` id, name, title/company, last touch, a one-line "why," and — if the pipeline entry carries one — the **signal type** (one of lead-engine's 7: Engagement / Job change / Funding / Hiring / Growth-expansion / Tech-stack change / Direct intent). The signal is stored on the row's `data-signal` so it auto-fills the tag without asking.
- **Research link (v0.5.0)** — capture a `research_url` + `link_label` for each contact so the user can open them up before reaching out. Resolve in this order:
  1. A LinkedIn profile URL stored on the contact's cortex node — check `<config-root>/memory/person/<slug>.md` and `<config-root>/memory/bizdev/<slug>.md` for a `linkedin:` field, a `## Linked entities` LinkedIn link, or a LinkedIn URL anywhere in the node. If found → `research_url = <that URL>`, `link_label = "🔗 LinkedIn"`. Also accept a LinkedIn URL carried directly on the pipeline entry.
  2. Else build a LinkedIn people-search URL from name (+ company if known): `https://www.linkedin.com/search/results/people/?keywords=<URL-encoded "Name Company">`. `link_label = "🔍 Research"`.
  3. Else (no name usable) a plain web search: `https://www.google.com/search?q=<URL-encoded "Name Company">`. `link_label = "🔍 Research"`.
  Always produce a non-empty `research_url`.
- Filter against surfacing-prefs. Hide empty tiers; hide the card if all tiers empty.

Each row renders per-contact actions (Sent / Nudge / Booked / Skip / Let go) plus an **optional** category quick-tag (bucket + value-add chips), revealed after an action is logged (decision D.3: optional/skippable, not required).

### Yesterday's Reflection (section 5)

Read `<config-root>/briefs/<yesterday_local>.md` `## Reflection` (written by `/end-day` Step 4). Show at minimum **biggest thing done** and **the one thing that has to move today** (the latter also feeds section 1).

- Exists with `## Reflection` → render read-only.
- Exists without it → "No reflection logged yesterday — run `/end-day` next time to capture one."
- Missing → "No brief yesterday."

Always render this card (it's required).

---

## Step 2 — Write the markdown twin

Write the assembled content to `<config-root>/briefs/<today_local>.md`. The markdown is the canonical text record; the artifact is a render of it. Section order is fixed and matches the artifact:

```markdown
# Today's Brief — <today_local>

> Generated <ISO-8601 timestamp> · <N> items filtered by surfacing-prefs

## 1. Center of Gravity

<one or two sentences>

## 2. Calendar

### <time> — <title>
- **With:** <attendees>
- **Notes:** <context / why this matters / last touch — if available>

(repeat per meeting; omit section if zero)

## 3. Priority Tasks

| Task | Due | Related | Priority |
|---|---|---|---|
| <title> | <due> | <related> | P0/P1 |

(P0/P1 only; omit section if zero)

## 4. Outreach Queue

### Today
- <name> — <company> · last touch <date> · <why> [signal: <type>] · [research](<research_url>)

### This week
- ...

### Backlog
- ...

(omit empty tiers; omit section if all empty)

## 5. Yesterday's Reflection

<content from yesterday's `## Reflection`, or the appropriate placeholder>
```

Create `<config-root>/briefs/` if missing. Overwrite today's file on re-run; yesterday's file is untouched.

---

## Step 3 — Render the Cowork artifact (stable id `todays-brief`)

**Artifact identity rule:** the artifact id is ALWAYS `todays-brief`. Both `/brief` and cortex `/end-day` Step 5 `update_artifact` this same surface. Never create a parallel artifact; never produce a markdown-only fallback when Cowork is available. Formatting MUST be identical regardless of which command produced it.

Load `references/brief-artifact-template.html` (v2 layout). Substitute these tokens with today's filtered data:

**Header / meta**
- `{{DATE_LONG}}` = e.g. "Tuesday, June 9". `{{DATE_ISO}}` = `<today_local>` (drives `LS_KEY = brief-<today_local>`).
- `{{HEADER_BADGE}}` = one-line day descriptor (e.g. "Light calendar · ship the B&S plan").
- `{{FOOTER_NOTE}}` = e.g. "Run /brief to refresh · /process-brief to act on annotations · actions feed /end-day".
- `{{META_DESCRIPTION}}` = one-line artifact description. `{{META_MCP_TOOLS}}` / `{{META_MCP_SERVERS}}` = JSON arrays of the runtime's connected calendar/email/CRM MCP tool ids + server names (emit `[]` if none). In Claude Code, drop the whole `<script id="cowork-artifact-meta">` block.

**1. Center of Gravity** — `{{CENTER_OF_GRAVITY}}` = the one-or-two-sentence directive.

**2. Calendar** — feed the visual strip via the `<script>` constants, then repeat the written `.cal-item` block:
- `{{TL_START_HOUR}}` / `{{TL_END_HOUR}}` = the day window in whole hours (default `8` / `18`). `{{TL_WINDOW_LABEL}}` = e.g. "8a–6p".
- `{{TL_BLOCKS_JSON}}` = a JS array literal, one object per event, **decimal hours**: `[{s:9.5,e:10,label:'Automation chat',cls:'meeting'},{s:12,e:13,label:'Focus',cls:'focus'}]`. `cls` ∈ `meeting` | `focus` | `personal`. Emit `[]` if no events. (The template's `buildTimeline()` positions blocks against the window — no manual `left%`/`width%`.)
- Repeat the `.cal-item` block per event: `{{EVENT_TIME}}` (e.g. "9:30–10:00"), `{{EVENT_TITLE}}`, `{{EVENT_WHO}}` (attendees + location), `{{EVENT_NOTES}}` (the one-line context / why-this-matters / last-touch; omit the `.cal-notes` div if no notes).

**3. Priority Tasks** — repeat the `.task-row` block per task (P0/P1 only, post-filter):
- `data-id="task-<id>"`, `{{TASK_ID}}` = same id, `{{TASK_NAME}}` = title, `{{TASK_PRIORITY}}` = `P0`/`P1`, `{{TASK_PRIORITY_CLASS}}` = `` for P0 or ` p1` for P1, `{{TASK_SUB}}` = one-line context, `{{HINT_TASK}}` = annotation placeholder from config.

**4. Outreach Queue** — repeat the `.outreach-row` block per contact (post-filter, ordered today → this week → backlog):
- `data-id="outreach-<id>"`, `{{CONTACT_ID}}` = same id, `{{CONTACT_NAME}}`, `{{CONTACT_SUB}}` = title/company · last touch · one-line why.
- **Signal auto-fill:** in the `.oc-signal` select, emit the contact's pipeline signal as the FIRST `<option>` so it's pre-selected (fall back to `—` first if no signal). Bucket/value-add stay at their template defaults (optional — the user can change them; they're captured on action).
- `{{CONTACT_LINK}}` = the `research_url` resolved in Step 1; `{{CONTACT_LINK_LABEL}}` = `🔗 LinkedIn` when a real profile URL is known, else `🔍 Research`.

**5. Yesterday's Reflection** — `{{YESTERDAY_LABEL}}` = e.g. "Mon 6/8"; `{{REFLECT_BIGGEST}}` / `{{REFLECT_BLOCKED}}` / `{{REFLECT_ONE_THING}}` from yesterday's `## Reflection` (use "—" for blanks).

Hide any non-required card whose content is empty by omitting its `<div class="card" data-section="...">`. Center of Gravity and Yesterday's Reflection always render. To hide the calendar strip cleanly when there are no events, emit `{{TL_BLOCKS_JSON}}` = `[]` and omit the calendar card.

### localStorage contract (canonical — schema_version 0.5.0)

ONE JSON blob per day at key `brief-<today_local>`:

```json
{
  "schema_version": "0.5.0",
  "tasks":            { "<task-id>":    { "action": "done|delegate|skip|not_important", "detail": "", "ts": "", "name": "" } },
  "annotations":      { "<item-id>":    "free text" },
  "outreach_actions": { "<contact-id>": { "name": "", "action": "sent|nudge|skip|let_go|booked|dead", "bucket": "", "signal": "", "value_add": "", "detail": "", "ts": "" } },
  "tasks_checked":    { "<task-id>": true },
  "last_interaction_at": "ISO8601"
}
```

`tasks_checked` is a back-compat mirror — the template sets `tasks_checked[id] = (action === "done")` whenever a task action fires, so v0.4.x readers keep working. New readers use `tasks`. The UI emits outreach actions `sent|nudge|skip|let_go`; `booked`/`dead` are reader-accepted synonyms (`dead` = `let_go`) but not rendered by default.

Reader pattern (used by `/process-brief` and `/end-day`):

```javascript
const state = JSON.parse(widget_context["brief-<date>"] || "{}");
const tasks       = state.tasks || {};               // {task_id: {action, detail, ts, name}}
const annotations = state.annotations || {};          // {item_id: free-text}
const outreach    = state.outreach_actions || {};     // {contact_id: {name, action, bucket, signal, value_add, detail, ts}}
```

**Migration from 0.4.x:** earlier briefs stored `tasks_checked: {id: bool}`. A reader encountering `tasks_checked` but no `tasks` should treat each `true` as `{action: "done"}`. The template's `save()` always writes `schema_version: "0.5.0"` going forward.

**State is read by `/end-day` Step 2c** (mine the brief) and Step 4.0 (pre-fill reflection) via `mcp__cowork__read_widget_context(artifact_id="todays-brief")`.

### Decide create vs. update (race-aware)

1. `mcp__cowork__list_artifacts` → look for id `todays-brief`.
2. If found, check `metadata.target_date`:
   - `== intended-date` (`today_local` for `/brief`, `tomorrow_local` for `/end-day` Step 5) → `mcp__cowork__update_artifact(artifact_id="todays-brief", content=<HTML>, metadata={target_date, plugin:"daily-brief", schema_version:"0.5.0"})`. Preserve matching annotation/task state by item id (Step 3a).
   - `!= intended-date` → DO NOT silently overwrite. Surface: "⚠ `todays-brief` exists with target_date `<existing>`; about to write `<new>`. This is a `/brief`↔`/end-day` race. Proceed (last-write-wins) or abort?" On proceed → update; on abort → exit Step 3.
   - older than `today_local` → update with fresh content (the old day's final state is in its markdown twin).
3. If not found → `mcp__cowork__create_artifact(id="todays-brief", artifact_type="html", content=<HTML>, metadata={target_date, plugin:"daily-brief", schema_version:"0.5.0"})`.

### Step 3a — Preserve state across same-day re-runs

When the artifact already exists for the same date:
1. `mcp__cowork__read_widget_context(artifact_id="todays-brief")` to read the current blob.
2. The template restores `state.tasks`, `state.annotations`, and `state.outreach_actions` automatically on load (keyed by item id) — so re-rendering with the same item ids preserves everything.
3. Items dropped from today's data (e.g., a cancelled meeting) leave their prior annotation only as a line in the markdown twin under "Dropped from today's brief at <regen time>."

In Claude Code (no Cowork artifact tools): Step 3 degrades to writing the markdown twin, printing "Cowork artifact tools not available in this runtime — brief saved as markdown at `<path>`", and stopping.

---

## Step 4 — Notify the user

Output exactly one chat message:

> "Brief ready for <today_local>. Open the 'Today's Brief' artifact — act on tasks/outreach inline (those actions feed `/end-day`), annotate anything you want drafted, then run `/process-brief`. Markdown twin at `<config-root>/briefs/<today_local>.md`. <N> items filtered by surfacing-prefs."

Don't dump the brief content into chat.

---

## Behavior rules

- **Read-only across all data sources.** No drafts, no task updates, no sends.
- **Always filter against `surfacing-prefs.md`** before rendering tasks/outreach (Step 0D). Suppressed items never appear; noise-class items never reach P0/P1.
- **Fixed section order**, identical formatting whether produced by `/brief` or `/end-day` pre-stage.
- Same-day re-run = refresh + preserve action/annotation state by item id. Different day = new content; yesterday's file preserved.
- If a source MCP isn't connected, that section renders "[Section]: source not connected — connect the MCP and re-run." No silent failure.
- Cost: ~3-5K tokens of synthesis on top of connector payloads. Per-section caps: 12 meetings, 12 tasks, 10 outreach.
- Telemetry: optionally log one line to core-ops `/log-agent-run` (skill: brief, item_counts, filtered_count, runtime_ms). Skip if core-ops isn't installed.
