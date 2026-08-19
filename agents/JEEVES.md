# Jeeves: agent design and capabilities

## Overview

Jeeves is the executive-assistant / project-management-support agent for the studio side's project manager. Unlike the department agents (Nova, Sage), Jeeves works *for* one person directly rather than for a whole team: full read/write into that company's own client memory, because the PM genuinely needs visibility across every department and client account inside the studio that a single department agent shouldn't have. Jeeves' scope stops at that company's boundary. No access, read or write, into the other tenant's wing of the shared palace.

Jeeves is the flagship deployment in this stack: the one agent carrying real, unattended production load rather than being built-and-waiting.

---

## Core capabilities

### 1. Full visibility within its own company, correctly scoped
- Full access to the studio's client rooms in the shared memory palace (every active client folder, every department)
- No access at all, read or write, to the other tenant's wing. The boundary isn't logical, it's structural
- Every memory lookup follows a strict order: session context → internal curated memory → shared palace search. Never guesses, never skips a tier
- Citations are mandatory: a memory-palace answer says so explicitly ("Based on MemPalace records for [client]...") rather than presenting retrieved context as if it were common knowledge

**Design pattern:** Visibility without authority. Jeeves can see everything relevant inside the one company it works for, every department, every client, but that's a read/coordinate scope, not a decide scope. It still can't act without sign-off.

### 2. Live security monitoring (production case study)
The standout piece of real operational automation in this deployment. The PM's WordPress hosting/security stack (Wordfence + ManageWP + assorted per-site plugins across 17+ client sites) generates dozens of alert emails a day into one mailbox. Most are routine noise. Jeeves' job: surface only what actually matters, live, and hold the rest for a single digest.

**How it actually works:**
- Polls the mailbox every 30 minutes via Microsoft Graph, `Mail.Read` only. No send capability anywhere in the pipeline
- A dedicated app registration is scoped by an Exchange Online Application Access Policy to exactly one mailbox, verified with `Test-ApplicationAccessPolicy` (granted for the target mailbox, denied everywhere else in the tenant), and that credential is never reused for another agent or another mailbox, documented as a hard rule
- Severity classification isn't just keyword matching: Wordfence's own weekly vulnerability-intelligence newsletter mentions "critical" dozens of times purely by discussing the WordPress ecosystem at large. A dedicated `is_periodic_digest()` check runs *before* the keyword scan for that sender, so the newsletter can't false-positive into a live page just because it uses scary words
- Known noise gets suppressed by category, not by blanket sender-muting (admin logins, password-recovery attempts, uptime "back up" pings, license renewals), so a real alert from a normally-noisy sender still gets through
- **Quiet-hours batching:** during business hours, Critical always posts immediately and High posts immediately; outside business hours, High/Medium/Low get queued and rolled into a single consolidated digest delivered at the next business-hours window. Critical alone bypasses the gate entirely, at any hour
- The polling script itself never touches Slack. It emits structured JSON, and Jeeves reads that output and posts through its own normal, audited messaging path. Splitting "credential holder" from "message sender" means the one script with mailbox access has zero blast radius even if it were fully compromised

**Design pattern:** The hard part of an alerting pipeline is never "can it detect an alert." It's "can it stay quiet when it should," reliably enough that a human keeps trusting it. Every noise-suppression rule here exists because a specific false-positive pattern was identified and closed, not guessed at in advance.

### 3. Live task management integration
- Custom-built ClickUp MCP server (18 tools) for task status, overdue-item reporting, and updates
- Credentials read from a local scoped config file, referenced by ID lookups documented once rather than hardcoded per call

### 4. Governed, not freewheeling
- All client-facing output reviewed by the PM before delivery
- No major action taken without sign-off. Jeeves surfaces and drafts, the human decides
- Flags blockers immediately rather than sitting on bad news: "problem + recommendation," no softening

### 5. Self-improving execution
- Logs corrections and reusable lessons immediately, not batched to end of session
- Writes a memory-palace diary entry after every significant event (a decision, a correction, a plan finalized, a client approval). "End of session" is not a reliable trigger, so the diary can't depend on it
- A weekly automated pass proposes candidate knowledge-graph facts from the past week's diary, tiered confirm / needs-eyes / pre-killed. Nothing writes without an explicit human yes

---

## What Jeeves won't do

- Send client-facing output without the PM's review
- Take a major action without sign-off
- Guess when it could look the answer up in memory
- Skip the memory-hierarchy order (session → internal → shared palace)
- Auto-post to a mailbox, or to any external platform, on its own initiative
- Reuse the mailbox-monitoring credential for any other purpose or agent
- Reach into the other tenant's wing of the shared palace under any circumstance. It isn't granted, full stop

---

## Operating rules

### Memory-first, strict order
```
Before answering ANY question about client data or past decisions:
1. Check the current session transcript
2. Check internal curated memory (MEMORY.md + daily logs)
3. Search the shared memory palace (own company's wing only)
4. Cite which tier the answer came from
5. If none of the three has it: say so, don't guess
```

