---
name: monthly-report
description: 007's monthly support report builder - pulls the target month from Intercom directly (no manual CSV export needed), categorizes tickets, and produces the Intercom-derived sections of Egor/Ben's monthly report. Validated against the real June 2026 report.
user-invocable: true
---

You are 007 building the monthly support report. Read `007/SOUL.md`, `007/IDENTITY.md`, `007/USER.md`, `007/AGENTS.md`, `007/MEMORY.md` first if not already loaded this session.

This process was built and validated 2026-07 by running it against June 2026 and comparing to Ben's actual sent report. The core ticket-count methodology is confirmed correct (exact match: 66). Follow it exactly - the gotchas below are all things that silently produced wrong numbers during validation.

## Step 1 - Pull the month's conversations

Use `search_conversations` (Intercom MCP) filtered by `created_at`. The tool only supports one-sided operators (`>=` or `<`), not a range in one call, and **results are NOT reliably sorted by `created_at`** - do not assume pagination order lets you stop early.

Working approach:
1. Query `created_at: {operator: ">=", value: <month_start_epoch>}` with `per_page: 50` to get `total_count` for "month start through now."
2. Query the same with `<month_end_epoch>` (next month's start) to get the count for "before month end" - subtract to sanity-check the target month's count, or just query `>= month_start` and paginate through ALL pages up to that total_count, filtering each page client-side to `created_at < month_end`. Do not stop until you've paged through the full `total_count` - the target month's records are not front-loaded and can be concentrated in later pages.
3. Each page will exceed the tool's inline size limit and get saved to a file - immediately extract only the fields you need via a Python script (see Step 2) and discard the rest. Do not try to read full pages into context.

For carry-over detection (Step 6), you need the prior 1-2 months too, at least for tickets that are still open or whose ticket object exists - see Step 6.

## Step 2 - Extract fields (avoid the HTML-entity bug)

From each conversation, pull: `id`, `created_at`, `state`, `open`, `company.name`, `admin_assignee_id`, `tags.tags[].name`, `ticket.state`, `ticket.ticket_custom_state_admin_label`, `ai_agent_participated`, `ai_agent.resolution_state`, `ai_agent.content_sources.total_count`, `statistics.first_admin_reply_at`, `statistics.first_close_at`, `source.author.email`, `custom_attributes.Tag`, `custom_attributes.jira_issue_key`.

**Critical bug found during validation:** Intercom's API returns tag names HTML-entity-encoded (e.g. `Discovery &amp; Configuration`, not `Discovery & Configuration`). Run `html.unescape()` on every tag name before comparing against category strings, or every "Discovery & Configuration" ticket will silently fall into "uncategorized."

**Do not trust `custom_attributes.Tag` blindly** - it's Fin AI's own auto-classification and can be wrong (confirmed case: a simple "how do I install an update" question was auto-tagged "Deployment" by Fin, but the correct category per content was Feature Guidance since no problem occurred). Use it as a hint, not ground truth.

## Step 3 - Human-handled filter

A conversation counts as human-handled if `statistics.first_admin_reply_at` is not null.

## Step 4 - Tag exclusion

Exclude a human-handled conversation from the categorized count only if **all** its tags are in: `Spam, Internal Query, test, no_reply, Pending Logs`. A ticket with `Premium Support` plus a real category tag is kept - `Premium Support` is an account-tier marker, not a category.

## Step 5 - Categorize

Valid categories and their real Intercom tag names (confirmed with Ben 2026-07-14 - do not re-derive this from count-matching, it was tried and the counts coincidentally lined up wrong the first time):

| Category shown in report | Intercom tag name |
|---|---|
| Feature Guidance | **Feature Query** (yes, different string - confirmed by Ben directly) |
| Feature Request | Feature Request |
| Discovery & Configuration | Discovery & Configuration |
| Bug | Bug *(see Bug/Escalation rule below - not just the tag)* |
| Deployment | Deployment |
| Upgrade | Upgrade |
| License | License |

Definitions (from Egor, use these to manually categorize anything untagged or ambiguous):
- **Feature Query / Feature Guidance** - questions about using Faddom (no bug, no real technical problem).
- **Feature Request** - customer raises a feature request or suggestion.
- **Discovery & Configuration** - how to configure Faddom features, including discovery services (data source discovery, software/user discovery).
- **Bug** - a bug discovered in Faddom.
- **Deployment** - deploying the Faddom server, sensor, or proxy.
- **Upgrade** - problems encountered while upgrading the Faddom server.
- **License** - problems with the customer/prospect's license.

If a conversation has no valid category tag after exclusions, read its actual content (get_conversation) and assign a category yourself using these definitions - don't guess from tags alone.

### Bug vs. Escalation rule (confirmed 2026-07-14)
A ticket only counts as **Bug** if it has a confirmed Jira ticket. Check, in order:
1. `custom_attributes.jira_issue_key` on the conversation itself.
2. The linked back-office/Dev Escalation ticket (`linked_objects`) - check ITS `custom_attributes.jira_issue_key`, or its `ticket.ticket_custom_state_admin_label` containing "Resolved with Jira."
3. An inline Jira-link event in `conversation_parts` referencing `atlassian.net/browse/PROD-####`.

If none of these exist but a back-office/Dev Escalation ticket IS linked (real problem, under investigation, not yet Jira'd) → tag it **Escalation** instead. Escalation is NOT one of the 7 pie-chart categories in the final report - it's tracked separately (matches Ben's report, which lists these in a separate "Open" list rather than the category breakdown). Exclude Escalation-only tickets from the categorized total.

## Step 6 - Carry-over (not yet fully validated - treat as best-effort)
Per the original SOP: include a ticket if created this month, OR created last month and closed this month, OR created 2 months ago and closed this month, OR still open with no close date and created in a prior month. This requires pulling the prior 1-2 months' tickets and checking their `ticket.state`/`first_close_at` - expensive (each month is ~200-260 conversations). Consider scoping this to only conversations that already have a `ticket` object (skip pure Fin-only chats, they don't carry over as tickets).

## Step 7 - Fin AI metrics (not yet fully validated)
Total Cases = human tickets (post Step 4-5) + Fin-deflected cases. Fin-deflected candidates are non-human-handled conversations with `ai_agent.resolution_state` in `assumed_resolution` or `confirmed_resolution`. The exact formula didn't fully reconcile against the reference report's Total Cases (147) during validation - revisit this before trusting it for a real report.

## Step 8 - Country breakdown (not yet attempted)
Use company/contact data; SOP says override any Intercom-assigned country using the customer's own email domain or company, never the Faddom agent's.

## What still needs external input
- **License data** (Step 11 of original SOP) - no connector for the license management system. Ask Egor/Ben for an export.
- **Dev Escalations / Jira issue table** - possibly buildable from linked back-office tickets' Jira data across the month (same mechanism as the Bug/Escalation check above) - not yet attempted at full scale. Worth trying next time before asking for a manual Jira export.

## Output
Show category breakdown with counts, human-handled total, and clearly flag anything approximated vs. validated. Don't chase individual ticket-level category mismatches once the total matches - that's noise per Egor (2026-07-14). Focus on getting the aggregate right and flag real process gaps (like the Watlow-style Escalation case) rather than re-litigating one-off placement.
