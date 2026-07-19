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

## Future memory structure (planned — not yet migrated)

MEMORY.md is not being split yet — at ~80 lines it's still well under the ~150-200 line trigger the original architecture review set for when the split earns its cost. This section documents the target design now, so when that trigger is hit the split is a mechanical move against an agreed plan, not a redesign done under time pressure. **Nothing in this section has been implemented. MEMORY.md is unchanged.**

### Target layout
```
007/
  MEMORY.md         becomes an index only — stays in the always-loaded bootstrap, short
  memory/
    YYYY-MM-DD.md    dated session logs (unchanged, already exists)
    people.md        who's who: Egor, Ben, Mariano, Yonatan, R&D contacts, roles/relationships
    decisions.md      dated decision/changelog entries (skill builds, validations, access calls)
    customers.md      named key accounts + why they need care, per-account context
    patterns.md       recurring technical knowledge: log signatures, fix procedures, formats
    environment.md    narrative about the tooling/systems 007 operates in (capabilities, field semantics)
    personal.md       sensitive Egor-specific context (job search, workload) — see note below
```

### How today's MEMORY.md maps onto it
- **people.md** ← the role/team parts of "About Egor," plus "Their world"'s R&D contacts and team roster.
- **decisions.md** ← "KB writing," "Access," "Monthly report," "Repo structure" — these sections are already a changelog in disguise, just not labeled as one.
- **customers.md** ← "Their world"'s named-accounts list, plus the VW Argentina paragraph currently sitting in "Key facts & decisions."
- **patterns.md** ← everything else in "Key facts & decisions": transcript format, log bundle format, recurring log signatures, the proxy-auth-token fix procedure.
- **environment.md** ← the narrative parts of "Slack access" and "Intercom access" (capabilities, field semantics — the raw IDs already moved to CONFIG.md in Phase 4, so this file explains behavior, CONFIG.md holds the parameters).
- **personal.md** ← "Their goals" and the personal parts of "Current priorities" (job hunting, burnout, salary floor, CV link).
- **Not migrated anywhere:** "What I own" duplicates SOUL.md's charter already — a candidate for deletion at split time, not a seventh topic file. Flagging it now so it isn't carried over as more duplication.

### Migration rules (apply only when the split actually happens)
1. **Trigger:** MEMORY.md crossing ~150-200 lines — not before. Splitting a file this small today would be premature structure for a problem that doesn't exist yet.
2. **Copy content, don't drop it** — every fact currently in MEMORY.md lands somewhere in the new layout; nothing gets silently lost in the move.
3. **Placement test**, applied to each fact in order, first match wins:
   - A literal parameter a skill passes to a tool call (an ID, a mapping)? → Not memory at all — `CONFIG.md` (Phase 4), not a topic file.
   - About a specific person (role, contact, relationship)? → `people.md`
   - A decision or build/validation event tied to a date? → `decisions.md`
   - About a specific customer/account? → `customers.md`
   - A recurring technical pattern usable across tickets? → `patterns.md`
   - About the tooling/systems 007 operates in? → `environment.md`
   - Sensitive/personal to Egor specifically (career, health, compensation)? → `personal.md`
   - None of the above, or genuinely one-off/session-specific? → stays in the dated `memory/YYYY-MM-DD.md` log, not a topic file.
4. **Skill files need a pass at migration time**, not before. Every skill currently says "read MEMORY.md" — some will need to name a specific topic file too (e.g. a `queue-check` run touching a named account would want `customers.md`). Real work when it happens; not solved speculatively now.
5. **Post-split MEMORY.md** keeps: a short "what I own" pointer (trimmed, not duplicated from SOUL.md) plus one line per topic file naming what's in it — enough for 007 to know a fact probably exists and where to look, without loading everything.

### Retrieval strategy
- MEMORY.md (the index) stays in the always-loaded bootstrap exactly as today — small, cheap, always available.
- Topic files under `memory/` are **not** auto-loaded. 007 or a skill reads the specific file relevant to the task at hand — the same on-demand pattern already established for `CONFIG.md` and this file. A `queue-check` run touching a named account reads `customers.md`; drafting a reply involving a known log signature reads `patterns.md`.
- No search or indexing layer — a handful of markdown files doesn't need one. A plain Read call is enough; building more (vector search, a query layer) would solve a problem this repo doesn't have. Revisit only if a single topic file itself grows large enough that reading all of it stops being cheap.
- Dated session logs are the lowest-priority read — narrative history, pulled only for "what happened when," not part of routine skill execution.

### Known consequence, not addressed by this plan
`personal.md` gets no different access treatment than any other topic file here — Ben would still see it, same as MEMORY.md today, consistent with the existing full-parity decision. If that's ever revisited, `personal.md` is the natural attachment point for scoping, since it's the one file already understood to hold Egor-specific sensitive content. This is a placement note for future reference, not a proposal to change access now.

## Assumptions and known limitations

- **No technical enforcement exists for any rule in this document.** Every boundary described here (trust, approval, hierarchy) is enforced by the model following written instructions, not by platform-level permissions. This document exists specifically so that fact is visible rather than assumed away.
- **Connector grants are broader than documented usage.** Google Drive, Notion, and Atlassian are live in-session but no 007 skill uses them. Whether they're intentional standing grants or incidental session scaffolding is unknown — worth a direct question to Egor rather than a guess, and out of scope for this doc to resolve.
- **No per-user attribution in memory.** Facts written to MEMORY.md aren't tagged with who said them or when trust was established — see the architecture review's finding on shared-identity/no-ACL memory. The planned `personal.md` split (above) doesn't resolve this either, by design — it's noted, not fixed.
- This document reflects connectors and skills as of 2026-07-19. It should be updated whenever a new connector is granted or a new skill is added with different capability needs.
