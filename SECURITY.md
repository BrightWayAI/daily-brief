# Security Policy

## What this plugin does with your data

Daily-brief turns your existing connectors (calendar, Gmail, CRM, cortex memory) into a single live working surface — a Cowork artifact titled "Today's Brief" — plus a markdown snapshot per day.

**Reads:**
- **Calendar MCP** — today's events, attendees, locations.
- **Gmail MCP** — recent threads, sender, subject, snippet (via direct fallback or, if installed, the `inbox-triage` plugin which classifies before passing the top 3-7).
- **HubSpot MCP (or other CRM)** — tasks owned by you, due today / overdue.
- **Plugin references** — `<config-root>/plugins/daily-brief.user-context.md` for section toggles, sort defaults, placeholder hints. Read-only.
- **Other plugin configs** — `<config-root>/plugins/weekly-outreach.*` (if installed) for the outreach queue. Read-only.
- **Cortex memory** — `<config-root>/memory/DASHBOARD.md` and any `person/`, `client/`, `bizdev/` node files referenced by today's meetings or inbox. Read-only.
- **Yesterday's brief snapshot** — `<config-root>/briefs/<yesterday>.md`. Read-only.
- **Shared identity** — `<config-root>/identity.md` for time zone and tool inventory. Read-only.
- **Cowork artifact state** — `mcp__cowork__read_widget_context` on the "Today's Brief" artifact, to retrieve annotation textarea values during `/process-brief` and during same-day `/brief` re-runs.

**Writes:**
- **Markdown snapshot** — `<config-root>/briefs/YYYY-MM-DD.md` per day (one file per day). Overwrites today's file on same-day re-run; never touches prior days.
- **Cowork artifact** — `mcp__cowork__create_artifact` (first run of the day) or `mcp__cowork__update_artifact` (subsequent runs and during `/process-brief`). One artifact titled "Today's Brief"; its content rotates to today's data.
- **Plugin user-context** — `<config-root>/plugins/daily-brief.user-context.md` (after `/setup-brief`).
- **Gmail drafts** — `/process-brief` creates Gmail drafts in response to `draft_reply` annotations. Drafts only; never sends.
- **CRM task due dates** — `/process-brief` updates task due dates in response to `reschedule_task` annotations. Date-only updates; never marks complete, never deletes, never creates new tasks.
- **Dismissed log** — `<config-root>/plugins/daily-brief.dismissed-log.md` (append-only). One line per dismissed annotation; first 80 chars of annotation text plus item ID.

**Does not:**
- **Send any email.** Every reply is a Gmail draft for user review.
- **Mark CRM tasks complete or delete them.** Only updates due dates.
- **Modify cortex memory or yesterday's brief.** Yesterday's `briefs/<yesterday>.md` is read-only; the brief never writes back into cortex memory.
- **Move data off your machine** beyond what your authorized connectors already send to their cloud services (Gmail, HubSpot, Cowork).
- **Run scheduled tasks.** Only registers them via core-ops `/register-schedules` if the user opts in during setup; execution is owned by Cowork's scheduled-tasks system.

## Where data lives

- Markdown snapshots at `<config-root>/briefs/YYYY-MM-DD.md` (your machine, in the folder you chose during first-time setup).
- Cowork artifact content lives in Cowork's storage (per Cowork's security model).
- Plugin user-context at `<config-root>/plugins/daily-brief.user-context.md` (your machine).
- Dismissed log at `<config-root>/plugins/daily-brief.dismissed-log.md` (your machine, append-only).

## Privacy considerations

- The markdown snapshot contains email snippets, contact names, and CRM task titles. If you sync `<config-root>` to a git repo, gitignore `briefs/` and `plugins/daily-brief.*` files.
- Person-page lookups (Phase 3 of the second-brain spec, not yet shipped) will deepen the cross-reference context. Same gitignore recommendation applies.
- The artifact's annotation textareas are visible to anyone who can view your Cowork workspace. Treat them like any internal-only note.

## What gets sent off your machine

- Whatever your authorized calendar / Gmail / CRM connectors send when invoked.
- Cowork artifact content goes to Cowork's artifact-storage service per Cowork's security model.
- Nothing else.

## Supported versions

| Version | Supported |
|---------|-----------|
| 0.1.x   | Yes       |

## Reporting a vulnerability

Report privately via GitHub Security Advisories:

https://github.com/BrightWayAI/daily-brief/security/advisories/new

Do not open a public issue for security concerns. We aim to respond within 5 business days.
