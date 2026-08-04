# Multi-Tenant AI Stack — Complete Architecture

## Overview

A multi-tenant automation stack standing up department-facing AI agents for two companies sharing one gateway. Where a single-owner stack optimizes for "the agent knows everything about me," this one optimizes for the opposite instinct: **the agent knows exactly its own scope, provably, and nothing else** — because the two tenants on this gateway are separate paying clients, not one person's separate life contexts.

**Core principle:** Isolation is a runtime property enforced by the gateway, not a behavior requested of the model.

---

## Component Architecture

### 1. OpenClaw Gateway — Central Control
**Role:** Self-hosted orchestration platform for all four agent configs (two live in production, two built and paused for rollout)

**Responsibilities:**
- Gateway process — manages tool access, agent lifecycle, channel routing
- Per-agent tool profiles (`minimal` + explicit allowlist, not "everything unless denied")
- Filesystem scoping (`fs.allowPaths`) enforced independently of tool allowlisting
- Skill Workshop — governed proposal/review/apply/rollback pipeline for capability installs
- Multi-channel routing — Slack, Discord, Telegram bindings to specific agents

**Design decision:** Every agent's capability surface is defined by what's explicitly granted, never by what isn't explicitly denied. A new department agent starts with nothing and earns tools one line at a time.

---

### 2. MemPalace — Physically Isolated Memory
**Role:** Multi-tenant long-term memory with hard structural isolation

**Architecture:**
- Three physical palace instances: 2 private (Orchestrator, Jeeves) + 1 shared (cross-agent, wing-scoped per tenant)
- Tool-schema-level exclusion — an agent literally cannot address a palace prefix it wasn't granted, verified by direct testing rather than assumed from config
- Knowledge graph lives in the shared palace, confirm-or-kill on every write
- AAAK-style compressed diary format, cron-backstopped

**Why physical separation over logical scoping?**
- A prompt can be talked around; a tool that isn't in the schema can't be invoked at all
- Two paying clients on one gateway is a fundamentally different risk profile than one person's separate contexts — the isolation bar has to be "provably can't leak," not "shouldn't leak"
- Full detail in [`MEMPALACE-STRUCTURE.md`](MEMPALACE-STRUCTURE.md)

---

### 3. Model Delegation — Two-Tier Cost Control
**Role:** Route judgment work and execution work to appropriately-priced models

**Setup:**
- Planning/analysis tier: a large-context model handles investigation, research, and judgment calls directly
- Execution tier: a cheaper model gets spawned as a subagent for the mechanical work of implementing an already-made plan
- The planning tier reviews and corrects the execution tier's output directly rather than re-delegating small fixes
- Bulk/mechanical/pagination-style work is routed to a `$0` script before either model tier touches it

**Why this split?**
- Judgment doesn't parallelize or cheapen well; mechanical execution does
- A subagent spawned for "do the thing that's already been figured out" is a legitimate cost-saver; a subagent spawned for raw data pagination is money spent on a problem a script already solves for free
- All cron/maintenance jobs (diary compression, weekly cleanup, security polling) are pinned to the execution tier for the same cost-discipline reason — routine automation shouldn't run on the expensive model by default

**Provider resilience:** The stack migrated providers mid-engagement after recurring billing/credit issues with the original provider. Because model selection lives in gateway config rather than being hardcoded into agent logic, the switch was a config change, not a rebuild.

---

### 4. Department-Scoped Agents — Nova & Sage Pattern
**Role:** Team-facing automation, not operator-facing tooling

**Setup:**
- `tools.profile: "minimal"` — zero tools by default
- Explicit `alsoAllow` list per agent, scoped to exactly what the role needs (research + content tools for a marketing agent; monitoring + suggestion tools for a social agent, deliberately excluding direct-post capability)
- `fs.allowPaths` restricting file read/write/edit to one department's folder tree — enforced at the OpenClaw runtime level, checked before every file operation

**Enforcement flow:**
```
Agent attempts a file write
  → OpenClaw checks the path against fs.allowPaths
  → In scope: proceed
  → Out of scope: blocked with a clear error, before the write ever happens
```

**Design decision:** These are department tools, not operator tools. They post to their team's channel, not to the person who built them — approval and final decisions sit with the team lead, not with the consultant who deployed the agent. That's a deliberate scaling choice: an automation model that routes every output back through the operator doesn't survive past the second department.

---

### 5. Skill Workshop — Governed Capability Installation
**Role:** Turn "install a new skill" from a judgment call into a reviewable, reversible process

**Pipeline:** propose → review → approve → apply, with every install producing a `PROPOSAL.md`, a `proposal.json`, and a `rollback.json`

**Why this matters in practice, not just in theory:**
A skill delivery attempted via a raw `curl` command mid-conversation — a classic prompt-injection delivery pattern — was independently caught and blocked on two separate channels by two different agent sessions before the same skill was later fetched, read in full, and properly installed through the workshop pipeline. The governance overhead paid for itself the first time it mattered, not hypothetically.

---

### 6. Multi-Channel Routing — Slack, Discord, Telegram
**Role:** Message gateway across three platforms, one gateway process

**Architecture:**
- Slack via Socket Mode as the primary department-facing channel, per-channel `requireMention` and allowlist policies
- Discord and Telegram as secondary channels, both pairing-gated for DMs
- Binding rules route by channel type to a specific agent — currently all Slack and Discord traffic routes to the executive-assistant agent as the single point of contact, ahead of department-agent channel activation

