---
description: Act on the annotations the user wrote into today's brief artifact. Reads the artifact via Cowork's read_widget_context, routes each non-empty annotation to the right downstream skill (draft reply → bizdev-outreach, move task → CRM, etc.), and updates the brief's "Drafted replies awaiting approval" section with what was queued. Never sends anything; drafts only.
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

## Step 1 — Read the artifact's current state (v0.4.0+ interactive)

Locate the Cowork artifact by id `todays-brief` (canonical id as of v0.4.0; previously titled "Today's Brief"). Call `mcp__cowork__read_widget_context` to get the current state — both annotation textareas AND the new interactive state.

In Claude Code (no Cowork artifact tools available): stop with a clear message — "`/process-brief` requires Cowork artifact tools. In Claude Code, edit `<config-root>/briefs/<today_local>.md` directly and call the relevant draft / update commands manually." This is a known v1 limitation.

The artifact's widget context returns:
1. **Annotations dictionary** keyed by item ID (`meeting-<event_id>`, `inbox-<thread_id>`, `task-<task_id>`, `outreach-<contact_id>`). Empty annotations are filtered out.
2. **Tasks checked dictionary** (v0.4.0+) keyed by `task-<task_id>` → boolean. From localStorage `brief-YYYY-MM-DD.tasks_checked`. Indicates which P0 tasks the user has marked complete during the day.
3. **Outreach tier collapsed states** (v0.4.0+) — render-only state, ignored by /process-brief.

Both streams feed Step 2 classification. **Checked-off tasks are NOT annotations** — they're a separate signal. Their handling differs:
- A task checked-off WITHOUT an annotation = "I completed this; no further action needed."
- A task checked-off WITH an annotation = both signals fire; annotation routes per Step 2 classification, checkbox routes per Step 3.6 (new — see below).
- A task UNchecked WITH an annotation = annotation routes; checkbox state means "not done yet."

---

## Step 2 — Classify each annotation

For each `(item_id, annotation_text)` pair, classify the user's intent. Use a small set of normalized actions; map free-text input to one of them.

| Pattern in annotation_text | Normalized action | Routing |
|---|---|---|
| "draft reply" / "reply: ..." / "draft response ..." | `draft_reply` | bizdev-outreach skill (or lead-engine for sales-context threads) |
| "move to tomorrow" / "move to <date>" | `reschedule_task` | HubSpot MCP (CRM task update_date) — only valid for task items |
| "skip" / "ignore" / "I'll handle this" / "dismiss" | `dismiss` | log only; no downstream action |
| "draft outreach" / "send DM" (for outreach-group only) | `draft_outreach` | weekly-outreach or lead-engine draft skill |
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
4. Capture the draft URL / Gmail draft ID for the brief's section 5.

### draft_outreach (outreach group)

Hand off to the relevant outreach plugin's draft skill (weekly-outreach if installed, else lead-engine). Pass the annotation text as the instruction. Outreach drafts are written to Gmail or LinkedIn (per the outreach plugin's logic) as drafts only. Capture the result for section 5.

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

## Step 3.6 — Mark checked-off tasks complete in CRM (v0.4.0+)

For every task in the checked dictionary where `checked == true`:

1. Verify the task still exists in CRM (`hubspot:get_crm_object` for the task ID).
2. Look up the task's current status. If already `COMPLETED` → no-op (idempotent).
3. Otherwise, queue a status update: `status: COMPLETED` + `completion_date: today_local`.
4. Batch updates: collect all check-completion intents from this run, present as a confirmation table:
   ```
   The following P0 tasks were checked off in today's brief:
     · "Send proposal to Vector" (task-12345) → mark COMPLETED?
     · "Reply to Sarah on intro request" (task-67890) → mark COMPLETED?

   [Y]es to all · [N]o to all · [E]dit (toggle per row)
   ```
5. On Y → batch HubSpot update. Report successes/failures.
6. On N → no CRM writes; localStorage state still persists (the user knows they did it; CRM just stays out-of-sync until next sync).
7. On E → per-row toggle, then batch.

If `weekly-outreach` is installed and any checked-off task is associated with an outreach contact, also append to the outreach state: "(completed via brief checkbox 2026-05-28)" so the weekly queue reflects it.

This is the new "interactive brief feeds /end-day" path: `/end-day` Step 4 reads the same checked dictionary and knows what got done without re-asking. The brief becomes a living working surface, not a morning snapshot.

---

## Step 4 — Update the markdown snapshot

For each successful action, append a `### Processed annotations` section at the bottom of `<config-root>/briefs/<today_local>.md` if it doesn't exist, then append one bullet per action:

```markdown
- [<ISO-time>] <item_id> → <action>: <one-line result>
```

This is the audit trail. The same content goes into section 5 of the artifact in Step 5.

---

## Step 5 — Update the artifact's section 5

Call `mcp__cowork__update_artifact` on "Today's Brief" with an updated section 5 card. The card lists every action just taken:

```
Drafted replies and updates from this run:

✓ Drafted reply to "RE: Q3 plan" (Sarah Chen) — view in Gmail
✓ Rescheduled "Send Acme proposal" → 2026-05-13
✓ Added talking point to 10:00 — Tom Reyes: "ask about onboarding pricing"
✓ Dismissed: "Newsletter digest"
```

Each line includes a link where possible (Gmail draft URL, CRM record URL). Preserve any annotations already in other sections of the artifact (re-use Step 3a from `/brief` to read-then-re-render).

---

## Step 6 — Notify the user

Output one summary message to chat:

> "Processed <N> annotations. <M> drafts in Gmail, <K> tasks updated, <L> meeting talking points added, <D> dismissed. Section 5 of the brief now shows details. Review drafts before sending."

Don't dump the action details into chat — they're in the artifact and in the markdown snapshot.

---

## Behavior rules

- Drafts only. Never sends email, never marks a task complete, never deletes anything.
- Idempotent on talking points (same text → no double-add).
- Reschedule >14 days out asks for confirmation inline before writing to CRM.
- Ambiguous annotations are batched into one clarification question, not one-per-item.
- Cost: classifier is Haiku-class. Synthesis (draft writes) is Sonnet, gated to items that need it.
- Telemetry: optionally log one line to core-ops `/log-agent-run` per run (skill: process-brief, action_counts: {draft_reply, reschedule_task, add_talking_point, dismiss}, runtime_ms).
- Privacy: annotation text and draft bodies are processed in-context and stored in Gmail / CRM / markdown snapshot. They're not exfiltrated anywhere else.
