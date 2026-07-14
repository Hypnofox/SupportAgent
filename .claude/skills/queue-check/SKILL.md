---
name: queue-check
description: 007's Intercom queue sweep - pulls every open ticket assigned to Egor/Ben and flags only the ones that actually fell through the cracks, since Egor already works the rest of the queue himself every morning.
user-invocable: true
---

You are 007 running a queue sweep. First read `007/SOUL.md`, `007/IDENTITY.md`, `007/USER.md`, `007/AGENTS.md`, `007/MEMORY.md` if you haven't already loaded them this session, so drafts come out in Egor's voice and follow his hard rules (no em dashes, "review" not "escalate", no bugs/root causes/Jira to customers, compact "Hi [name], can you please do X, thanks" format, etc).

## Scope
Tickets assigned to Egor (admin id `8866794`) or Ben (admin id `9486249`), `open: true`. Use the Intercom MCP tools (`search_conversations`, `get_conversation`) - query per admin_assignee_id and merge (in practice these two lists don't overlap, so no dedup needed, but check anyway).

If Mariano's or Yonatan's admin IDs become known later, fold them into scope too - ask Egor for the IDs if this ever needs expanding.

## Priority order (stop at the first non-empty bucket)
Each conversation's `ticket.ticket_custom_state_admin_label` is the human label that matches Egor's Intercom sidebar views exactly. Bucket the pulled tickets by that label:

1. **Submitted** - if any tickets have this label, these haven't been worked at all yet. Pull full detail, summarize each, show them to Egor. No draft needed - just surface them.
2. **In Progress** (the literal admin label "In Progress," not "Waiting on dev" or other sub-states) - only if Submitted was empty.
3. **All** - only if both Submitted and In Progress were empty. Every open ticket in scope.

In practice (confirmed 2026-07-14), Submitted and In Progress are usually empty because Egor triages his own queue every morning - so this usually lands on "All."

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

## Hard rules (inherited from AGENTS.md - do not skip)
- No action that touches Intercom (sending replies, changing ticket status) without explicit approval - this skill only reads and drafts.
- No em dashes in drafts.
- Never "set up a call" in a customer-facing draft.
- Say "review," never "escalate," to a customer.
- Never mention bugs, root causes, or internal Jira issue keys in a customer-facing draft.
- Address the actual customer in instructions - never phrase Egor's own lab testing as something the customer did.
