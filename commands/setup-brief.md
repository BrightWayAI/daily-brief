---
description: Configure the daily-brief plugin — section order, what to show when sources are empty, default sort, annotation placeholder text. Writes results to `<config-root>/plugins/daily-brief.user-context.md` (where `<config-root>` is the folder you chose during first-time setup, stored at `~/Documents/.claude-plugin-config-root`). Re-run anytime to update.
---

# /setup-brief

Short interview that configures your daily brief: sort defaults, empty-state behavior, the task annotation hint, and the default source set for `/end-day`'s review-and-cost card. (The 5-section layout itself is fixed as of v0.5.0.)

This plugin assumes identity, voice, and core-ops / relationships / lead-engine setups are already done (or skipped). It doesn't re-ask those questions.

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

### Section 1 — Section model (v0.5.0)

The canonical brief is **5 fixed sections in a fixed order** (End-Day Routine Improvement Spec Part A). The order is not user-configurable; only the empty-content sections hide automatically:

1. **Center of Gravity** — the single most important thing today (always; not interactive)
2. **Calendar Block** — visual timeline strip + written list with per-meeting notes (hide if zero meetings)
3. **Priority Tasks** — P0/P1 only, richer actions (done/delegate/skip/not_important/annotate) (hide if zero)
4. **Outreach Queue** — tiered (today / this week / backlog), per-contact actions + optional tags (hide if empty)
5. **Yesterday's Reflection** — read-only (always)

There's nothing to enable/disable; confirm the user understands the fixed model and move on. (No `sections_enabled` list in v0.5.0.)

### Section 2 — Sort defaults

- Meetings: `start_time_asc` (default) | `start_time_desc`
- Tasks: `priority_then_due` (default) | `due_then_priority`
- Outreach queue: comes pre-sorted from the relationships/lead-engine pipeline; default `accept_upstream`

### Section 3 — Empty-state behavior

For the auto-hiding cards (calendar / tasks / outreach), ask what should happen when the source returns zero items:

- `hide` (default) — collapse the card entirely (cleaner surface)
- `show_empty` — render the card with a short "Nothing today" placeholder line

Center of Gravity and Yesterday's Reflection always render.

### Section 4 — Annotation placeholder + source-review defaults

The only annotation textarea is on priority tasks (revealed by 📝 Annotate). Default hint:

- Task items: "e.g., draft reply / move to tomorrow / context note"

Also ask which sources should be **default-checked** in cortex `/end-day`'s Step 0.7 review-and-cost card (Today's Brief is always required & first):

- `end_day.sources:` default `[transcripts, email, crm]`; offer to add `slack`.

Ask: "Want to customize the placeholder hint or the default source set, or accept defaults?"

### Section 5 — Brief auto-run

Should `/brief` register a scheduled task to run automatically each weekday morning?

- `no` (default) — user runs `/brief` manually
- `yes` — register via core-ops `/register-schedules` if installed; otherwise note that scheduling is unavailable in this install

If yes, ask for time (default 7:30am in the user's time zone) and weekdays (default Mon-Fri).

### Section 6 — Feature toggles

Single-line yes/no:

- `brief_enabled: true` — master switch
- `auto_open_artifact: false` — when /brief runs, also open the artifact tab (Cowork only; default off so the brief regenerates silently)

### Section 7 — Enable brief auto-sync (Cowork only, v0.6.1)

The brief artifact records your done/skip/not-important clicks in its own sandboxed storage. To get them out automatically (so `/end-day` reads them without asking), the artifact needs a **filesystem MCP server** it can call — the Cowork artifact sandbox has no built-in file access.

1. Check whether a file-write MCP tool is already connected (e.g. `mcp__filesystem__write_file`). If yes → tell the user auto-sync is already available; done.
2. If not, explain the trade-off in one line and offer setup:

> "Optional: brief auto-sync. Without it, you click 🔄 Sync for end-day in the brief before closing your day (one extra click). With it, every action in the brief lands on disk instantly. Enabling it means adding a filesystem MCP server scoped to `<config-root>` — want the config snippet?"

3. If yes, have them add the reference filesystem server to the Claude desktop app's MCP settings (Settings → Extensions/Connectors, or `claude_desktop_config.json` → `mcpServers`), **scoped to `<config-root>` only** — never a broader root:

```json
"filesystem": {
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "<config-root>"]
}
```

4. After an app restart, re-run `/brief` — Step 3.0 verifies the tool and switches the artifact to auto-sync (status line shows "Synced for end-day · <time>" after each action instead of the manual-sync banner).

Skippable; the manual Sync button + `/end-day`'s paste path work without it.

---

## Step 3 — Write the user-context file

Populate `<config-root>/plugins/daily-brief.user-context.md` from the template in `references/user-context.template.md`. Fill in the captured values; leave commented placeholders for anything skipped. Create the parent `plugins/` directory if it doesn't exist.

Also create `<config-root>/briefs/` if it doesn't exist (where the markdown snapshots will land).

---

## Step 4 — Confirm and offer next step

Summarize what was captured (which sections enabled, sorts, auto-run choice). Then offer:

> "Daily brief configured. Run `/brief` now to generate today's first one — it'll create the 'Today's Brief' artifact (stable id `todays-brief`). Act on tasks/outreach inline (those feed `/end-day`), annotate anything you want drafted, then run `/process-brief`."

---

## Behavior rules

- Idempotent. Re-running updates only the sections the user wants to update.
- Never silently overwrite. If a value would change, confirm with the user.
- Section toggles are simple bools, not arrays of conditions. Keep config simple.
- Default behavior is conservative: all sections on, all empty states shown, no auto-run, master toggle on.
