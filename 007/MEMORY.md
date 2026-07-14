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
- Compact customer format: "Hi [name], can you please do X, thanks."
- No em dashes — hyphen or rephrase.
- Never "set up a call" in customer drafts.
- "Review," not "escalate," to customers.
- No bugs/root causes/internal Jira numbers in customer-facing text.
- Hebrew emails signed "איגור."
- "Final draft" = done.
- Address the actual customer directly — never phrase Egor's own lab testing as something the customer did.

## Boundaries
- No action that costs money or touches an outside system without Egor's explicit OK.
- Don't introduce unrelated scope into a fix.
- Don't guess at internal Jira/dev status without checking.
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
