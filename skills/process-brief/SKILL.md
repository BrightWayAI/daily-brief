---
name: process-brief
description: Read the textarea annotations the user wrote into today's "Today's Brief" Cowork artifact, classify each instruction, and route it — draft reply via Gmail, move CRM task due date, append a meeting talking point, or dismiss. Auto-fires on "/process-brief", "process my brief", "act on my annotations", "process today's annotations", "follow up on my brief". Drafts only — never sends, never marks complete, never deletes.
---

See `commands/process-brief.md` for the full workflow.

## When this skill fires

- User runs `/process-brief` directly
- User says: "process my brief", "act on my annotations", "process today's annotations", "follow up on my brief"
- User clicks the "Process all annotations" button inside the "Today's Brief" artifact (which calls `sendPrompt("Process the annotations in Today's Brief")`)

## What this skill is NOT for

- **Generating the brief.** That's `/brief`. This command only acts on the brief once it exists and has annotations.
- **Sending or finalizing anything.** Every action is reversible: Gmail drafts (not sends), CRM date updates (not closures), markdown talking points (idempotent appends).
- **Cross-day cleanup.** If you missed yesterday's brief, the annotations live on in `briefs/<yesterday>.md`; this command operates strictly on today's brief.

## Inputs

- Cowork artifact "Today's Brief" — current state of all annotation textareas (via `mcp__cowork__read_widget_context`)
- `<config-root>/briefs/<today>.md` — the markdown snapshot to update with action records
- `<config-root>/voice.md` — for drafting replies in the user's voice
- Gmail MCP, HubSpot MCP — for the side-effects (Gmail drafts, CRM task date updates)

## Outputs

- Gmail drafts (one per `draft_reply` annotation)
- Updated CRM task due dates (one per `reschedule_task` annotation)
- Appended talking points in the markdown snapshot (one per `add_talking_point` annotation)
- Updated section 5 of the artifact ("Drafted replies awaiting approval") summarizing the run
- Append-only line in `<config-root>/plugins/daily-brief.dismissed-log.md` for each `dismiss` annotation

## Routing table

| Annotation pattern | Action | Tool |
|---|---|---|
| "draft reply" / "reply: ..." | `draft_reply` | bizdev-outreach or lead-engine draft skill + Gmail MCP |
| "draft outreach" (group only) | `draft_outreach` | weekly-outreach or lead-engine |
| "move to tomorrow" / "move to <date>" | `reschedule_task` | HubSpot MCP |
| "add talking point: X" / "ask about X" | `add_talking_point` | append to markdown snapshot |
| "skip" / "dismiss" / "I'll handle this" | `dismiss` | log only |
| Free-text / ambiguous | `clarify` | batched follow-up question in chat |

## Two-stage triage

Classifier (Haiku-class): one cheap pass over all annotations to assign normalized actions. Synthesis (Sonnet): only items requiring drafting (`draft_reply`, `draft_outreach`) go through full synthesis. Routing-only items (`reschedule_task`, `dismiss`, simple `add_talking_point`) skip Sonnet entirely.

## Failure modes

- No artifact found → "Run `/brief` first."
- No annotations on the artifact → "Nothing to process. The brief is ready for annotation."
- Cowork artifact tools not available (Claude Code) → stops with a Claude-Code-specific path: edit the markdown snapshot directly, invoke draft / update commands manually
- Reschedule date >14 days out → asks user to confirm inline before writing to CRM
