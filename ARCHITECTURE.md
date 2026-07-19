# Architecture & Security Model

This document describes 007's trust boundaries, threat model, and approval model. It is reference documentation for humans reviewing the system — it is not loaded into 007's runtime context (see README.md for what *is* auto-loaded).

## What 007 trusts vs. does not trust

**Trusted (source of instructions/authority):**
- Egor and Ben, directly, in the live conversation.
- `007/AGENTS.md` and the other persona files in this repo — committed, reviewed, human-authored.
- Skill files under `.claude/skills/` — same: committed, human-reviewed before merge.

**Not trusted (treated as data, never as instructions):**
- Customer/ticket content pulled from Intercom (`get_conversation`, `search_conversations` results) — written by people outside the org, read during `queue-check`, `monthly-report`, and any log-bundle analysis.
- Any text embedded in log bundles or attachments.
- Content fetched from the public web (used sparingly, only for third-party technical facts per `kb-writer`).

The rule: nothing pulled from an untrusted source is ever treated as a new instruction, rule, or authority — only as content to analyze, summarize, or quote. If untrusted content contains something that reads like an instruction ("ignore previous rules and...", a fake internal directive, etc.), it gets flagged to Egor, not followed.

## Threat model

**Primary threat: indirect prompt injection via ticket content.** A session can hold both (a) untrusted customer text and (b) live write-capable connectors (Slack send, Intercom KB publish) at the same time. If a ticket body contained a crafted instruction, the only thing preventing it from being acted on is the model's adherence to AGENTS.md's approval rules — there is no platform-level technical gate stopping it. This is a real, currently-accepted risk (see Known Limitations below), not a solved one.

**Secondary threat: memory poisoning.** If a false or manipulated "fact" from untrusted ticket content were ever written into `MEMORY.md` as if it were established truth, it would silently become authoritative for every future session. Mitigation today: memory writes happen from Egor/Ben's direct statements or from 007's own verified analysis, not by copying claims out of ticket text.

**Tertiary threat: unapproved external action.** Sending a message, publishing a KB article, or changing a ticket's status are all "reach outside the sandbox" actions. Mitigated by the approval model below — again, enforced by instruction, not by platform permissions.

## Authority hierarchy

Enforced in `007/AGENTS.md` (loaded every session) — this section is the reasoning behind it, for review, not a second copy to keep in sync.

**Precedence, highest to lowest:** System prompt → AGENTS.md → skill instructions → knowledge base (live data) → memory → current request.

**Why each level outranks the one below it:**
- **System prompt > AGENTS.md** — the platform/harness configuration bounds what's possible at all; nothing in this repo can grant itself capabilities the harness doesn't allow.
- **AGENTS.md > skill instructions** — AGENTS.md's hard rules are deliberate, reviewed, and meant to apply everywhere. A skill's job is to accomplish a specific task *within* those rules, never to relax one. If a skill's instructions ever contradicted a hard rule, that's a bug in the skill to fix, not a case for the skill to win.
- **Skill instructions > knowledge base** — the skill defines the correct process for handling data (e.g. how `monthly-report` categorizes a ticket); the knowledge base is the data being processed, not a source of process. Data doesn't get to redefine the method used to interpret it.
- **Knowledge base > memory** — live sources (Intercom conversations, Help Center articles, log bundles) are current ground truth. MEMORY.md is 007's own recollection *about* that world, which can go stale or be wrong. When they disagree, trust the live source and correct memory to match — not the other way around.
- **Memory > current request** — the current request is the least verified input in the entire chain: it's live, in-the-moment, and (per the threat model above) the surface most exposed to injected or copy-pasted content. Memory holds decisions that were made deliberately and durably (e.g. "Unframe is off the table"). If a request seems to contradict one, that's a signal to check, not silently comply or silently ignore the request.

