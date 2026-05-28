# Changelog

All notable changes to daily-brief are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/). Versions match `plugin.json`.

## [0.4.0] — Interactive brief + canonical 6-section format (2026-05-28)

Closes the brief-interactivity observation logged 2026-05-21 in workstream/nucleus-improvements: "the brief is read-only today; it should be a working surface — check off items, annotate inline, de-prioritize, and have state feed `/end-day`."

### Added — interactive priority tasks
- P0 tasks now have checkboxes keyed by `task-<task_id>`.
- Checkbox state persists in browser localStorage under `brief-YYYY-MM-DD.tasks_checked`.
- Sticky header includes a progress bar reflecting checked-off ratio in real time.
- Only P0 tasks are shown in this card — P1+ are excluded to reduce clutter.

### Added — tiered bizdev outreach queue
- Outreach items grouped into 4 collapsible tiers: Today / Next week (≤ +7d) / Early next month (≤ +35d) / Backlog.
- Each tier collapses independently; collapsed state persists in `brief-YYYY-MM-DD.outreach_tier_collapsed`.
- Empty tiers hidden; whole card hidden if all tiers empty.

### Added — day-at-a-glance timeline strip
- Horizontal time strip (8am–6pm scale) above the meetings card.
- Meeting blocks placed by time; click jumps to corresponding meetings-card item.

### Changed — sticky header
- Date badge, generated-at timestamp, total counts line, progress bar.
- Sticky on scroll.

### Changed — canonical artifact id
- Artifact id is ALWAYS `todays-brief`. Both `/brief` and cortex `/end-day` Step 5 reference the same persistent surface. Closes the format-inconsistency observation from 2026-05-21.

### Changed — /process-brief reads checked dictionary
- Step 1 reads both annotations AND `tasks_checked` from widget context.
- New Step 3.6: for every checked-off task, queue HubSpot status: COMPLETED with batch confirmation table.

### Cortex contract (cortex v4.12.0)
- Cortex `/end-day` Step 4 reads the checked dictionary via `read_widget_context`. The brief feeds `/end-day` — closes the working-surface loop.
- Cortex `/end-day` Step 5 references this v0.4.0 format as canonical render target.

### Canonical 6-section format (locked)
1. Sticky header (always visible)
2. Day-at-a-glance timeline strip (always visible)
3. Meetings card (read-only; hide if empty)
4. Priority tasks card (interactive — P0 only, checkboxes + progress bar; hide if empty)
5. Bizdev outreach queue (tiered; hide if all tiers empty)
6. Yesterday's reflection (read-only; always visible — placeholder if missing)

### Out of scope for v0.4
- De-prioritize on-the-fly (drag to reorder)
- Multi-day state persistence (rotates by date)
- Bulk-action toolbar

---

## [0.3.0] — Cleanup pass: remove brief overhead (2026-05-16)

### Why this exists
Real-user feedback: `/brief` and `/process-brief` "feel like overhead" — the artifact suggests there's lots to act on, but most of the annotation fields don't route to anything actionable. Empty motions train you to ignore the ritual. This release cuts the no-op surfaces so what's left is the actionable spine.

### Removed
- **Section 7 (end-of-day prompts) is gone from `/brief`.** Three reflection textareas at brief-generation time were dead weight — you don't fill them until end-of-day, and only after running `/end-day`. Reflection capture is now owned by cortex `/end-day` Step 4 (which appends a `## Reflection` section to the day's brief markdown). Tomorrow's `/brief` Section 6 reads from there.
- **Meeting annotation textareas removed.** Meeting cards are now read-only context cards. The old `add_talking_point` annotation appended a bullet to the markdown snapshot — not useful enough to justify a textarea per meeting. Use cortex `/recall <person>` for prep context instead.
- **`add_talking_point` action removed from `/process-brief` classifier.** Three real actions remain (`draft_reply`, `reschedule_task`, `draft_outreach`) plus `dismiss` and `clarify`.

### Changed
- **Empty sections now auto-hide in the artifact.** If you have zero meetings, the meetings section doesn't render at all. Same for inbox, tasks, outreach. The brief shrinks to what's actually present today instead of carrying empty placeholders.
- **Section 5 (drafted replies awaiting approval) hides when empty.** Only renders after `/process-brief` has produced drafts today.