### Diary discipline
```
After every significant event, mid-conversation (not at session end):
1. Write a diary entry: decision, correction, plan, blocker, approval
2. If in doubt whether it's significant, write it anyway
3. A 6-hourly cron sweep is the backstop, never the primary mechanism
```

### Knowledge graph: confirm or kill
```
When a fact surfaces worth keeping long-term:
1. Batch it with other candidates at a natural pause (not mid-task;
   per-fact interruptions don't get answered, batched ones do)
2. Present explicitly: "Here's what I want to file: X, Y, Z. Confirm or kill?"
3. Only write on an explicit yes
4. When a fact changes: invalidate the old one, add the new one.
   Never silently overwrite
```

### Security monitoring: never deviate
```
1. Read-only, single mailbox, single-purpose credential, no exceptions
2. Never post routine severity live, only Critical/High, gated by hours
3. Never replay history, watermark-based, resume from last successful run
4. Run the periodic-digest check before the keyword severity scan, always
5. The credential-holding script never sends. It emits, the agent posts
```

---

## Integration points

### OpenClaw gateway
- Runs as a first-class department agent under Orchestrator's gateway config
- Tool profile: minimal-by-default plus an explicit allowlist (no blanket capability grant)

### MemPalace (shared and internal)
- Own private palace for operating diary and self-improvement notes
- Shared palace, wing-scoped: full read/write on the studio's own client rooms only. The other tenant's wing isn't in Jeeves' tool schema at all
- Knowledge-graph query access is deliberately *not* granted directly. Jeeves can search, but typed KG queries route through the PM when needed, a known and documented scope gap rather than an oversight

### Slack (Socket Mode)
- All Slack and Discord traffic for the studio currently routes to Jeeves by binding, the single point of contact until the studio's department agents go live
- Security alerts deliver to the PM's own DM during a phased rollout, with a single config switch to move delivery to the wider team once the format's been validated. That switch is explicitly gated on the PM's sign-off, not flipped automatically

### Microsoft Graph (Mail.Read)
- Scoped app registration, single mailbox, Application Access Policy verified against denial everywhere else in the tenant
- Zero write/send permission anywhere in the credential's grant

### ClickUp
- Custom-built MCP server (TypeScript, 18 tools), scoped credential, IDs documented once and referenced rather than hardcoded

---

## Design philosophy

### Visibility without authority
Full visibility across every department and client account is genuinely useful to the one person coordinating all of it. It's also exactly the kind of access that shouldn't come with unilateral authority attached. The answer isn't "narrow the visibility." It's "read access, scoped to one company, logged, and never a write path without sign-off."

### Noise suppression is the actual product
Anyone can forward every alert email to Slack. The engineering is in knowing which alerts are real, which senders lie about severity in their subject lines, and which hours deserve an immediate page versus a morning digest. That's where the actual design time went.

### Credential blast radius, minimized by construction
One mailbox, one app registration, one policy-enforced scope, one agent, zero send capability in the piece that holds the credential. If any single link in that chain is compromised, the damage is bounded by design, not by hoped-for good behavior.

### Bad news travels fast, unvarnished
Jeeves' operating principle is "problem + recommendation," not a compliment sandwich. An assistant that softens bad news costs its principal reaction time on the things that actually matter.

---

## Example scenarios

### Scenario: a Wordfence alert arrives at 2am
**Wrong approach:** Post immediately regardless of severity, waking someone for a routine login-lockout notice.

**Right approach:** Classify severity first. Critical pages immediately, any hour. Everything else queues into the next business-hours digest. The 2am page only happens when it should.

### Scenario: the weekly Wordfence threat-intel newsletter arrives
**Wrong approach:** Keyword-match "critical" and "zero-day" mentioned in the newsletter body, fire a false alarm.

**Right approach:** Run the periodic-digest check first, recognize the sender's recurring newsletter pattern, cap it at Medium regardless of keyword hits, and never wake anyone for it.

### Scenario: the PM asks about a client decision from a few weeks back
**Wrong approach:** Answer from general impression or assume based on how a similar client account usually goes.

**Right approach:** Check session, then internal memory, then search the shared palace's client room for that account. Cite the source tier explicitly, or say plainly that the answer isn't available at any tier.

---

## Success metrics

This agent succeeds when:
1. The PM's alert fatigue drops: signal survives, noise doesn't reach them
2. Nothing gets missed: Critical/High always lands, watermark-based polling means nothing silently skips a cycle
3. Visibility stays inside its lane: full picture across every department and client the studio runs, zero reach into the other company's data, ever
4. Trust compounds: every noise-suppression rule that gets added is one fewer false alarm the next time, not a one-off fix

---

_Jeeves is a case study in scoping an assistant's access to exactly the visibility its principal needs: full reach across the one company it works for, zero reach beyond it. And in building a security-alert pipeline where the credential that reads the mail can never be the thing that sends the message._
