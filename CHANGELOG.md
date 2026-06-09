# Changelog

All notable changes to daily-brief are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/). Versions match `plugin.json`.

## [0.5.0] — End-Day Routine Improvement Spec: 5-section brief, richer actions, surfacing-prefs filter (2026-06-08)

Implements Part A of the End-Day Routine Improvement Spec. Coordinated with cortex v4.13.0.

### Changed — canonical brief is now 5 fixed sections (was 6)

`references/brief-artifact-template.html` rebuilt from Zach's hand-built reference prototype. Fixed section order, identical formatting whether produced by `/brief` or cortex `/end-day` pre-stage:

1. **Center of Gravity** — accent banner, the single most important thing (from yesterday's "one thing that has to move" + top P0). Not interactive.
2. **Calendar Block** — visual timeline strip (8a–6p, blocks positioned by time, color-coded meeting/focus/personal/open) + written block list with per-meeting notes (context / why this matters / last touch from the linked person/client node).
3. **Priority Tasks** — P0/P1 only. Richer per-row actions replace the bare checkbox: ✓ Done · 👉 Delegate (→ who) · ⏭ Skip (→ how long) · 🚫 Not important · 📝 Annotate. Progress bar driven by `done`.
4. **Outreach Queue** — tiered (today / this week / backlog), per-contact actions (Sent / Nudge / Booked / Skip / Let go) + an **optional** category quick-tag (bucket + value-add chips; signal auto-fills from the pipeline entry).
5. **Yesterday's Reflection** — read-only, from yesterday's `## Reflection`.

The old "inbox action items" and "drafts awaiting approval" sections are gone from the canonical surface; the brief is the working surface — mining/drafting happen in `/process-brief` + `/end-day`.

### Template — reconciled to the spec-v2 reference layout (visual source of truth)

`references/brief-artifact-template.html` is now based on the handoff `brief-artifact-template.v2.html` / `todays-brief.reference-2026-06-09.html` (the stated visual source of truth), not the earlier hand-built draft:
- Visual calendar strip is built in JS from a `BLOCKS` array (`{{TL_BLOCKS_JSON}}`, decimal-hour `{s,e,label,cls}` with cls meeting/focus/personal) + `{{TL_START_HOUR}}`/`{{TL_END_HOUR}}` — the skill fills the array rather than emitting pre-positioned divs.
- v2 token scheme (`{{DATE_LONG}}`, `{{CENTER_OF_GRAVITY}}`, `{{EVENT_*}}`, `{{TASK_*}}`, `{{CONTACT_*}}`, `{{REFLECT_*}}`, `{{DATE_ISO}}`, `{{FOOTER_NOTE}}`) + a tokenized `cowork-artifact-meta` block (`{{META_DESCRIPTION}}` / `{{META_MCP_TOOLS}}` / `{{META_MCP_SERVERS}}`).
- Outreach category selects (bucket/signal/value-add) are **always visible**; signal auto-fills by emitting the matching `<option>` first. Outreach buttons are Sent / Nudge / Skip / Let go (Booked dropped from the default render to match the reference; `booked`/`dead` remain reader-accepted synonyms).
- Preserved contracts: per-task annotation textarea (📝 Annotate toggle) keyed by `data-item-id`; `schema_version:"0.5.0"` + `last_interaction_at`; `tasks_checked` back-compat mirror written on every `done`; the research link (below); target_date race guard owned by the skill.

### Added — research link per outreach contact

Each outreach row now shows a research link next to the name — a "🔗 LinkedIn" link when a profile URL is known (from the contact's cortex person/bizdev node or the pipeline entry), else a "🔍 Research" link built as a LinkedIn people-search (or web search) from name + company. Always non-empty so every contact is one click from prep. New template tokens `{{OUTREACH_LINK}}` / `{{OUTREACH_LINK_LABEL}}`; the markdown twin includes a `[research](url)` link too.

### Added — surfacing-prefs filter (suppression learning)

`/brief` now reads `<config-root>/memory/surfacing-prefs.md` (Step 0D) and **filters priority tasks + outreach against the Do-not-resurface list and noise rules before render**. Suppressed items never appear; noise-class items never reach P0/P1. Closes the "brief resurfaces items I've ignored for weeks" complaint.

### Changed — localStorage contract → schema_version 0.5.0 (breaking)

One JSON blob per day at `brief-YYYY-MM-DD`:
- `tasks: {id: {action: done|delegate|skip|not_important, detail, ts, name}}` (was `tasks_checked: {id: bool}`)
- `annotations: {item_id: free-text}`
- `outreach_actions: {id: {name, action: sent|skip|nudge|let_go|booked|dead, bucket, signal, value_add, detail, ts}}`
- `last_interaction_at`

Back-compat: readers map a legacy `tasks_checked` `true` → `{action:"done"}`. `/process-brief` and cortex `/end-day` updated to read the new shape.

### Changed — `/process-brief` handles the new action types

Reads the v0.5.0 blob; routes task actions (done→CRM COMPLETED, delegate→delegatee task, skip→defer, not_important→log for `/end-day` suppression) and outreach actions (sent/nudge→touch, booked→prep task, let_go/dead→log). Intra-day actor; durable memory write-backs + suppression learning are owned by cortex `/end-day` Step 2c.

### Notes

- Markdown twin at `<config-root>/briefs/YYYY-MM-DD.md` mirrors the 5 sections and footers the filtered-item count.
- `/brief` reads `briefs/<date>.seed.json` (written by `/end-day` Steps 4.5/4.6) to seed tomorrow's priorities/outreach.

## [0.4.2] — Artifact race serialization + stale-refs cleanup (2026-05-28)

Followup to v0.4.1 completing the deferred items from the v0.4.0/0.4.1 review punch list. Coordinated with cortex v4.12.3 + relationships v0.2.3.

### Added — Artifact race serialization via target_date metadata

`/brief` Step 3 and cortex `/end-day` Step 5 both write to artifact id `todays-brief`. v0.4.0 had no race protection — last-write-wins could produce confusing state ("artifact shows tomorrow's content labeled with today's date").

v0.4.2 adds `target_date` field on artifact metadata:
- `/brief` writes `target_date: today_local`.
- cortex `/end-day` Step 5 pre-stage writes `target_date: tomorrow_local`.
- Before any update, both readers check existing metadata.target_date. If conflict (existing != intended), surface to user with proceed/abort prompt rather than silent overwrite.
- Same-date overwrites preserve annotation + tasks_checked state per existing Step 3a logic.
- Metadata also carries `schema_version: "0.4.2"` so future readers can detect.

### Changed — Stale-references sweep (continuation from v0.4.1)

Three more files now use current plugin names:
- `commands/setup-brief.md` — Section 1 enable-list, Section 2 sort defaults, and intro paragraph all reference `relationships` (v0.2.0+) with legacy weekly-outreach fallback note.
- `commands/plan-tomorrow.md` — Step 0 companion plugins extract list, Step 2E outreach plan check rewritten to prefer `relationships` first.
- `commands/process-brief.md` — description, Step 2 classification table, Step 3 draft_outreach routing, and Step 3.6 association handling all route to `relationships` `/draft-touchpoint` and `/touchpoint` instead of legacy weekly-outreach.

### Acceptance

- [ ] Artifact metadata.target_date check before overwrite.
- [ ] No live-plugin references to weekly-outreach / bizdev-outreach in setup-brief, plan-tomorrow, process-brief, brief. Only migration-context references remain.
- [ ] `plugin.json` bumped to 0.4.2.

---

## [0.4.1] — Canonical localStorage shape lock + brief.md / process-brief.md alignment (2026-05-28)

Same-day patch fixing the CRITICAL localStorage-shape mismatch surfaced by post-ship review: v0.4.0 described the state shape three different ways across `brief.md:184` (per-task sub-keys), `brief.md:208-214` (JSON blob), and `process-brief.md:41` (nested key path). UIs would have hit silent state-not-found errors.

### Fix

- **Canonical shape locked: single JSON blob at key `brief-YYYY-MM-DD`** containing `{schema_version: "0.4.1", tasks_checked: {<task_id>: bool}, annotations: {<item_id>: str}, outreach_tier_collapsed: {...}, last_interaction_at: iso8601}`. No per-task sub-keys; no nested namespaces.
- **`brief.md` Section 4 corrected** — checkbox state path is now described as "the single JSON-blob localStorage entry described below" (was the per-task sub-key form).
- **`brief.md` Interactive state contract section rewritten** with the canonical shape + the exact JS incantation for reading: `JSON.parse(localStorage.getItem("brief-YYYY-MM-DD") || "{}")`.
- **`process-brief.md` Step 1 rewritten** with the v0.4.1 canonical shape (was the v0.4.0 nested-key description).
- **`schema_version: "0.4.1"` field added** so future readers can gate compatibility. Missing → treat as v0.4.0 (pre-canonical-lock).

### Stale-reference cleanup

- **`brief.md` Section 4 outreach queue:** replaced `weekly-outreach` plugin reference with `relationships` v0.2.0+ (reads from `<config-root>/relationships/today.json`); kept `weekly-outreach` fallback path for migration grace.

### Coordinated with cortex v4.12.2

- Cortex `/end-day` Step 4.0 (NEW in v4.12.2) reads `state.tasks_checked` and `state.annotations` via `mcp__cowork__read_widget_context` using the v0.4.1 canonical shape. Sanitizes annotations (paraphrases, doesn't quote verbatim) before writing to reflection.

### Coordinated with nucleus contracts.md

- contracts.md row for `mcp__cowork__update_artifact(id: "todays-brief")` updated with the canonical localStorage shape + data-flow trace ("annotation content may transit through browser localStorage → cortex /end-day Step 4.0 → reflection → committed memory; /end-day sanitizes").

### Not in this release

- Drag-to-reorder, bulk-action toolbar — still v0.5 candidates.
- `process-brief.md` Step 3.6 (mark-complete in CRM) unchanged — already canonical.

---

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
