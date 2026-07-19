# SupportAgent

This repo *is* 007 — Egor's (and Ben's) AI second brain for Faddom support. There's no application code here; the "system" is a persona + memory file set that Claude Code loads, plus a set of reusable skills built on top of it.

## Layout

```
007/                  Persona and long-term memory (loaded every session)
  SOUL.md             Who 007 is, mission, what it cares about
  IDENTITY.md         Short role card (name, vibe, reports to)
  USER.md             About Egor — role, comms preferences
  AGENTS.md           Canonical hard rules (voice, format, boundaries) — single source of truth
  MEMORY.md           Long-term facts: accounts, contacts, recurring bug signatures, decisions
  CLAUDE.md           Auto-loaded project instructions (bootstraps the files above)
  memory/YYYY-MM-DD.md  Dated session logs — what happened, appended at end of a working session

.claude/skills/        Reusable slash-command skills, each a SKILL.md
  007/                 The base "become 007" bootstrap (triggered by /007)
  queue-check/         Intercom queue sweep — flags only tickets that fell through the cracks
  monthly-report/      Builds the monthly support report straight from Intercom
  kb-writer/           Writes/edits/proofreads Faddom Help Center articles against the live KB
  repo-audit/          Read-only security audit of an untrusted repo before running anything in it
```

## How it boots

Two independent trigger paths load the same persona files, by design:
- **`007/CLAUDE.md`** auto-loads as project instructions any time Claude Code operates in this repo, so identity/memory come along even without an explicit command.
- **`.claude/skills/007/SKILL.md`** is the explicit `/007` slash command — same bootstrap, invoked on demand (and referenced by the other skills so they can run standalone).

## Hard rules

AGENTS.md is canonical. Every other file (SOUL, USER, MEMORY) points to it rather than restating the list, to avoid drift.

## Memory protocol

Durable facts (goals, preferences, decisions, new accounts, new failure patterns) get written to `MEMORY.md` immediately, no permission needed. End-of-session summaries go in `007/memory/YYYY-MM-DD.md`.

## Access

Egor and Ben have full parity access — same repo, same memory, no sanitized version (Egor's explicit call, logged in MEMORY.md). Note that `MEMORY.md` contains some personal context about Egor (job search, workload) by design.
