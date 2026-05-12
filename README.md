# daily-brief

> Live daily working surface for your business. One Cowork artifact, seven sections, annotated by you and acted on by Claude.

Pulls today's calendar, inbox action items, CRM priority tasks, outreach queue, yesterday's reflection — into a single Cowork artifact titled "Today's Brief." You open it once a day, write short instructions into the textareas under each item ("draft reply: short ack + my view," "move to tomorrow," "ask about pricing"), and then run `/process-brief`. Claude reads your annotations and acts: drafts replies in Gmail, reschedules tasks in your CRM, appends talking points to meeting prep — all as drafts and reversible updates, never sends.

This is Phase 1 of the second-brain v2 extension. The brief is the surface that turns memory into action.

---

## Install

This plugin lives in the [BrightWay AI marketplace](https://github.com/BrightWayAI/nucleus). Install via Cowork or Claude Code's plugin marketplace mechanism, pointed at that marketplace.

Standalone install (less common): `gh repo clone BrightWayAI/daily-brief` into your plugin directory.

## Setup

1. Make sure you have a `<config-root>` set up (any plugin's `/setup-*` command does this on first run). Identity should be captured via cortex's `/setup-identity`.
2. Run `/setup-brief`. Short interview captures section preferences, sort defaults, annotation placeholder hints, and (optional) auto-run schedule.
3. Run `/brief` to generate today's first brief.

## Commands

| Command | Purpose |
|---|---|
| `/brief` | Generate or refresh today's brief artifact and markdown snapshot. Read-only across sources. |
| `/process-brief` | Read your annotations off the artifact and route each one — drafts, reschedules, talking points, dismissals. |
| `/setup-brief` | Configure the plugin. Re-run anytime to update. |

## How it works

```
                    ┌──────────────┐
                    │  /brief      │
                    └──────┬───────┘
                           │ pulls
   ┌──────────────┬────────┼────────┬─────────────┬──────────────┐
   ▼              ▼        ▼        ▼             ▼              ▼
calendar     inbox-triage  CRM   weekly-outreach  cortex      yesterday's
            (or Gmail fallback)                  memory       briefs/<yest>.md
                           │
                           ▼
              ┌──────────────────────────┐
              │  briefs/<today>.md       │  ← markdown snapshot (audit trail)
              └──────────┬───────────────┘
                         │ rendered into
                         ▼
              ┌──────────────────────────┐
              │  Cowork artifact:        │
              │  "Today's Brief"         │  ← live working surface (7 sections,
              │                          │     textarea per item)
              └──────────┬───────────────┘
                         │ user writes annotations,
                         │ clicks "Process all annotations"
                         ▼
              ┌──────────────────────────┐
              │  /process-brief          │
              └──────────┬───────────────┘
                         │ classifies + dispatches
   ┌─────────────┬───────┼──────────────┬──────────────────┐
   ▼             ▼       ▼              ▼                  ▼
Gmail drafts  CRM task  markdown      dismissed-log    follow-up
              date      talking-pt    append           clarify in chat
              update    append
```

### Two-stage triage

Process-brief uses a Haiku-class classifier on annotation text first to assign one of five normalized actions. Only `draft_reply` and `draft_outreach` go through Sonnet-tier synthesis. Routing-only actions (`reschedule_task`, `add_talking_point`, `dismiss`) skip Sonnet entirely. Keeps daily cost modest.

### Drafts only

Every side-effect is reversible: Gmail drafts (not sends), CRM date updates (not closures), markdown talking-point appends (idempotent). You're always in the loop.

## Platform support

- **Cowork**: full artifact lifecycle — `mcp__cowork__create_artifact`, `update_artifact`, `read_widget_context`.
- **Claude Code**: markdown snapshot at `<config-root>/briefs/<today>.md` is fully supported. The interactive artifact surface degrades gracefully; `/brief` prints the snapshot path and `/process-brief` prints a clear message about needing the Cowork artifact tools to read annotations back.

## Configuration

All per-user config lives at `<config-root>/plugins/daily-brief.user-context.md`. The first-run `/setup-brief` interview generates this file. Re-run setup any time to change preferences.

`<config-root>` is the user-chosen folder for marketplace plugin config, recorded at `~/Documents/.claude-plugin-config-root`. See the marketplace README for the convention.

## Roadmap

This is Phase 1 of a larger second-brain extension. Coming next:

- **Phase 2** (`inbox-triage`) — cheap Haiku classifier for inbox section content. Replaces the simple "unread last 24h" fallback with quality-filtered triage.
- **Phase 3** (cortex `person/` pages) — canonical per-person pages so brief section 1 (meetings) can surface 5-second context for any attendee.
- **Phase 5** (`/end-day`) — orchestration chain that ties yesterday's reflection, today's brief, and cortex commit triage into one end-of-day flow.

See `docs/proposals/SECOND-BRAIN-V2-SPEC.md` in the marketplace repo for the full plan.

## License

MIT — see `LICENSE`.
