---
description: Generate or refresh today's daily brief as a live Cowork artifact titled "Today's Brief". Pulls calendar, inbox action items, CRM priority tasks, outreach queue, yesterday's reflection. Writes a markdown snapshot to `<config-root>/briefs/YYYY-MM-DD.md` and renders the artifact via Cowork's create_artifact / update_artifact. Run again any time to refresh.
---

# /brief

Builds today's working surface. The output is two things: a markdown snapshot for audit (`<config-root>/briefs/YYYY-MM-DD.md`) and a live Cowork artifact titled "Today's Brief" with textarea annotations per item. After you annotate, run `/process-brief` to act on the annotations.

This command is **read-only across all sources**. It does not draft replies, modify CRM tasks, or send anything. That work happens in `/process-brief`.

---

## Step 0 — Resolve plugin config root and load context

### A — Resolve `<config-root>`

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.

- **Pointer exists**: read line 1 → that's `<config-root>`. Ensure access to `<config-root>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<config-root>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Continue to section B.
- **Pointer missing**: prompt the user to run `/setup-brief` (or any other plugin's setup command) first, then stop. The brief cannot run without a config root.

### B — Load plugin config

Read `<config-root>/plugins/daily-brief.user-context.md`.

- **Exists** → parse `sections_enabled`, sort defaults, empty-state behavior, annotation placeholders.
- **Missing** → ask: "I don't see your daily-brief config. Want to run `/setup-brief` first, or use defaults (all 7 sections, conservative settings)?" Default to "use defaults" if the user is in a hurry.

If `brief_enabled` is `false`, stop with: "Daily brief is disabled. Re-enable in `<config-root>/plugins/daily-brief.user-context.md`."

### C — Load shared identity

Read `<config-root>/identity.md` for time zone (defines "today") and tool inventory (decides which sections will have data).

### D — Resolve today's date

`today_local = current date in user's time zone from identity.md` (fallback: system local date if time zone missing). Format: `YYYY-MM-DD`.

`yesterday_local = today_local - 1 day` (calendar day, not 24 hours).

---

## Step 1 — Pull source data (in parallel where possible)

Each enabled section in `sections_enabled` triggers its source pull. Sections disabled in config are skipped entirely.

### Section 1 — Today's meetings

Pull today's calendar via the connected calendar MCP. For each event:

- Title, start, end, location / video link
- Attendees (excluding you)
- For each external attendee, attempt to find a cortex node:
  - Look for `<config-root>/memory/person/<firstname>-<lastname>.md` (if Phase 3 person pages exist)
  - Fall back to `<config-root>/memory/client/<slug>.md` if the attendee's company appears in any client node
  - Capture: most recent activity date, last known thread, any open commitments to / from this person
- Skip blocked time and personal events the user has flagged personal (configurable later — for v1, include all events).

Sort per config (default: `start_time_asc`).

**Meetings are read-only context cards as of daily-brief v0.3.0.** The annotation textarea per meeting was removed — talking-point annotations didn't route to anything more useful than a bullet append to the markdown file, and the empty textareas were the loudest source of "process-brief overhead" UX friction. To add prep notes for a meeting, edit the markdown snapshot directly or use cortex `/recall <person>` to surface context.

If there are no meetings today, hide the entire section in the artifact (don't render an empty card).

### Section 2 — Inbox action items

If the user has the `inbox-triage` plugin installed and its skill `triage-inbox` is available, delegate to it:

- Run the skill; expect a structured list of top 3-7 Needs-reply-today threads with sender, subject, snippet, suggested framing.

If `inbox-triage` is not installed, fall back to the simple heuristic:

- Gmail search: `is:unread newer_than:1d (to:me OR cc:me)` AND the last message is not from the user.
- Return up to 7 threads. Each item: sender display name, subject, first 200 chars of last message, message_id (for reply linking later).

### Section 3 — Today's priority tasks

Query CRM (HubSpot MCP if available; otherwise skip with a "CRM not connected" placeholder):

- Tasks where `owner = current user`, `due_date <= today_local`, `status != completed`.
- Sort per config (default: priority desc, then due_date asc).
- Cap at 12. Anything beyond goes into a "+N more" tail line.

For each task: title, due date, related contact / deal (if any), priority.

### Section 4 — Outreach queue

If `weekly-outreach` plugin is installed and has produced a current week's queue (look in `<config-root>/plugins/weekly-outreach.*` for the latest queue file referenced by that plugin), pull today's slice:

- For each contact in the queue scheduled for today (or unscheduled but flagged for this week with no specific day), include: name, company, last touch date, draft status (drafted / pending / sent).
- If no current queue → if config `empty_state.outreach == hide`, omit the section; otherwise render with "No outreach scheduled today."

### Section 5 — Drafted replies awaiting approval

This section is populated by `/process-brief`. **Hide entirely on a fresh `/brief` run when no drafts exist** (daily-brief v0.3.0 — don't show a placeholder for nothing). Render only after `/process-brief` has produced at least one draft today; thereafter the section persists until the next morning's brief regenerates.

### Section 6 — Yesterday's reflection

Read `<config-root>/briefs/<yesterday_local>.md` if it exists.

- **Exists with a `## Reflection` section** (appended by yesterday's `/end-day`) → display the reflection content read-only.
- **Exists without a `## Reflection` section** (yesterday's `/end-day` wasn't run) → "No reflection logged yesterday — run `/end-day` next time to capture one."
- **Missing** → "No brief yesterday."

End-of-day reflection capture moved to cortex's `/end-day` Step 4 as of daily-brief v0.3.0 — the brief no longer carries empty reflection textareas. Reflection prompts live where reflection actually happens.

---

## Step 2 — Write the markdown snapshot

Write the assembled brief content to `<config-root>/briefs/<today_local>.md`. Use this structure (the markdown is the canonical record; the artifact is a render of it):

```markdown
# Today's Brief — <today_local>

> Generated <ISO-8601 timestamp>

## 1. Meetings

### <time> — <title>
- **With:** <attendee list>
- **Context:** <cortex snippet, 1-2 lines>
- **Open thread:** <if any>

(repeat per meeting — read-only context, no annotation)
(omit this whole section if zero meetings)

## 2. Inbox action items

### <sender> — <subject>
- **Received:** <timestamp>
- **Snippet:** <first 200 chars>
- **Suggested framing:** <1-2 lines from inbox-triage if available>
- **Annotation:** _(empty)_

(repeat — omit section if zero items)

## 3. Priority tasks

| Task | Due | Related | Priority |
|---|---|---|---|
| ... | ... | ... | ... |

- **Annotation per task:** _(empty)_

(omit section if zero tasks)

## 4. Outreach queue

(list — omit section if no outreach scheduled today)

## 5. Drafted replies awaiting approval

(omit section if no drafts yet; populated by /process-brief)

## 6. Yesterday's reflection

<content from yesterday's brief's `## Reflection` section, or "No reflection logged yesterday">

(end-of-day prompts removed in v0.3.0 — reflection is captured by cortex /end-day Step 4 instead)
```

Create the `<config-root>/briefs/` directory if it doesn't exist. Overwrite today's file if it already exists (re-runs replace; yesterday's file is untouched).

---

## Step 3 — Render the Cowork artifact (v0.4.0 canonical 6-section format)

**Artifact identity rule (v0.4.0+):** the artifact id is ALWAYS `todays-brief`. Both `/brief` and cortex `/end-day` Step 5 reference the same persistent surface. Never create a new artifact with a different id; never produce a markdown-only fallback when Cowork is available.

Load the HTML template from `references/brief-artifact-template.html` (bundled with the plugin source). Substitute template placeholders with today's data — section by section. Each annotation field becomes a real `<textarea>` with the configured placeholder hint.

### Canonical 6-section format (v0.4.0 — closes workstream/nucleus-improvements observation 2026-05-21)

The artifact MUST include these sections in this order. Sections with zero content are hidden (cleaner surface) UNLESS the section is required (header, yesterday's reflection).

1. **Sticky header** — date badge, "generated at" timestamp, total counts line ("3 meetings · 5 inbox items · 7 tasks · 4 outreach"), and a progress bar reflecting checked-off priority tasks. Always visible (sticky on scroll).

2. **Day-at-a-glance timeline strip** — horizontal strip with meeting blocks placed by time (8am-6pm scale). Each block links to the corresponding meeting card below on click. Quick visual of "how booked is today." Always visible; renders empty timeline if no meetings.

3. **Meetings card** — per-meeting context blocks. **Read-only** as of v0.3.0 (no annotation textareas; use cortex `/recall <person>` for prep). Hide if zero meetings.

4. **Priority tasks card — INTERACTIVE (v0.4.0)** — P0 tasks only. Each row:
   - **Interactive checkbox** keyed by task ID. State persists in the single JSON-blob localStorage entry described below (path: `brief-YYYY-MM-DD` → field `tasks_checked.<task_id>`).
   - Task title, due date, related contact/deal, priority badge.
   - Per-task annotation textarea (existing behavior).
   - Progress bar in sticky header reflects checked-off ratio in real time.
   - **P1 and lower tasks are NOT shown in this card.** Reduces clutter; surfaces only what matters today.
   - Hide card if zero P0 tasks.

5. **Bizdev outreach queue — TIERED (v0.4.0)** — outreach items grouped into 4 collapsible tiers:
   - **Today** — items scheduled or flagged for today
   - **Next week** — items with `due_date <= today + 7d` or weekly-cadence overdue items
   - **Early next month** — items with `due_date <= today + 35d`
   - **Backlog** — everything else flagged but not date-scoped
   
   Each item: contact name, company, last touch, draft status (drafted / pending / sent). Per-item annotation textarea preserved.
   
   Hide tiers with zero items. Hide the whole card if all tiers empty.

6. **Yesterday's reflection (read-only)** — content from yesterday's `## Reflection` (written by cortex `/end-day` Step 4). Always visible (renders "No reflection logged yesterday" placeholder if missing).

### Interactive state contract (v0.4.0+; canonical shape locked in v0.4.1)

**ONE JSON blob per day, stored in `localStorage` under a single date-rotating key.** All readers and writers use the same shape. No per-task sub-keys, no nested namespaces — just the one blob:

```javascript
// Key: "brief-YYYY-MM-DD" (rotates with the date; yesterday's key is left in place for audit but no longer read)
localStorage.setItem("brief-YYYY-MM-DD", JSON.stringify({
  schema_version: "0.4.1",
  tasks_checked: { "task-12345": true, "task-67890": false },   // task id → bool
  annotations: { "inbox-<thread_id>": "draft reply: ...", "task-12345": "moved to tomorrow" },   // item id → free-form
  outreach_tier_collapsed: { today: false, next_week: false, early_next_month: true, backlog: true },
  last_interaction_at: "2026-05-28T14:32:00-04:00"
}));
```

**Reader pattern (use this exact incantation):**

```javascript
const state = JSON.parse(localStorage.getItem(`brief-${today_local}`) || "{}");
const tasksChecked = state.tasks_checked || {};
const annotations = state.annotations || {};
```

**State persists across the day.** Re-running `/brief` mid-day preserves checkbox state and annotations (see Step 3a below). The day's checked-off-task ratio drives the progress bar in the sticky header.

**State is read by `/end-day`**: cortex `/end-day` Step 4.0 (per `cortex v4.12.2+` spec) reads `state.tasks_checked` and `state.annotations` via `mcp__cowork__read_widget_context(artifact_id="todays-brief")` and uses them to pre-fill the reflection prompts. This closes the loop — the brief is a living surface, not a one-shot snapshot.

**v0.4.1 fix from v0.4.0 review:** earlier versions of this spec described the localStorage shape in three different ways across `brief.md` and `process-brief.md` — `brief-YYYY-MM-DD.tasks.<task_id>` (per-task keys), `brief-YYYY-MM-DD` JSON blob with `tasks_checked` dict, and `brief-YYYY-MM-DD.tasks_checked` (nested key). The canonical shape is the JSON-blob form above. Any earlier reader that expected per-task sub-keys must be updated to read the JSON blob.

Decide create vs. update (v0.4.2+ race-aware):

1. Call `mcp__cowork__list_artifacts` and look for the artifact by id `todays-brief` (canonical id; v0.4.0+).
2. If found, **check its metadata `target_date` field** (v0.4.2+):
   - If `target_date == <intended-date>` (where intended-date is `today_local` for `/brief`, or `tomorrow_local` for cortex `/end-day` Step 5 pre-stage) → safe to update. Call `mcp__cowork__update_artifact(artifact_id="todays-brief", content=<new HTML>, metadata={target_date: <intended-date>, plugin: "daily-brief", schema_version: "0.4.2"})`. **Preserve any non-empty annotation textarea values + tasks_checked state** that match by item ID (best-effort — see Step 3a).
   - If `target_date != intended-date` (e.g., `/brief` runs at 4:55pm while `/end-day` Step 5 is mid-pre-stage of tomorrow's brief), **DO NOT silently overwrite**. Surface to the user:
     > "⚠ Artifact `todays-brief` already exists with `metadata.target_date: <existing>`. About to overwrite with `target_date: <new>`. This indicates a race between `/brief` (today) and `/end-day` Step 5 (tomorrow), or two concurrent `/brief` invocations. Proceed (last-write-wins, may stomp pre-staged tomorrow content), or abort?"
     On `proceed`: update as normal. On `abort`: exit Step 3 without modifying the artifact.
3. If found but its `target_date` is older than `<today_local>` (e.g., yesterday's brief that never got pre-staged) → call `mcp__cowork__update_artifact` with fresh content. The old day's annotations belong to a different day; their final state is already in yesterday's markdown snapshot.
4. If not found → call `mcp__cowork__create_artifact(id="todays-brief", artifact_type="html", content=<rendered>, metadata={target_date: <intended-date>, plugin: "daily-brief", schema_version: "0.4.2"})`.

**Artifact race serialization (v0.4.2+):** the `target_date` field on metadata is the canonical race-arbiter. Both `/brief` and cortex `/end-day` Step 5 check before overwriting. If they conflict, surface to the user — never silently overwrite. This closes the race surfaced in the second-pass review where "artifact shows tomorrow's content labeled with today's date" could happen.

### Step 3a — Preserve annotations across same-day re-runs

When the artifact already exists for today and the user is re-running `/brief`:

1. Call `mcp__cowork__read_widget_context` on the artifact to retrieve the current annotation textarea values.
2. Build a map keyed by item ID (e.g., `meeting-<event_id>`, `inbox-<thread_id>`, `task-<task_id>`).
3. When rendering the new HTML, pre-fill each textarea with its prior value if the same item ID still exists in today's pull.
4. Items that have dropped out of today's data (e.g., a meeting got cancelled between morning and noon) leave their old annotation only as a line in the markdown snapshot's section under "Dropped from today's brief at <regen time>", so the user can still see it.

In Claude Code (no Cowork artifact tools available), this Step 3 entirely degrades to: write the markdown snapshot, print a "Cowork artifact tools not available in this runtime — brief saved as markdown at `<path>`" message, and stop.

---

## Step 4 — Notify the user

Output exactly one chat message:

> "Brief ready for <today_local>. Open the 'Today's Brief' artifact, annotate any sections, then run `/process-brief` when you're done. Markdown snapshot saved at `<config-root>/briefs/<today_local>.md`."

Don't dump the brief content into chat. The whole point is that it lives in the artifact / on disk, not in chat.

---

## Behavior rules

- Read-only across all data sources. No drafts, no task updates, no sends.
- Same-day re-run = refresh + preserve annotations where item IDs match. Different day = new artifact content; yesterday's file is preserved at `briefs/<yesterday>.md`.
- If a source MCP isn't connected (e.g., no HubSpot), that section renders as "[Section name]: source not connected — re-run `/diagnose` or connect the MCP and re-run `/brief`." No silent failure.
- Cost: this command should fit in ~3-5K tokens of synthesis on top of raw connector payloads. Cap each connector payload at a reasonable per-section size (e.g., 12 calendar events max, 7 inbox items max, 12 tasks max).
- Telemetry: optionally log a single line to core-ops `/log-agent-run` recording (skill: brief, item_counts: {meetings, inbox, tasks, outreach}, runtime_ms). Skip if core-ops isn't installed.
