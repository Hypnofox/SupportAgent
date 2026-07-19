# CONFIG — Centralized operational reference

Not auto-loaded every session (same reasoning as ARCHITECTURE.md — this is data a skill reads when it needs it, not persona/memory content worth paying context cost on every session). Skills reference this file instead of embedding these values inline. If a value here changes, it changes in exactly one place.

## Intercom admin IDs
| Person | Admin ID | Notes |
|---|---|---|
| Egor Marachev | `8866794` | |
| Ben Nissley | `9486249` | |
| Fin AI / bot | `7160726` | Unassigned-conversation placeholder — `queue-check` uses this to detect unclaimed tickets |
| Mariano | unknown | Ask Egor before expanding `queue-check` scope to him |
| Yonatan | unknown | Ask Egor before expanding `queue-check` scope to him |

## Slack IDs
| Person | Slack user ID |
|---|---|
| Ben Nissley | `U0998V7JE73` |

## Monthly-report category ↔ Intercom tag mapping
Confirmed with Ben 2026-07-14 — don't re-derive this from count-matching, it was tried once and the counts coincidentally lined up wrong.

| Category shown in report | Intercom tag name |
|---|---|
| Feature Guidance | **Feature Query** (different string — confirmed by Ben directly) |
| Feature Request | Feature Request |
| Discovery & Configuration | Discovery & Configuration |
| Bug | Bug *(also requires a confirmed Jira ticket — see `monthly-report/SKILL.md`'s Bug vs. Escalation rule)* |
| Deployment | Deployment |
| Upgrade | Upgrade |
| License | License |

**Exclusion tags** (a human-handled conversation is excluded from the categorized count only if *all* its tags are in this list): `Spam, Internal Query, test, no_reply, Pending Logs`.

**Not an exclusion tag:** `Premium Support` — it's an account-tier marker, not a category. A ticket with `Premium Support` plus a real category tag is kept.
