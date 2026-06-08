---
description: Act on the actions + annotations the user wrote into today's brief artifact. Reads the artifact via Cowork's read_widget_context (canonical v0.5.0 JSON-blob shape — tasks/annotations/outreach_actions), routes each annotation to the right downstream skill (draft reply → relationships /draft-touchpoint, move task → CRM), and stages task actions (done→COMPLETED, delegate→delegatee task, skip→defer) and outreach actions (sent/nudge/booked/let_go) to CRM and person/bizdev nodes. Intra-day actor; durable memory write-backs + suppression learning happen in /end-day Step 2c. Never sends anything; drafts only.
---

# /process-brief

Closes the loop on the brief. The user has already opened today's "Today's Brief" artifact and written instructions into one or more textarea annotations. This command reads those instructions back and acts on them.

**This is a drafting / staging command.** It writes Gmail drafts, updates CRM task due dates, appends meeting talking points. It does NOT send emails, mark tasks complete, or do anything irreversible without explicit user approval.

---

## Step 0 — Resolve config root and load context

### A — Resolve `<config-root>`

Ensure access to `~/Documents`. In Cowork, call `request_cowork_directory(~/Documents)` once if not already granted. In Claude Code (or any environment with direct filesystem access), no mount is needed. Then read `~/Documents/.claude-plugin-config-root`.

- **Pointer exists**: read line 1 → that's `<config-root>`. Ensure access to `<config-root>`. If running in Cowork and the folder isn't already mounted in this session, call `request_cowork_directory(<config-root>)`. If running in Claude Code or another environment with direct filesystem access, no mount call is needed. Continue.
- **Pointer missing**: stop with "No plugin config root found. Run `/setup-brief` (or any plugin's setup) first."

Resolve `<today_local>` the same way `/brief` does (user time zone from `<config-root>/identity.md`).

### B — Verify today's brief exists

Read `<config-root>/briefs/<today_local>.md`.

- **Exists** → continue.
- **Missing** → stop with: "No brief found for <today_local>. Run `/brief` first to generate one."

---

## Step 1 — Read the artifact's current state (v0.5.0 — canonical localStorage shape)

Locate the Cowork artifact by id `todays-brief`. Call `mcp__cowork__read_widget_context(artifact_id="todays-brief")` to get the current state.

In Claude Code (no Cowork artifact tools available): stop with a clear message — "`/process-brief` requires Cowork artifact tools. In Claude Code, edit `<config-root>/briefs/<today_local>.md` directly and call the relevant draft / update commands manually." This is a known v1 limitation.

### Reading the JSON-blob localStorage state (canonical v0.5.0 shape)

The widget context returns the single localStorage entry at key `brief-<today_local>`. Parse it (per `daily-brief/commands/brief.md` localStorage contract):

```javascript
const state = JSON.parse(widget_context["brief-<today_local>"] || "{}");

const tasks       = state.tasks || {};            // {task_id: {action: done|delegate|skip|not_important, detail, ts, name}}
const annotations = state.annotations || {};      // {item_id: free-form-text}
const outreach    = state.outreach_actions || {}; // {contact_id: {name, action: sent|skip|nudge|let_go|booked|dead, bucket, signal, value_add, detail, ts}}
const schemaVersion = state.schema_version || "0.4.0";
```

**Back-compat:** if you see `state.tasks_checked` (a `{id: bool}` map from v0.4.x) but no `state.tasks`, treat each `true` as `{action: "done"}`. If `schema_version` is newer than `0.5.0`, surface a one-line warning and proceed with the fields you recognize.

### Three streams feed Step 2/3

- **`annotations`** — free-form user input per item. Routed by Step 2 patterns (draft_reply / reschedule_task / dismiss / draft_outreach / clarify).
- **`tasks`** — per-task action objects, NOT annotations. Routed per Step 3.6 (done → CRM COMPLETED; delegate → CRM task for delegatee; skip → defer due date; not_important → log for `/end-day` suppression).
- **`outreach_actions`** — per-contact action objects with categories. Routed per Step 3.7 (sent/nudge → draft touchpoint; booked → prep task; let_go/dead → log).

Behavior matrix (task action + annotation are independent signals on the same row):
- Task `done` without annotation → Step 3.6 only.
- Task `done` with annotation → both fire: annotation routes per Step 2, action routes per Step 3.6.
- No task action but annotation present → annotation routes; the task isn't acted on.

