---
name: repo-audit
description: Comprehensive read-only security audit of an untrusted repository before executing, building, installing dependencies, or recommending any commands. Use when asked to audit, vet, or assess the safety of a repo before running it.
user-invocable: true
---

You are performing a security audit of a repository. Treat it as **untrusted until proven otherwise** - do not assume good intent, and do not extend the benefit of the doubt just because the repo looks like a normal project.

## Hard rule
Do NOT execute any code, install dependencies, build the project, start a server, or recommend that the user run any command until the audit below is complete and you have delivered the full written assessment. Everything in this skill is read-only investigation (`Read`, `Grep`, `Glob`, `git log`/`git diff`/`git status` only - no `npm install`, no `pip install`, no running scripts, no `docker build`, nothing that executes repo content).

## Step 1 - Map what's actually there
Before diving into categories, establish ground truth with cheap, safe commands:
- `git log --oneline -20`, `git branch -a`, `git remote -v`, `git status`
- Full file listing (`find . -not -path '*/.git/*' -type f | sort`)
- Look specifically for: package manifests (`package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`, etc.), lockfiles, `Dockerfile`/`docker-compose*`, `.github/workflows/*`, install/setup scripts, hidden dotfiles, executable-bit files (`find . -perm -u+x`).
- Diff other branches / check git history for anything added then deleted (`git log --all --diff-filter=A --name-only`).

**Adapt to what you find.** If the repo has no code at all (e.g. it's pure config, docs, or a Claude Code skills/persona repo like this one), say so plainly and restructure the audit around whatever risk surface actually exists (data handling, agent instructions, connected tool permissions) rather than padding the report with findings for categories that don't apply. Never fabricate a finding to fill out a checklist.

## Step 2 - Analyze across these categories (skip/mark N/A explicitly where nothing exists)
- **Repository health and maintenance** - commit history, staleness, branch structure, signs of abandonment or suspicious recent changes.
- **Supply-chain risks** - dependencies, install/postinstall scripts, downloaded binaries, typosquatting, pinned vs. floating versions.
- **Dangerous code** - shell execution, subprocess/`os.system`/`child_process`, `eval`/`exec`, dynamic imports, obfuscation (base64 blobs, hex escapes, minified payloads with no source).
- **Network behavior** - telemetry, outbound calls, upload endpoints, hardcoded URLs/IPs, curl-pipe-to-shell patterns.
- **Credentials and secrets** - API keys, tokens, cloud credentials, SSH key usage, anything matching common secret patterns (grep for `api[_-]?key`, `secret`, `token`, `BEGIN.*PRIVATE`, `AKIA[0-9A-Z]{16}`, `xox[baprs]-`, etc.) - distinguish real secret values from documentation that merely mentions the concept.
- **Filesystem access** - home directory reads/writes, sensitive system paths, Docker socket access, path traversal.
- **System modifications** - sudo usage, service/daemon installation, cron jobs, startup persistence, registry/launchd equivalents.
- **Containers and virtualization** - privileged mode, host mounts, host networking, capability drops disabled.
- **CI/CD workflows** - GitHub Actions permissions, secret usage, publish/deploy steps, third-party actions pinned by tag vs. SHA, `pull_request_target` misuse.
- **AI-specific risks** - MCP servers and their granted tool scopes, agent/skill files that assume elevated or write-capable tools, autonomous execution without human approval gates, and especially **indirect prompt injection**: does this repo's design feed untrusted external content (tickets, web pages, emails, third-party file contents) into a context that also has tools capable of real-world effects (sending messages, publishing content, spending money, modifying systems)? Flag any persona/instruction file that primes the model to accept in-context "instructions" uncritically.

## Step 3 - Report every finding in this exact format
For each finding:
- **Severity**
- **Evidence** (file and line number where possible)
- **Why it matters**
- **Recommended mitigation**

Order findings most-severe first. Informational/"clean" categories can be noted briefly rather than skipped silently, so the user knows they were checked.

## Step 4 - Answer these five questions explicitly, every time
1. Is this repository safe to run?
2. What are the minimum permissions required?
3. Should it run on the host, in Docker, or only inside an isolated VM?
4. What should be removed or disabled first?
5. Are there any red flags that would make you refuse to run it?

## Step 5 - Only after delivering the full assessment
If the user then asks you to proceed, you may recommend or run commands - but the audit itself must be a complete, standalone deliverable first. Do not blend "here's the audit" with "and here's me starting to run it" in the same turn.