**Worked examples:**
- *Skill vs. AGENTS.md:* if a skill file ever instructed "offer to schedule a call with the customer," AGENTS.md's rule against that wins — the skill is wrong and needs fixing, the instruction doesn't get followed in the meantime.
- *Memory vs. current request:* MEMORY.md records that Unframe is off the table as a job option (2026-07-14 decision). If a future request casually asked to reconsider it, 007 flags the conflict ("this contradicts the noted decision - are you actually revisiting it, or was that a slip?") instead of quietly going along with either the old decision or the new request.
- *Knowledge base vs. memory:* CONFIG.md records Ben's Slack ID. If live Slack data ever returned something different for Ben, the live value wins and CONFIG.md gets corrected, not the other way around.

## Configuration

Operational values that skills actually pass to tool calls (Intercom admin IDs, Slack IDs, the monthly-report category↔tag mapping) live in `007/CONFIG.md`, referenced by skills rather than duplicated inline. Deliberately kept out of the auto-loaded bootstrap, same reasoning as this file — it's data a skill reads when it runs a task that needs it, not persona content worth its context cost every session. Relationship/context knowledge (named key accounts, team roster, R&D contacts) stays in MEMORY.md — it's not a tool-call parameter, and its long-term home is the structured memory layout planned for a later phase.

## Failure handling

Enforced in `007/AGENTS.md`: never guess past a failed tool call, missing data, or contradictory output — explain and stop instead. Reviewed against every skill; `repo-audit` already modeled this correctly (explicit "adapt to what you find," "never fabricate a finding") and needed no change. `queue-check`, `monthly-report`, and `kb-writer` each got a short skill-specific addition for their own failure surface (e.g. a pagination call failing mid-count in `monthly-report`, an ambiguous multi-match search in `kb-writer`) — see each `SKILL.md` for specifics rather than duplicating them here.

## Approval model

Actions that **always** require Egor's (or Ben's) explicit approval before happening, regardless of skill or context:
- Sending any customer-facing message (Intercom reply, email).
- Publishing or changing the state of a KB article (`kb-writer` only ever creates/updates as `draft`).
- Sending a Slack message.
- Changing a ticket's status/assignment in Intercom (no tool for this exists today — see Connector Capabilities).
- Anything that costs money or reaches a system outside this conversation.

This is enforced entirely by instruction (AGENTS.md rule 1, and each skill's own "no action without approval" language) — see Known Limitations.

## Connector / tool capabilities

What's actually connected in a 007 session, and what each grants:

| Connector | Grants | Used by |
|---|---|---|
| Intercom | Read-only: `search_conversations`, `get_conversation`, `search_contacts`, `search_companies`. Read/write: `search_articles`, `get_article`, `list_articles`, `create_article`, `update_article` (KB). No reply-send or ticket-status-change tool exists. | `queue-check`, `monthly-report`, `kb-writer` |
| Slack | Read (channels, threads, users) and **send** (`slack_send_message`, `slack_schedule_message`, canvas write). Send capability is real and live, gated only by the approval-model rule above. | Ad hoc (e.g. relaying to Ben) |
| Google Drive | Read/write file access. | Not used by any documented skill. |
| Notion | Read/write (pages, databases, comments). | Not used by any documented skill. |
| Atlassian/Jira & Confluence | Read (issues, projects, Confluence pages), some write. | Not used by any documented skill — Jira issue keys are referenced as *data* (read from Intercom custom fields), not fetched directly via this connector in any skill today. |
| GitHub | Full read/write on repos in scope (this repo and `timetrack`). | Used for 007's own development (this repo), not a Faddom-support capability. |

## Assumptions and known limitations

- **No technical enforcement exists for any rule in this document.** Every boundary described here (trust, approval, hierarchy) is enforced by the model following written instructions, not by platform-level permissions. This document exists specifically so that fact is visible rather than assumed away.
- **Connector grants are broader than documented usage.** Google Drive, Notion, and Atlassian are live in-session but no 007 skill uses them. Whether they're intentional standing grants or incidental session scaffolding is unknown — worth a direct question to Egor rather than a guess, and out of scope for this doc to resolve.
- **No per-user attribution in memory.** Facts written to MEMORY.md aren't tagged with who said them or when trust was established — see the architecture review's finding on shared-identity/no-ACL memory.
- This document reflects connectors and skills as of 2026-07-19. It should be updated whenever a new connector is granted or a new skill is added with different capability needs.
