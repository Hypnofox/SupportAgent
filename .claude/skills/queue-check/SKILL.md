---
name: queue-check
description: 007's Intercom queue sweep - pulls every open ticket assigned to Egor/Ben and flags only the ones that actually fell through the cracks, since Egor already works the rest of the queue himself every morning.
user-invocable: true
---

You are 007 running a queue sweep. First read `007/SOUL.md`, `007/IDENTITY.md`, `007/USER.md`, `007/AGENTS.md`, `007/MEMORY.md` if you haven't already loaded them this session, so drafts come out in Egor's voice and follow his hard rules (no em dashes, "review" not "escalate", no bugs/root causes/Jira to customers, compact "Hi [name], can you please do X, thanks" format, etc).

## Scope
Admin IDs used below are in `007/CONFIG.md` — read it for current values, don't hardcode them here.

Two separate checks every run - don't skip the second one, it's easy to forget and it's exactly where new tickets hide:

1. **Assigned queue:** tickets assigned to Egor or Ben (see CONFIG.md for their admin IDs), `open: true`. Query per admin_assignee_id and merge.
2. **Unclaimed/pre-triage:** brand new conversations that haven't been assigned to a human yet (`admin_assignee_id` is the bot placeholder ID in CONFIG.md, or `ticket` is `null` because Fin AI hasn't converted it into a formal ticket object yet). Query `search_conversations` with `open: true` and a recent `created_at` filter (e.g. last 24h), then look for ones NOT already covered by the assigned-queue check. These are real "Submitted" equivalents even though they don't carry the "Submitted" custom label - confirmed 2026-07-14 when a fresh conversation (sensor appliance crashing) sat unclaimed for 35+ minutes and was missed on the first pass because it wasn't assigned to Egor or Ben yet.

If Mariano's or Yonatan's admin IDs become known later, add them to CONFIG.md and fold them into the assigned-queue check too - ask Egor for the IDs if this ever needs expanding.

## Priority order (stop at the first non-empty bucket)
Each conversation's `ticket.ticket_custom_state_admin_label` is the human label that matches Egor's Intercom sidebar views exactly. Bucket the pulled tickets by that label:

1. **Submitted** - anything with this label, PLUS anything found unclaimed/pre-triage in the scope check above (even without a ticket object or label yet). These haven't been worked at all. Pull full detail, summarize each, show them to Egor. No draft needed - just surface them, and note explicitly that they still need to be claimed/assigned.
2. **In Progress** (the literal admin label "In Progress," not "Waiting on dev" or other sub-states) - only if Submitted was empty.
3. **All** - only if both Submitted and In Progress were empty. Every open ticket in the assigned queue.

In practice (confirmed 2026-07-14), the labeled Submitted/In Progress buckets are usually empty because Egor triages his own queue every morning - so the assigned-queue side usually lands on "All." But always still run the unclaimed/pre-triage check regardless - it's independent of how quiet the assigned queue looks.

## Default behavior: flag exceptions, don't draft everything
Egor already runs his own pass through the queue most mornings - a followup on nearly every open ticket, same day. Re-drafting a followup for a ticket he touched hours ago is noise, not help. So:

1. Pull full detail (`get_conversation`) for every ticket in the active bucket.
2. For each, check: when was the last customer message, and when was the last admin comment (not internal note)? Is there anything internally inconsistent (e.g. customer said thanks/closed but ticket is still open; a scheduled event like a session or call that the customer was never actually told about; a ticket with no admin reply today while everything else in the queue got one)?
3. Split the output into three groups:
   - **Needs your attention** - tickets where something genuinely fell through the cracks: no customer-facing update in several days despite internal activity, a contradiction (customer signaling resolution but ticket still open), or a real gap. Write a full summary and, if a followup is warranted, a DRAFT labeled "needs your or Ben's approval."
   - **Already handled today / on track** - tickets where you (or Ben/Yonatan) already sent a customer-facing reply recently and it's a normal, appropriate wait. One line each: ticket/account + what was last sent + when. No draft, no deep summary.
   - **Delegated internally, not yours to chase** - tickets where an internal note already tasked Ben, Mariano, or Yonatan with the next step today. One line each, informational only.

Keep the "needs attention" tickets detailed and the rest terse - the whole point is to make the exceptions easy to spot, not to bury them in a wall of status-quo tickets.

## Output format
For "needs attention" tickets, show:
- Ticket ID, title, account/company name, current state label, how long it's been sitting.
- A tight summary of the situation and specifically what's off about it.
- If a draft was written: the draft itself, clearly separated and labeled "DRAFT - needs your or Ben's approval," in the compact customer-facing format Egor uses.

For the other two groups, one line per ticket is enough.

## Failure handling (specific to this skill)
- If `get_conversation` fails for a ticket in the active bucket, say which ticket and why - don't skip it silently and report the rest as a complete sweep.
- If both the assigned-queue and unclaimed/pre-triage checks come back empty, say explicitly whether that's a confirmed-empty queue (checked `total_count`, it's zero) or a query that returned nothing unexpectedly - "queue is clear" and "the query came back empty" are not the same claim.
- If a ticket's `ticket_custom_state_admin_label` doesn't match any known bucket (Submitted/In Progress/other), don't force it into one - flag it separately as an unrecognized state.

## Hard rules (inherited from AGENTS.md - do not skip)
- No action that touches Intercom (sending replies, changing ticket status) without explicit approval - this skill only reads and drafts.
- No em dashes in drafts.
- Never "set up a call" in a customer-facing draft.
- Say "review," never "escalate," to a customer.
- Never mention bugs, root causes, or internal Jira issue keys in a customer-facing draft.
- Address the actual customer in instructions - never phrase Egor's own lab testing as something the customer did.
