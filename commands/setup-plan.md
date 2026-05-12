---
description: Configure plan-tomorrow for your CRM, working hours, calendar conventions, and companion-plugin integrations. Writes results to <config-root>/plugins/plan-tomorrow.user-context.md so the planning command can adapt to your setup. Re-run anytime to update.
---

# /setup-plan

Short interview that captures the context plan-tomorrow needs to actually be useful for you.

---

## Step 0 — Resolve plugin config root

Per-plugin config in this marketplace lives under a user-chosen folder, recorded at `~/Documents/.claude-plugin-config-root` (single-line text file in the user's home).

### A — Try the pointer

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.
- **Exists**: read line 1 → that's the config root path. Ensure access to `<config-root>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<config-root>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Skip to section C.
- **Missing**: continue to section B.

### B — First-time bootstrap

Prompt: "First-time plugin setup. Where should I store your plugin config — identity, voice, and per-plugin settings? Pick a folder you control (e.g., `~/Documents/Claude/` or `~/Documents/PluginConfig/`). The folder will hold `identity.md`, `voice.md`, and a `plugins/` subdirectory."

Then:
1. Ensure access to `<path>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<path>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Create `<path>/plugins/`. Write absolute path to `~/Documents/.claude-plugin-config-root`.

### C — Read shared identity

Read `<config-root>/identity.md` (cortex's `/setup-identity` output).
- **Populated** → pre-fill Section 1 (Identity) and the primary-tool questions in Section 3 (CRM, calendar). Skip those; just confirm what you read.
- **Missing** → offer `/setup-identity` first or proceed inline.

For the rest of this document, **`<config-root>`** refers to the resolved path. This plugin's config file lives at **`<config-root>/plugins/plan-tomorrow.user-context.md`**.

---

## Step 1 — Check for existing config

Read `<config-root>/plugins/plan-tomorrow.user-context.md` if it exists.

- Populated → ask: "Update sections, or start over?"
- Missing → start fresh.

---

## Step 2 — The interview

One section at a time. Confirm before moving on.

### Section 1 — Identity
- Your name
- Your company
- One-sentence description of what you do

### Section 2 — Working hours
- Start of workday (e.g., 9:00 AM)
- End of workday (e.g., 5:00 PM)
- Time zone (e.g., America/New_York)
- Default block size for quick tasks (default 30 min)
- Default block size for deep work (default 60–90 min)
- Max task blocks per day (default 6)

### Section 3 — CRM
- Which CRM? (HubSpot / Salesforce / Pipedrive / Attio / other / "none")
- If "none" → note that CRM-task pulls will be skipped.
- Your CRM owner ID (if applicable)
- Status names that mean "needs work" (e.g., "NOT_STARTED, IN_PROGRESS")
- Priority field name (e.g., "hs_task_priority")
- Priority values (e.g., HIGH, MEDIUM, LOW)

### Section 4 — Calendar conventions
- Color ID for task blocks (Google Calendar uses "1"–"11"; default "5" Banana yellow)
- Emoji map for block types — defaults: deep work 🔨, emails 📧, meeting prep 📝, capture 📋, buffer 🔲
- Send-updates default — usually "none" for personal blocks (recommended; otherwise attendees get spammed)

### Section 5 — Companion plugin integrations
- Is `claude-cortex` installed? (Y/N — drives whether memory is read)
- Is `core-ops` installed? (Y/N — drives whether `pipeline-analyst` is delegated to)
- Is `weekly-outreach` installed? (Y/N — drives whether outreach contacts are pulled from the weekly queue)

### Section 6 — Optional preferences
- Always include a buffer block? (Y/N, default Y)
- Always include post-call capture if external meetings happened? (Y/N, default Y)
- Default outreach block size if applicable (default 30 min)
- Anything else that should be a block on every day? (e.g., "30 min reading first thing")

---

## Step 3 — Write the config

Populate `<config-root>/plugins/plan-tomorrow.user-context.md`:

```markdown
# plan-tomorrow user context

_Last updated: [date]_

## Identity
- **Name:** ...
- **Company:** ...
- **What we do:** ...

## Working hours
- **Start:** ...
- **End:** ...
- **Time zone:** ...
- **Default quick-task block:** ... min
- **Default deep-work block:** ... min
- **Max blocks per day:** ...

## CRM
- **Tool:** ...
- **Owner ID:** ...
- **Active-status values:** ...
- **Priority field:** ...
- **Priority values:** ...

## Calendar
- **Task-block color ID:** ...
- **Emoji map:**
  - Deep work: ...
  - Emails: ...
  - Meeting prep: ...
  - Capture: ...
  - Buffer: ...
- **Send-updates default:** none

## Companion plugins
- **claude-cortex:** [installed / not installed]
- **core-ops (pipeline-analyst):** [installed / not installed]
- **weekly-outreach:** [installed / not installed]

## Preferences
- **Include buffer block:** ...
- **Include post-call capture:** ...
- **Outreach block size:** ...
- **Always-on blocks:** ...
```

---

## Step 4 — Confirm and offer next step

Summarize what was saved (one short paragraph). Offer:
> "Try `/plan-tomorrow` to plan your next workday."

---

## Behavior rules

- One section at a time. Don't bombard.
- Skip what doesn't apply. "I don't have a CRM" is a valid answer.
- Idempotent. Re-running updates without re-doing finished sections.
