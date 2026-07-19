# MEMORY — 007's long-term brain
_Last updated: 2026-07-14_

## About Egor
Technical Services Engineer at Faddom Ltd. (agentless IT infrastructure discovery and dependency mapping, Israel-based). Inherited the support function and has been optimizing it ever since — SLAs, escalation paths, Intercom workflows, ticket classification, a large knowledge base — and still is. Leads Ben, Mariano, and Yonatan without the formal title. Works Sunday-Thursday.

Day to day: an intense live support queue — SSL/certificate issues, VMware/sensor crashes, PostgreSQL diagnostics, proxy auth failures, LDAP/AD problems, network discovery/SNMP issues, proxy registration. Deep log-bundle analysis (Faddom-Server.log, Faddom-Proxy.log, catalina logs). Drafts both customer-facing replies and internal escalation notes to R&D (Ronen Grinberg, Ofer Regev/CTO, Itamar/CPO).

## Their goals
- **Big goal:** Actively job hunting - primarily due to burnout and lack of formal recognition for lead-level work. Financial floor ~22,000 NIS/month. Evaluated Quantum Art and others; Unframe is off the table now, not happening. Live CV site at hypnofox.github.io.
- **Right now / this month:** Keep the support queue running while interviewing; reduce the friction of repetitive log analysis and drafting.

## What I own
Faddom support work specifically: log analysis, drafting customer replies, tracking recurring patterns across tickets (SNMP/firewall, proxy auth tokens, WMI permissions, etc.), keeping tone/format consistent, remembering which accounts need extra care and why.

## Their world
- **Top accounts needing extra care:** BMO, GEICO, Knesset, Clal, VW Argentina, Watlow, Angst+Pfister.
- **R&D contacts for escalation:** Ronen Grinberg, Ofer Regev (CTO), Itamar (CPO).
- **Team (reports to Egor informally):** Ben, Mariano, Yonatan.

## Standards & voice
Hard rules (voice, format, boundaries) are canonical in AGENTS.md — not restated here to avoid drift. This section is for facts/decisions that AREN'T already a hard rule.
- Customer comms stay factual and calm regardless of how messy the underlying issue is.

## Key facts & decisions
- Ticket transcripts (Intercom export format): header block with Ticket ID, participants, ticket type/state/status/assignee, then chronological `HH:MM AM/PM | Speaker: message` entries. Internal-only notes are tagged `(internal)`. Jira issue numbers (e.g. PROD-4519) appear only in internal notes, never in customer-facing replies. Attachments show as `[Attachment: "name.zip"]`.
- Egor's real reply pattern (from VW Argentina ticket #63533559): greeting by first name -> numbered step-by-step instructions -> sign-off ("Best regards, Egor" / "Saludos cordiales, Egor" in Spanish). Internal escalations to R&D read like: "Hi Ronen Grinberg, looks like [customer] isn't able to [X], please advise, see logs attached."
- Log bundle format (zip): `proxyDetails.txt` (proxy count/ID/IP/type/connection status), `serverDetails.txt` (Faddom version, OS, JVM memory, disk), `dbDetails.txt` (active Postgres queries + largest tables), and a `logs/` folder containing `Faddom-Server.log`, `Faddom-Proxy.log` (plus rotated `.gz` per day), `catalina.<date>.log`, tomcat stdout/stderr, commons-daemon logs, `localhost_access_log.<date>.txt`.
- Recurring log signatures worth flagging across tickets:
  - `error 997: Permission Denied` on the Windows proxy service when fetching network sensors / discovery tasks - a Windows service-account permissions issue, not a Faddom bug.
  - HikariPool `ProxyLeakTask` "Connection leak detection triggered" warnings during sensor-data processing - usually benign under load, watch if it recurs tightly.
  - `com.faddom.jb` "Asset not found" burst errors during `TopologyUpdateThread` runs - topology referencing a since-removed asset.
  - Proxy log "has dropped N packets... received -N packets" WARN spam on a sensor connectivity interface - counter-rollover noise, not usually the root cause.
  - Catalina `IllegalStateException: Illegal access... web application instance has been stopped already` during log4j2 config-file probing - benign shutdown artifact, not worth chasing.
- Real fix pattern - proxy not showing as connected after reinstall/clone: copy the current proxy auth key from Faddom UI (Settings > Proxy Configuration), paste into `C:\Program Files\Faddom\Tomcat\conf\faddom-proxy.properties` in the `auth.token` field (check for trailing spaces), restart Tomcat on both Faddom server and proxy, refresh the Proxy Configuration screen to confirm connected.
- VW Argentina context: migrating a Faddom VM via Convert2RHEL (PostgreSQL 12 -> dump/restore to compatible version, Tomcat path changed from `/usr/share/tomcat9` to `/usr/share/tomcat`). Key contacts: Osvaldo Capizzano (IT-Application Architect), Juan Pablo Coustillas, Adrian Abalos, Pablo Colla; external Unix support via T-Systems (Agustin Vergara) and Stefanini service desk.

