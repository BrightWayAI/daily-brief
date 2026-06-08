# daily-brief user context (TEMPLATE)

_Run `/setup-brief` to generate your real `<config-root>/plugins/daily-brief.user-context.md` (gitignored)._

_Last updated: [filled by setup]_

## Feature toggles
- **brief_enabled:** [true / false — master switch; if false, `/brief` exits cleanly]
- **auto_open_artifact:** [true / false — when /brief runs, also open the artifact tab; default false]

## Sections (v0.5.0 — fixed order, 5 sections)
The canonical brief renders 5 fixed sections in this order (End-Day Routine Improvement Spec Part A). Center of Gravity and Yesterday's Reflection always render; the other three hide when empty.
- [1] Center of Gravity (always; not interactive)
- [2] Calendar Block (visual timeline + written list with notes; hide if zero meetings)
- [3] Priority Tasks (P0/P1 only; richer actions; hide if zero)
- [4] Outreach Queue (tiered; per-contact actions + optional tags; hide if empty)
- [5] Yesterday's Reflection (always; read-only)

## Sort defaults
- **meetings:** [start_time_asc / start_time_desc — default start_time_asc]
- **tasks:** [priority_then_due / due_then_priority — default priority_then_due]
- **outreach:** [accept_upstream — default; pre-sorted by the relationships/lead-engine pipeline]

## Empty-state behavior
- **calendar:** [show_empty / hide — default hide]
- **tasks:** [show_empty / hide — default hide]
- **outreach:** [show_empty / hide — default hide]
- (Center of Gravity + Yesterday's Reflection always render their placeholder.)

## Annotation placeholder hints
Ghost-text shown in a task's annotation textarea (revealed by the 📝 Annotate action).
- **task_items:** "e.g., draft reply / move to tomorrow / context note"

## Source review & cost gate (cortex /end-day Step 0.7)
Default-checked sources for the end-of-day review card. (Today's Brief is always required & first.)
- **end_day.sources:** [transcripts, email, crm — default; add `slack` to include it by default]

## Auto-run schedule
- **enabled:** [true / false — default false]
- **time:** [HH:MM in user's time zone — default 07:30]
- **weekdays:** [Mon-Fri default; or list of days]

## Per-section caps
Maximum items rendered per section (everything beyond falls into a "+N more" tail).
- **meetings:** [12 default]
- **tasks:** [12 default]
- **outreach:** [10 default]
