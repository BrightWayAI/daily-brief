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
- Skip blocked time and personal events the user has flagged personal (configurable later — for v1, include all events; the artifact has a per-item annotation that can say "personal — hide").

Sort per config (default: `start_time_asc`).

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

This section is populated by `/process-brief`. On a fresh `/brief` run, it's empty. Render an empty placeholder card so the layout is stable: "Drafts will appear here after you run `/process-brief`."

### Section 6 — Yesterday's reflection

Read `<config-root>/briefs/<yesterday_local>.md` if it exists.

- **Exists** → extract the "End-of-day prompts" responses if present. Display read-only: what got done, what's open, what to start with tomorrow.
- **Missing** → "No reflection logged yesterday."

### Section 7 — End-of-day prompts

Three read-only placeholders during the morning brief generation. They become writable inputs when `/end-day` is run later. For now: render three empty textareas with placeholders ("What got done today?", "What's still open?", "What's the first thing tomorrow?").

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
- **Annotation:** _(empty — filled in artifact)_

(repeat per meeting)

## 2. Inbox action items

### <sender> — <subject>
- **Received:** <timestamp>
- **Snippet:** <first 200 chars>
- **Suggested framing:** <1-2 lines from inbox-triage if available>
- **Annotation:** _(empty)_

(repeat)

## 3. Priority tasks

| Task | Due | Related | Priority |
|---|---|---|---|
| ... | ... | ... | ... |

- **Annotation per task:** _(empty)_

## 4. Outreach queue

(list — or "No outreach scheduled today")

## 5. Drafted replies awaiting approval

_(populated by /process-brief)_

## 6. Yesterday's reflection

<content from yesterday's brief — or "No reflection logged yesterday">

## 7. End-of-day prompts

- **What got done today?** _(empty)_
- **What's still open?** _(empty)_
- **What's the first thing tomorrow?** _(empty)_
```

Create the `<config-root>/briefs/` directory if it doesn't exist. Overwrite today's file if it already exists (re-runs replace; yesterday's file is untouched).

---

## Step 3 — Render the Cowork artifact

Load the HTML template from `references/brief-artifact-template.html` (bundled with the plugin source). Substitute template placeholders with today's data — section by section. Each annotation field becomes a real `<textarea>` with the configured placeholder hint.

Decide create vs. update:

1. Call `mcp__cowork__list_artifacts` (or the equivalent listing tool the runtime exposes) and look for an existing artifact titled exactly "Today's Brief".
2. If found and its stored date metadata equals `<today_local>` → call `mcp__cowork__update_artifact` with the new HTML, **preserving any non-empty annotation textarea values** that match by section + item ID (best-effort — see Step 3a below).
3. If found but its date is older → call `mcp__cowork__update_artifact` with fresh content. The old day's annotations are not preserved (they belong to a different day); their final state is already in yesterday's markdown snapshot.
4. If not found → call `mcp__cowork__create_artifact` with title "Today's Brief", `artifact_type: "html"`, content = the rendered template, and metadata `{date: <today_local>, plugin: "daily-brief"}`.

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
