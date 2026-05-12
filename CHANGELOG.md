# Changelog

All notable changes to daily-brief are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/). Versions match `plugin.json`.

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
