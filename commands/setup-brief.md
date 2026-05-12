---
description: Configure the daily-brief plugin — section order, what to show when sources are empty, default sort, annotation placeholder text. Writes results to `<config-root>/plugins/daily-brief.user-context.md` (where `<config-root>` is the folder you chose during first-time setup, stored at `~/Documents/.claude-plugin-config-root`). Re-run anytime to update.
---

# /setup-brief

Short interview that configures the seven sections of your daily brief artifact: which to show, how to sort them, what the annotation hints say, what to do when a section's source is empty.

This plugin assumes identity, voice, and core-ops / weekly-outreach / lead-engine setups are already done (or skipped). It doesn't re-ask those questions.

---

## Step 0 — Resolve plugin config root

Per-plugin config in this marketplace lives under a user-chosen folder, recorded at `~/Documents/.claude-plugin-config-root` (a single-line text file in the user's home directory). Resolve it before doing anything else.

### A — Try the pointer

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.

- **Pointer exists**: read line 1 → that's the config root path. Ensure access to `<config-root>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<config-root>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Skip to section C.
- **Pointer missing**: continue to section B.

### B — First-time bootstrap

This is the user's first plugin setup of any kind. Prompt:

> "First-time plugin setup. Where should I store your plugin config — identity, voice, and per-plugin settings? Pick a folder you control. Examples: `~/Documents/Claude/` (a common pick — and where cortex memory already writes if you have it installed) or `~/Documents/PluginConfig/` or any other path you prefer. The folder will hold one `identity.md`, one `voice.md`, and a `plugins/` subdirectory with one file per plugin you set up."

Once the user provides the path:

1. Ensure access to `<path>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<path>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed — proceed to read or write the file.
2. Create `<path>/plugins/` if it doesn't exist.
3. Write the absolute path to `~/Documents/.claude-plugin-config-root`.
4. Confirm: "Saved. All marketplace plugin configs will live under `<path>` from now on. You can change this later by editing `~/Documents/.claude-plugin-config-root` directly."

### C — Read shared identity (light touch)

Read `<config-root>/identity.md` if present. This setup doesn't need most of it — but the user's time zone matters for "today" calculations, and the configured calendar / email / CRM tools influence which sections this plugin actually has data for. If `identity.md` is missing, prompt the user to run cortex's `/setup-identity` first (preferred) or skip and accept defaults.

For the rest of this document, **`<config-root>`** refers to the resolved path. This plugin's config file lives at **`<config-root>/plugins/daily-brief.user-context.md`**, and the brief markdown snapshots live at **`<config-root>/briefs/YYYY-MM-DD.md`**.

---

## Step 1 — Check for existing config

Read `<config-root>/plugins/daily-brief.user-context.md` if it exists.

- If it exists and is populated → ask: "You've already configured daily-brief. Want to update specific sections, or start over?"
  - "Update [section]" → jump to that section, ask only those questions, write back.
  - "Start over" → continue full interview.
- If it doesn't exist → start fresh. Read `references/user-context.template.md` (bundled with the plugin source) for the structure to populate.

---

## Step 2 — The interview

Ask one section at a time. Confirm before moving on. Don't bombard.

### Section 1 — Section enable / disable

Read out the seven sections in order. Ask which to enable. Default is all seven on; the user can turn any off.

1. Today's meetings (calendar + cortex context per attendee)
2. Inbox action items (Gmail / inbox-triage if installed)
3. Today's priority tasks (CRM, due today / overdue, owner = you)
4. Outreach queue (weekly-outreach, if installed)
5. Drafted replies awaiting approval (filled by `/process-brief`)
6. Yesterday's reflection (read-only display from `briefs/<yesterday>.md`)
7. End-of-day prompts (filled by `/end-day` chain, three text inputs)

Capture: `sections_enabled: [1,2,3,4,5,6,7]` (or a subset).

### Section 2 — Sort defaults

Within each section, ask the sort preference:

- Meetings: `start_time_asc` (default) | `start_time_desc`
- Tasks: `priority_then_due` (default) | `due_then_priority` | `created_desc`
- Inbox items: `triage_score_desc` (default) | `received_desc`
- Outreach queue: comes pre-sorted from weekly-outreach; default `accept_upstream`

### Section 3 — Empty-state behavior

For each enabled section, ask what should happen if its source returns zero items:

- `hide` — collapse the card entirely
- `show_empty` (default) — render the card with a short "Nothing today" placeholder line

This matters most for the outreach queue and yesterday's reflection (which won't exist on day 1).

### Section 4 — Annotation placeholders

Each annotation textarea shows placeholder hint text. Defaults:

- Meeting items: "e.g., add talking point: X / move 15 min later / cancel"
- Inbox items: "e.g., draft reply: short ack + my view / move to tomorrow / skip"
- Task items: "e.g., move to tomorrow / mark done / reassign to [name]"
- Outreach group: "e.g., skip Marcus this week / draft all / pick 3"
- End-of-day prompts: free-text, no hint

Ask: "Want to customize any of the placeholder hint text, or accept defaults?"

### Section 5 — Brief auto-run

Should `/brief` register a scheduled task to run automatically each weekday morning?

- `no` (default) — user runs `/brief` manually
- `yes` — register via core-ops `/register-schedules` if installed; otherwise note that scheduling is unavailable in this install

If yes, ask for time (default 7:30am in the user's time zone) and weekdays (default Mon-Fri).

### Section 6 — Feature toggles

Single-line yes/no:

- `brief_enabled: true` — master switch
- `auto_open_artifact: false` — when /brief runs, also open the artifact tab (Cowork only; default off so the brief regenerates silently)

---

## Step 3 — Write the user-context file

Populate `<config-root>/plugins/daily-brief.user-context.md` from the template in `references/user-context.template.md`. Fill in the captured values; leave commented placeholders for anything skipped. Create the parent `plugins/` directory if it doesn't exist.

Also create `<config-root>/briefs/` if it doesn't exist (where the markdown snapshots will land).

---

## Step 4 — Confirm and offer next step

Summarize what was captured (which sections enabled, sorts, auto-run choice). Then offer:

> "Daily brief configured. Run `/brief` now to generate today's first one — it'll create a Cowork artifact titled 'Today's Brief' that you'll see in the artifacts list. Open it, annotate any sections, then run `/process-brief` to act on the annotations."

---

## Behavior rules

- Idempotent. Re-running updates only the sections the user wants to update.
- Never silently overwrite. If a value would change, confirm with the user.
- Section toggles are simple bools, not arrays of conditions. Keep config simple.
- Default behavior is conservative: all sections on, all empty states shown, no auto-run, master toggle on.