**Design decision:** Channel and agent are decoupled in config — adding a new department's Slack channel and pointing it at a new agent is a binding-rule change, not a code change.

---

### 7. Live Security Monitoring — Production Automation
**Role:** Read-only, credential-isolated alert triage across a multi-site hosting footprint

**Setup:**
- Single-purpose credential (Microsoft Graph, `Mail.Read` only) scoped by an Exchange Application Access Policy to exactly one mailbox — verified denied everywhere else in the tenant
- 30-minute poll cadence, watermark-based (never replays history, never double-reports)
- Severity classification with an explicit periodic-digest filter to prevent a routine newsletter from false-positiving into a live page
- Quiet-hours batching — Critical bypasses gating entirely; everything else during off-hours rolls into one digest at the next business-hours window
- The credential-holding script never sends a message itself — it emits structured output that the agent posts through its own normal, audited path

**Cost impact:** Turns "dozens of alert emails a day" into "a handful of pages that actually matter, plus one digest," without a human ever having to triage the noise manually.

---

## Data Flow Architecture

### Department Request (Nova/Sage pattern)
```
Team member posts in department Slack channel
    ↓
OpenClaw routes to the bound department agent
    ↓
Agent checks tool profile + fs.allowPaths before any action
    ↓
Research / content generation (as permitted)
    ↓
Draft saved to department's own folder (enforced path)
    ↓
Agent posts to the department channel — not to the operator
    ↓
Team lead reviews and approves
```

### Scoped Executive Assist (Jeeves pattern)
```
PM asks a question about a client or department within their own company
    ↓
Jeeves checks session → internal memory → shared palace search
    ↓
Full read/write on that company's own rooms, across every department
    ↓
Answer delivered with explicit source-tier citation
    ↓
No path into the other tenant's wing at all — not read, not write
```

### Security Alert Pipeline
```
Alert email arrives in the monitored mailbox
    ↓
30-minute poll picks it up (watermark-based, no replay)
    ↓
Periodic-digest check → severity classification → noise suppression
    ↓
Critical: post immediately, any hour
High: post immediately during business hours, else queue
Medium/Low: always queue outside business hours
    ↓
Script emits JSON; agent posts through its own messaging path
    ↓
Digest fires once at the next business-hours window if anything queued
```

---

## Security Architecture

### Tenant Isolation
- Two companies, physically separate memory wings within the shared palace
- Every department agent's contract states the hard rule explicitly: no cross-tenant access, ever — backed by the same enforcement mechanism as department scoping, not left as policy alone

### Tool & File Scoping
- Department agents: minimal-by-default tool profile + explicit allowlist + filesystem path restriction, all three enforced independently
- A blocked action fails with a clear error before it executes — never a silent no-op, never a workaround the agent could reason its way into

### Credential Isolation
- Each integration credential is scoped to exactly the resource it needs (one mailbox, read-only) and never reused across agents or purposes
- Send capability is deliberately separated from credential-holding wherever a monitoring pipeline exists — the piece that can read is never the piece that can post externally

### Knowledge Graph Integrity
- Confirm-or-kill on every write, no exceptions
- Facts that change get invalidated and re-added, never silently overwritten, preserving an accurate historical record

---

## Scaling Design

### Adding a New Department
1. Give the new agent its own `fs.allowPaths` scoped to its own folder tree
2. Grant a minimal tool profile, expand deliberately as real needs surface
3. Bind its Slack channel once the team is ready — no code change, config only
4. Add memory infrastructure (its own palace, if warranted) only once it's actually running, not speculatively

### Adding a New Tenant
The current two-tenant wing structure inside the shared palace generalizes directly — a third company would get its own wing, its own department folder trees, and the same tool-schema-level exclusion pattern already proven for the first two.

### Design Principle
**Isolation is cheap to build in from the start and expensive to retrofit after the first cross-contamination incident.** Every structural boundary in this stack was built before it was strictly needed, not after.

---

## Dependency Summary

| Component | Purpose | Local? |
|---|---|---|
| OpenClaw | Gateway, orchestration, scoping enforcement | Yes |
| MemPalace (×3) | Physically isolated memory | Yes |
| Planning-tier model | Analysis, judgment, direct execution | No (API) |
| Execution-tier model | Subagent grunt work | No (API) |
| Local embedding model | Memory search | Yes |
| Slack / Discord / Telegram | Channel interfaces | No (services) |
| ClickUp | Live task/project data | No (API) |
| Microsoft Graph | Read-only security-alert monitoring | No (API) |
| Skill Workshop | Governed capability installs | Yes |

---

## Why This Stack

**Multi-tenant AI needs different priorities than single-owner AI:**

1. **Provable isolation > convenient scoping** — the boundary has to survive a hostile prompt, not just a well-behaved one
2. **Team ownership > operator convenience** — automation that scales past one department has to report to the team that owns the work
3. **Governed capability growth > move-fast installs** — a review pipeline that catches one real injection attempt has already paid for its overhead
4. **Credential minimalism > integration convenience** — one mailbox, one scope, one purpose, every time
5. **Structural enforcement > documented policy** — every rule in this stack that matters is backed by something a prompt literally cannot talk around

This stack prioritizes all five, in that order.