### Migration
- No user action required. Re-running `/brief` regenerates the markdown snapshot and artifact in the new shape.
- Yesterday's brief Section 7 entries (if any) are still preserved in their markdown snapshots — the v0.3.0 brief just doesn't generate new ones.
- `daily-brief.user-context.md` is unchanged; no setup migration.

### Why this matters
The brief should be a working surface where every visible item is something you can act on. Empty annotation fields with no downstream action erode trust in the artifact. After this release, what's on-screen is what the system can actually do — and the ritual stops feeling like overhead.

## [0.2.0] — Absorbed plan-tomorrow (2026-05-12)

### Added
- **`/plan-tomorrow` command + skill folded in from the deprecated standalone plan-tomorrow plugin.** Same data sources (CRM tasks, cortex memory, inbox action items, calendar) the daily-brief was already pulling — different verb, different output (creates calendar blocks for the next business day). Pulls together the daily ops loop in one plugin instead of two.
- **`/setup-plan` command + skill** moved alongside, configures /plan-tomorrow's working hours, CRM ownership, calendar conventions.
- **User-context backward compatibility.** /plan-tomorrow continues reading `<config-root>/plugins/plan-tomorrow.user-context.md` exactly as it did in the standalone plugin — no migration required for existing users.

### Why this matters
Daily-brief and plan-tomorrow shared ~90% of their data sources. Two plugins for one ops loop added decision cost ("which do I run?") without adding capability. Folding them keeps both verbs (today's working surface + tomorrow's calendar planning) but reduces the marketplace catalog by one and clarifies the mental model.

The standalone `BrightWayAI/plan-tomorrow` GitHub repo is being archived (read-only). Anyone who had it installed via the marketplace will see the deprecation notice; new installs should use daily-brief.

## [0.1.0] — Initial release (2026-05-12)

### Added
- Daily-brief plugin — live daily working surface as a Cowork artifact, with a per-day markdown snapshot for audit.
- **`/brief`** — pulls calendar, inbox action items, CRM priority tasks, outreach queue, yesterday's reflection. Renders an HTML artifact titled "Today's Brief" with a `<textarea>` per item for annotations. Writes a markdown snapshot to `<config-root>/briefs/YYYY-MM-DD.md`. Read-only across all sources.
- **`/process-brief`** — reads the artifact's annotation values via `mcp__cowork__read_widget_context`. Two-stage triage: Haiku-class classifier assigns each annotation to a normalized action (`draft_reply`, `reschedule_task`, `add_talking_point`, `dismiss`, `clarify`); Sonnet-tier synthesis only runs on drafting actions. Drafts go to Gmail; CRM tasks get rescheduled; meeting talking points get appended to the markdown snapshot.
- **`/setup-brief`** — captures section toggles, sort defaults, empty-state behavior, annotation placeholder hints, optional auto-run schedule. Writes results to `<config-root>/plugins/daily-brief.user-context.md`.
- **HTML artifact template** at `references/brief-artifact-template.html` — seven sections (meetings, inbox, tasks, outreach, drafts-awaiting-approval, yesterday's reflection, end-of-day prompts). Each section's items render with their own `<textarea>` keyed by item ID. A "Process all annotations" button calls `window.sendPrompt` to trigger `/process-brief` from inside the artifact.
- **Cross-plugin integration** — pulls inbox items from `inbox-triage` (Phase 2 plugin) when installed; falls back to direct Gmail otherwise. Pulls outreach queue from `weekly-outreach` when installed. Pulls meeting attendee context from cortex `person/` and `client/` nodes when available.
- **Same-day annotation preservation** — re-running `/brief` on the same day reads existing annotation textareas via `read_widget_context` and pre-fills the new render with prior annotation values where item IDs still match.
- **Platform-aware** — works in both Cowork (full artifact lifecycle) and Claude Code (markdown snapshot only; a single clear message explains the artifact-tool limitation).

### Why this exists
Phase 1 of SECOND-BRAIN-V2-SPEC. The brief is the surface that turns cortex memory + connector context into action. Before this plugin, cortex memory existed and the per-domain plugins existed, but there was no daily working surface that pulled it all together. Now there is.