**Division of labor with `/end-day`:** `/process-brief` is the intra-day actor — it stages reversible side-effects on demand (Gmail drafts, CRM date/status updates, touchpoint drafts) so you can process mid-day. The **memory write-backs and suppression learning** (writing `not_important` items into `surfacing-prefs.md`, logging touches onto person/bizdev nodes, incrementing skip counters) happen in `/end-day` Step 2c, which reads the same blob. `/process-brief` is idempotent and safe to run repeatedly; `/end-day` mines the final end-of-day state.

---

## Step 2 — Classify each annotation

For each `(item_id, annotation_text)` pair, classify the user's intent. Use a small set of normalized actions; map free-text input to one of them.

| Pattern in annotation_text | Normalized action | Routing |
|---|---|---|
| "draft reply" / "reply: ..." / "draft response ..." | `draft_reply` | relationships `/draft-touchpoint` (or lead-engine for sales-context cold-prospect threads) |
| "move to tomorrow" / "move to <date>" | `reschedule_task` | HubSpot MCP (CRM task update_date) — only valid for task items |
| "skip" / "ignore" / "I'll handle this" / "dismiss" | `dismiss` | log only; no downstream action |
| "draft outreach" / "send DM" (for outreach-group only) | `draft_outreach` | relationships `/draft-touchpoint` (preferred) or lead-engine; legacy weekly-outreach fallback only when relationships not installed |
| Anything else / ambiguous | `clarify` | ask the user a one-line follow-up in chat |

**Note:** `add_talking_point` (formerly for meeting annotations) was removed in daily-brief v0.3.0 — meetings are now read-only context cards in `/brief`. Use cortex `/recall <person>` for prep context instead, or edit the markdown snapshot directly.

Two-stage triage: classify cheaply first (Haiku-class, just the text + a one-word menu), then dispatch only the items that need synthesis (draft_reply, draft_outreach) to Sonnet-tier work. Items that are pure routing (reschedule_task, dismiss) don't need synthesis at all.

For `clarify` items, batch them: ask the user one combined question listing each ambiguous annotation and offering "[draft / move / talking point / skip / other]" per item. Don't ask one-at-a-time.

---

## Step 3 — Execute the actions

### draft_reply (inbox item)

1. Read the original thread via Gmail MCP using the `thread_id` from the item ID.
2. Call the writing-style plugin's draft skill if installed (or inline draft using `<config-root>/voice.md`). Build a reply that captures the user's intent from `annotation_text`.
3. Save as a Gmail draft via `mcp__f77d3a90-04d4-4394-96b5-a5d4402dfe0a__create_draft` (or whichever Gmail tool the runtime exposes).
4. Capture the draft URL / Gmail draft ID for the run summary + markdown twin.

### draft_outreach (outreach group)

