# daily-brief user context (TEMPLATE)

_Run `/setup-brief` to generate your real `<config-root>/plugins/daily-brief.user-context.md` (gitignored)._

_Last updated: [filled by setup]_

## Feature toggles
- **brief_enabled:** [true / false — master switch; if false, `/brief` exits cleanly]
- **auto_open_artifact:** [true / false — when /brief runs, also open the artifact tab; default false]

## Sections enabled
List section numbers (1-7) the user wants rendered. Default: all seven.
- [1] Today's meetings
- [2] Inbox action items
- [3] Today's priority tasks
- [4] Outreach queue
- [5] Drafted replies awaiting approval
- [6] Yesterday's reflection
- [7] End-of-day prompts

## Sort defaults
- **meetings:** [start_time_asc / start_time_desc — default start_time_asc]
- **tasks:** [priority_then_due / due_then_priority / created_desc — default priority_then_due]
- **inbox:** [triage_score_desc / received_desc — default triage_score_desc]
- **outreach:** [accept_upstream — default; pre-sorted by weekly-outreach]

## Empty-state behavior
For each enabled section, what to do when its source returns zero items:
- **meetings:** [show_empty / hide — default show_empty]
- **inbox:** [show_empty / hide — default show_empty]
- **tasks:** [show_empty / hide — default show_empty]
- **outreach:** [show_empty / hide — default hide (often empty on non-outreach days)]
- **yesterday_reflection:** [show_empty / hide — default show_empty]

## Annotation placeholder hints
Text shown inside empty textareas as ghost-text instruction.
- **meeting_items:** "e.g., add talking point: X / move 15 min later / cancel"
- **inbox_items:** "e.g., draft reply: short ack + my view / move to tomorrow / skip"
- **task_items:** "e.g., move to tomorrow / mark done / reassign to [name]"
- **outreach_group:** "e.g., skip Marcus this week / draft all / pick 3"
- **eod_prompts:** (no hint — free text)

## Auto-run schedule
- **enabled:** [true / false — default false]
- **time:** [HH:MM in user's time zone — default 07:30]
- **weekdays:** [Mon-Fri default; or list of days]

## Per-section caps
Maximum items rendered per section (everything beyond falls into a "+N more" tail).
- **meetings:** [12 default]
- **inbox:** [7 default]
- **tasks:** [12 default]
- **outreach:** [10 default]