## Current priorities
- Support job hunting: be ready to help review/tighten CV or interview prep if asked. Unframe is out - don't bring it up as an option.
- Keep absorbing real tickets and log bundles to sharpen pattern recognition — more is better, feed me more when there's time.

## KB writing
- Intercom Help Center access confirmed 2026-07-15: unlike conversations (read-only), articles are read/write - `search_articles`, `get_article`, `list_articles` (216 articles total), `create_article`, `update_article`.
- Egor has a separate "Faddom Tech Writer" Claude Project with a detailed style guide (tone, voice, formatting, UI label rules, word choice, punctuation) for WRITE/EDIT/PROOFREAD tasks on KB articles - that project has no Intercom access and works from pasted text. Built `kb-writer` skill 2026-07-15 to bring the same style guide into 007, but working directly against the live KB. Key carried-over principles: add-don't-rewrite by default on edits, approval-gated one-article-at-a-time, lean over comprehensive (no debug-level detail in public articles), never auto-publish - always draft state for Egor's review.

## Access
- Ben gets the same access/privileges as Egor to this repo and to 007 - full parity, no sanitized/split version. Egor made this call explicitly on 2026-07-14 after I flagged that MEMORY.md/USER.md contain personal info (job hunt, burnout, salary floor). Adding Ben as a GitHub collaborator is Egor's action to take, not something I do.

## Monthly report
- Built the `monthly-report` skill 2026-07-14 after validating against the real June 2026 report end to end. Confirmed rules baked into the skill: category tags need `html.unescape()` (Intercom returns `Discovery &amp; Configuration`); "Feature Query" tag displays as "Feature Guidance" in the report (confirmed by both Egor and Ben - don't re-derive this from count-matching, it looked like a different mapping at first and was wrong); Bug requires a confirmed Jira ticket (check the conversation, its linked back-office ticket, or an inline Jira-link event); a real problem with a Dev Escalation ticket but no Jira yet is "Escalation," not Bug, and Escalation isn't one of the 7 pie-chart categories. Ticket-count methodology is validated exact (66=66) once Escalation is excluded from the categorized total. Carry-over logic and Fin AI deflection metrics are still unvalidated - full details in the skill file.
- License management data and the Dev Escalations/Jira table still need external input or further work - see skill file for specifics.

## Repo structure (2026-07-19)
Ran a scoped architecture cleanup on this repo (no app code exists here, so most of a normal "production-quality" checklist didn't apply). AGENTS.md is now the single canonical source for the hard-rules list - SOUL.md/USER.md/MEMORY.md point to it instead of restating it (they'd already started drifting out of sync with each other). Added a root README.md for repo orientation (folder layout, the two boot paths). Ran a secrets scan - clean, nothing to fix.

Followed up with a 5-phase architecture hardening pass, all documented in `ARCHITECTURE.md`: security/trust model, a formalized and enforced authority hierarchy (AGENTS.md now has the precedence rule - conflicts get named and explained, never silently resolved), per-skill failure handling (never guess, never continue past bad data), and centralized operational config in `007/CONFIG.md` (admin IDs, Slack IDs, monthly-report tag mapping - no longer duplicated across skill files and MEMORY.md). Last step designed the future `memory/` structure (people/decisions/customers/patterns/environment/personal.md split) - documented in ARCHITECTURE.md, not implemented yet; trigger is MEMORY.md crossing ~150-200 lines.

## Slack access
- Slack connected 2026-07-14. I can send messages directly (e.g. to Ben - see `007/CONFIG.md` for his Slack ID) rather than only drafting for Egor to forward. Still applies the same judgment as customer drafts - only send after Egor's reviewed/approved the content, since this is external-facing (visible to a teammate) communication.

## Intercom access
- Intercom MCP tools are connected in Egor's Claude Code sessions: `search_conversations`, `get_conversation`, `search_contacts`, `search_articles`, `list_companies`, etc. Read-only - no reply/status-change tool exists, which matches the "no action without OK" boundary.
- Each conversation carries `ticket.state` (raw enum: submitted/in_progress/waiting_on_customer/etc) and `ticket.ticket_custom_state_admin_label` (the human label shown in Egor's Intercom sidebar views, e.g. "Submitted," "Waiting on dev," "Waiting on customer"). The admin label is what to bucket on to match his Views.
- Admin IDs (Egor, Ben, unassigned/bot placeholder, Mariano/Yonatan status) are centralized in `007/CONFIG.md` - not restated here to avoid drift.
- Built the `queue-check` skill (`.claude/skills/queue-check/SKILL.md`) 2026-07-14: sweeps Intercom in priority order (Submitted -> In Progress -> All open, oldest to newest). Tuned same day after a full 17-ticket test run: Egor already triages his own queue most mornings, so the default behavior is now to flag only the exceptions (tickets with no customer update in days, or a contradiction like the customer signaling resolution while the ticket's still open) with a full summary + draft, and just one-line everything that's already been handled today or delegated internally. Locked in as default behavior 2026-07-14.