Hand off to the relevant outreach plugin's draft skill — preferred: relationships `/draft-touchpoint` (v0.2.0+); fallback: lead-engine or legacy weekly-outreach if `relationships` isn't installed. Pass the annotation text as the instruction. Outreach drafts are written to Gmail or LinkedIn (per the outreach plugin's logic) as drafts only. Capture the result for the run summary + markdown twin.

### reschedule_task (task item)

1. Parse the date from `annotation_text` ("tomorrow", "next Monday", explicit ISO date).
2. Update the CRM task's due date via HubSpot MCP `manage_crm_objects` (or `update_records_for_table` for non-HubSpot CRMs).
3. Confirm with the user inline only if the parsed date is more than 14 days in the future (sanity check).

### dismiss

Log to `<config-root>/plugins/daily-brief.dismissed-log.md` (append-only) one line:

```
<today_local> <ISO-time> <item_id> dismissed: <first 80 chars of annotation>
```

Do nothing else.

### clarify

Already handled in Step 2 — by the time you reach Step 3, the user has answered. Re-classify based on the answer and re-route to the appropriate action.

---

## Step 3.6 — Route task actions to CRM (v0.5.0)

Walk `state.tasks`. Route by `action`:

**`done`** — verify the task still exists in CRM (read it back via the HubSpot MCP); if already `COMPLETED`, no-op (idempotent); else queue `status: COMPLETED` + `completion_date: today_local` via `manage_crm_objects` (or `update_records_for_table` for non-HubSpot CRMs).

**`delegate`** (`detail` = delegatee, default Erica) — queue: reassign the source task's owner to the delegatee if the CRM supports it, OR create a new CRM task for the delegatee mirroring the original, due `today_local`. Note the delegation in the run summary.

**`skip`** (`detail` = snooze duration, e.g. `3d`) — queue a due-date push on the source task by the parsed duration (same path as `reschedule_task`). This is the intra-day version; `/end-day` separately increments the per-task skip counter for the repeat-ignore rule.

**`not_important`** — do NOT write to CRM here. Log it for `/end-day` to fold into `surfacing-prefs.md`: append a line to `<config-root>/plugins/daily-brief.dismissed-log.md` as `<today> <ISO-time> task-<id> not_important: <title>`. (The authoritative suppression write happens in `/end-day` Step 2c.)

Batch all CRM intents and present one confirmation table before writing:

```
From today's brief task actions:
  · "Deliver B&S plan" (task-123) → mark COMPLETED
  · "CBM admin" (task-456) → delegate to Erica (create her task)
  · "Globant reply" (task-789) → skip 3d (push due date)

[Y]es to all · [N]o to all · [E]dit (toggle per row)
```

On Y → batch CRM writes; report successes/failures. On N → no CRM writes (localStorage state persists; `/end-day` will still mine it). On E → per-row toggle, then batch.

If `relationships` is installed and a `done`/`delegate` task is associated with a person in `memory/person/`, invoke `/touchpoint <person> --channel=task --summary="<action> via brief"` to log it. (Legacy fallback: `weekly-outreach` outreach state.)

## Step 3.7 — Route outreach actions (v0.5.0)

Walk `state.outreach_actions`. Each entry carries `action` + optional `bucket` / `value_add` / `signal`.

- **`sent`** — log a touch on the contact's person/bizdev node (or hand to relationships `/draft-touchpoint` if a draft is still needed). Record `bucket` + `value_add` + `signal` on the touch so outreach analytics can roll them up.
- **`nudge`** — log a follow-up touch (with `detail` snooze if present).
- **`booked`** — advance the pipeline stage and create a CRM prep task.
- **`let_go` / `dead`** — log and remove the contact from the active queue.
- **`skip`** — defer reappearance by `detail` duration; no node write.

Batch these into the same confirmation flow as Step 3.6 (or a second table) so nothing writes without a single approval. The authoritative node write-backs are owned by `/end-day` Step 2c; in `/process-brief` these are the intra-day stages — keep them idempotent (don't double-log the same touch).

This is the "interactive brief feeds /end-day" path: `/end-day` Step 2c reads the same `tasks` + `outreach_actions` dictionaries and does the durable memory write-backs + suppression learning without re-asking. The brief is a living working surface, not a morning snapshot.

---

## Step 4 — Update the markdown snapshot

For each successful action, append a `### Processed annotations` section at the bottom of `<config-root>/briefs/<today_local>.md` if it doesn't exist, then append one bullet per action:

```markdown
- [<ISO-time>] <item_id> → <action>: <one-line result>
```

This is the audit trail. It's appended under `### Processed annotations` at the bottom of the markdown twin.

> Note (v0.5.0): the 5-section brief has no "Drafted replies awaiting approval" card. The action results live in the markdown twin (above) and in the artifact's existing task/outreach action logs (which already reflect the user's clicks). `/process-brief` does NOT add a new artifact section; it reports results in chat (Step 5).

---

## Step 5 — Notify the user

Output one summary message to chat:

> "Processed <N> annotations + <A> task actions + <O> outreach actions. <M> Gmail drafts, <K> CRM updates (completions/delegations/reschedules), <T> touchpoints logged, <D> dismissed. Review drafts before sending. End-of-day write-backs + suppression learning will run in `/end-day`."

Don't dump the action details into chat — they're in the markdown twin and the artifact's action logs.

---

## Behavior rules

- Drafts only. Never sends email, never marks a task complete, never deletes anything.
- Idempotent on talking points (same text → no double-add).
- Reschedule >14 days out asks for confirmation inline before writing to CRM.
- Ambiguous annotations are batched into one clarification question, not one-per-item.
- Cost: classifier is Haiku-class. Synthesis (draft writes) is Sonnet, gated to items that need it.
- Telemetry: optionally log one line to core-ops `/log-agent-run` per run (skill: process-brief, action_counts: {draft_reply, reschedule_task, dismiss, task_done, task_delegate, task_skip, task_not_important, outreach_sent, outreach_nudge, outreach_booked, outreach_let_go}, runtime_ms).
- Privacy: annotation text and draft bodies are processed in-context and stored in Gmail / CRM / markdown snapshot. They're not exfiltrated anywhere else.
