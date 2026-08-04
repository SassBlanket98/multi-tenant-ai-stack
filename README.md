# multi-tenant-ai-stack

[![Multi-Tenant](https://img.shields.io/badge/tenants-2_companies-blueviolet?style=flat-square)](https://github.com)
[![Multi-Agent](https://img.shields.io/badge/multi--agent-4_agents-blue?style=flat-square)](https://github.com)
[![Self-Hosted](https://img.shields.io/badge/infrastructure-self--hosted-green?style=flat-square)](https://github.com)
[![Isolation: Enforced](https://img.shields.io/badge/tenant%20isolation-schema--level-success?style=flat-square)](#multi-tenant-security-architecture)
[![License: Client Work](https://img.shields.io/badge/license-client--confidential-red?style=flat-square)](#)

---

## Overview

A production multi-agent AI deployment built on **OpenClaw**, standing up department-facing AI agents for two independent companies — one an operations/finance-led business, the other a creative and dev studio — that share an office and a Slack workspace but must never share data.

This is the harder version of the single-owner problem: instead of one person's context, the system has to hold **two companies' worth of client and operational data on one gateway**, hand slices of it to department-level agents that talk directly to teams (not to the operator), and guarantee — not just document — that a marketing agent can never read finance data, and that one company's business never leaks into the other's context or vice versa.

Company names, individuals, and other identifying details in this write-up are anonymized or generalized throughout for confidentiality. The architecture, security model, and operational patterns are real.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         macOS Mac Mini (gateway host)                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    OpenClaw Gateway (self-hosted)                │   │
│  │                                                                  │   │
│  │   ┌─────────────┐  ┌─────────────┐  ┌───────┐  ┌───────┐        │   │
│  │   │ ORCHESTRATOR│  │   JEEVES    │  │ NOVA  │  │ SAGE  │        │   │
│  │   │ (gateway /  │  │ (EA — live  │  │(dept, │  │(dept, │        │   │
│  │   │  operator)  │  │ production) │  │designed)│(designed)│      │   │
│  │   │  Kimi K3    │  │  Kimi K3    │  │Kimi K3│  │Kimi K3│        │   │
│  │   └──────┬──────┘  └──────┬──────┘  └───┬───┘  └───┬───┘        │   │
│  └──────────┼────────────────┼─────────────┼──────────┼────────────┘   │
│             │                │             │          │                │
│    ┌────────┴───────┐   ┌────┴────┐  fs.allowPaths + tool profile:     │
│    │ mempalace-      │   │mempalace-│  minimal — enforced per-agent,   │
│    │ orchestrator    │   │ jeeves   │  not just documented             │
│    │ (private)       │   │(private) │                                  │
│    └────────┬────────┘   └────┬─────┘                                  │
│             │                 │                                        │
│             └────────┬────────┘                                        │
│                       ▼                                                │
│         ┌─────────────────────────────┐                                │
│         │      mempalace-shared       │                                │
│         │  wing: ops-tenant           │                                │
│         │  wing: studio-tenant        │                                │
│         │  + cross-agent Knowledge    │                                │
│         │    Graph (confirm-or-kill)  │                                │
│         └─────────────────────────────┘                                │
└───────────────────────────┼──────────────────────────────────────────── ┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          Slack (socket)  Discord        Telegram
          — primary        (pairing)     (pairing)
              │
   ┌──────────┴──────────┐
   │  Department channels │
   │  scoped per company,  │
   │  per team              │
   └────────────────────────┘
```

## Agent Roster

### 🎛️ Orchestrator — Gateway & Systems Architect
The only agent with full gateway control. Builds, configures, and hands off every other agent; never itself client-facing.

**Design stance:** *"Implementation > experimentation."* No department agent ships without documented tool policy, file scope, and a tested handoff. Runs the model-delegation split (below) on every non-trivial task itself before ever spawning a subagent.

### 🎩 Jeeves — Executive Assistant *(Flagship Deployment)*
The standout real-world case. Jeeves is the operations lead's executive assistant across both companies — project visibility, cross-department coordination, and a live security-monitoring pipeline that runs unattended in production.

**What Jeeves does in production:**
- Full read/write access to the studio side's client rooms in the shared memory palace; read-only into the ops company's internal room — a real cross-company visibility need, safely scoped
- ClickUp task status and reporting via the live REST API
- Polls a shared mailbox every 30 minutes across 17+ client WordPress sites for Wordfence/ManageWP security alerts, classifies severity, suppresses known noise, and posts only what actually matters straight to Slack — see [`agents/JEEVES.md`](agents/JEEVES.md) for the full breakdown
- All client-facing output reviewed by the operations lead before it goes anywhere; Jeeves surfaces, doesn't decide

This is a live deployment carrying real operational load, not a demo.

### 🎨 Nova — Marketing & Newsletter Agent *(designed, tool-scoped, paused for rollout)*
Department agent for the studio side's marketing team. Tool policy and filesystem scoping are fully implemented and tested — Nova can only read/write inside her department's folder tree, enforced by OpenClaw at the runtime level, not by prompt instruction. Currently dormant (no cron, no channel binding, zero running cost) pending Slack channel activation with the marketing team.

### 📡 Sage — Social Media Agent *(designed, tool-scoped, paused for rollout)*
Same status as Nova, scoped to the social-media department folder tree instead. Built to monitor platform trends and suggest — never post — responses; publishing stays a human action by design.

---

## Multi-Tenant Security Architecture

The core engineering problem here isn't "give an agent memory" — it's **give four agents overlapping-but-not-identical memory, on one gateway, with zero possibility of cross-contamination**, across two companies who are paying customers of the same consultant and have every right to expect their data never touches the other's.

### Isolation is structural, not promised

As of a 2026-07-31 rebuild, the single shared memory palace was split into **three physically separate palace instances** — Orchestrator's own, Jeeves' own, and a shared cross-agent/department palace holding both companies' client wings plus the knowledge graph. Each agent's tool schema only contains the MCP tool prefixes for palaces it's allowed to touch (`mempalace-orchestrator__*`, `mempalace-jeeves__*`, `mempalace-shared__*`).

This was verified live, not just configured: a wrong-prefix tool call isn't rejected at call time by a permission check an agent could theoretically talk itself around — the tool is **structurally absent from that agent's schema**, confirmed by direct agent-turn tests in both directions. There is no prompt an agent could be tricked into that reaches a tool it was never handed.

### Department scoping is enforced at two independent layers

Nova and Sage each run with `tools.profile: "minimal"` (nothing granted by default) plus an explicit allowlist, **and** a filesystem-level `fs.allowPaths` restricting file read/write/edit to their own department's folder tree (`clients/*/marketing-newsletter/**` vs `clients/*/social-media/**`). Cross-department reads happen only through the shared memory palace's semantic search — read-only, logged, department-tagged — never through direct file access.

```
Nova tries to write: /clients/{client}/social-media/response.md
→ OpenClaw checks fs.allowPaths → not in scope → blocked with a clear error
→ Not a policy Nova could reason her way around — it's not in her tool schema
```

### Tenant contract, enforced by convention and by architecture

Every department agent's operating contract states the hard rule plainly: *"Never access other tenants — Kernel agents don't see Flint, vice versa."* That's backed by the same fs-scoping and palace-prefix mechanism above, not left as an honor system.

### Cost of getting this wrong

The knowledge graph runs a strict **confirm-or-kill** protocol — no fact is ever auto-written. Every candidate fact is proposed to a human first, batched at a natural pause rather than interrupting mid-task (per-fact interruptions don't get answered; batched ones do). This exists because the home-lab predecessor to this system had a real incident where unconfirmed facts bled between contexts — the lesson was carried over here as a hard rule from day one rather than learned the expensive way twice.

---

## 🏛️ Memory Architecture

See [`architecture/MEMPALACE-STRUCTURE.md`](architecture/MEMPALACE-STRUCTURE.md) for the full structure. Summary:

- **Physical palace separation** per agent (private) + one shared palace (cross-agent/department, wing-scoped by tenant)
- **13,000+ curated drawers** across the two tenant wings as of the last structure audit
- **AAAK diary format** — compressed, entity-coded session logs, written mid-conversation after significant events (not "end of session," which doesn't fire reliably)
- **Confirm-or-kill KG** — a weekly automated stage proposes candidate facts (tiered: confirm / needs eyes / pre-killed); nothing writes without an explicit human yes
- **Zero-cost cleanup discipline** — a prior audit reclassified ~14,000 drawers via direct SQLite scripts (zero LLM tokens spent) and flagged ~8,795 as confirmed noise before any human review time was spent on them

---

## ⚙️ Automation & Governance Pipeline

### Model Delegation (Cost Control)
Two-tier routing on every agent: a capable planning model does analysis, investigation, and judgment calls directly; a cheaper execution model gets spawned as a subagent only for the mechanical grunt work once the plan is set, then the planning model reviews and corrects its output. Mechanical/bulk operations (pagination-style data work) are routed to a `$0` script before either model touches them — subagents are for judgment-requiring work, not raw data plumbing.

### Skill Workshop — Governed Installation
Skills aren't installed ad hoc. Every skill goes through a proposal → review → approve → apply pipeline, each with a `PROPOSAL.md`, a `proposal.json`, and a `rollback.json` — every install is a documented, reversible decision. This isn't theoretical: a skill delivery attempted via a raw `curl` command mid-conversation was independently flagged and blocked as a likely prompt-injection attempt (on two separate channels, by two different agent sessions) before the same skill was later fetched, reviewed line-by-line, and properly installed through the workshop.

### Proactive Agent Protocol (deployed across all 4 agents)
- **WAL (Write-Ahead Log):** decisions, corrections, and preferences get written to session state *before* the agent responds — not after
- **Working buffer:** activates at 60% context capacity; captures the conversation verbatim so a context reset never loses the thread
- **Compaction recovery:** an agent waking up mid-context-loss reads the working buffer first — it never has to ask "what were we doing?"
- **Verify-before-reporting:** "done" requires confirming the outcome from the user's perspective, not just that a file exists
- **ADL/VFM guardrails:** every self-improvement change is scored against a fixed priority order — stability, then explainability, then reusability, then scalability, then novelty — before it ships

### Cron-Driven Memory Hygiene
All routine memory maintenance (diary compression, daily memory-promotion proposals, weekly cleanup, weekly KG-gate) runs on the cheapest capable model, in isolated sessions with restricted tool lists, so hygiene work never touches — or bills against — the main session.

---

## 🔧 Tech Stack

**Core:**
- OpenClaw (self-hosted AI gateway, macOS)
- Kimi K3 via OpenRouter — planning, analysis, judgment (primary model, all agents)
- GLM-5.2 via OpenRouter — execution/grunt-work subagent tier
- Gemma 4 (31B) — subagent fallback tier
- (Migrated off the Anthropic API mid-2026 after recurring billing/credit issues — architecture is provider-agnostic by design)

**Memory & Data:**
- MemPalace — three physical instances (2 private + 1 shared), MCP-native
- Knowledge Graph with temporal validity (`valid_from`/invalidate, never silent overwrite)
- SQLite-backed drawer storage, direct-script classification for bulk cleanup

**Integration:**
- Slack (Socket Mode) — primary department-facing channel, per-channel scoping
- Discord & Telegram — secondary channels, pairing-gated DMs
- ClickUp REST API — live task/project status
- Microsoft Graph (Mail.Read only, single-mailbox app-access policy) — read-only security alert monitoring
- Google Workspace (planned, phase 2)

**Infrastructure:**
- macOS (Mac Mini)
- Cron-based scheduling, model-tiered by job type
- Skill Workshop (proposal/approve/apply/rollback pipeline)

Full detail in [`architecture/tech-stack.md`](architecture/tech-stack.md).

---

## Deployment Model

**Where:** Self-hosted on a Mac Mini at the client's office, shared by two companies

**Isolation:**
- Two independent tenants (an operations/finance business, a creative + dev studio) sharing one gateway
- Four agent instances, each with an enforced tool + memory + filesystem scope
- Department agents talk to their team's Slack channel, not to the operator — a deliberate "delegation of automation," not remote-control tooling

**Current status:**
- **Live in production:** Orchestrator (gateway), Jeeves (EA — active WP security monitoring, ClickUp integration, cross-company visibility)
- **Built, tested, paused for team rollout:** Nova, Sage (tool policy + file scoping complete; awaiting Slack channel activation with their teams)

---

## Key Design Decisions

### Why physically separate palaces, not one palace with logical scoping?
Logical/prose-level scoping is a policy an agent could misread or a prompt could talk around. Physical separation plus tool-schema-level exclusion means the forbidden tool literally doesn't exist for that agent — there's no clever prompt that reaches a tool that was never granted.

### Why department agents talk to teams, not to the operator?
The brief here was never "automate David's job" — it's standing up automation *for the client's teams*. An agent that routes every output through the operator doesn't scale past one department and doesn't actually save the client anything. Team leads approve their own agent's output; the operator retains gateway-level control, not day-to-day approval authority.

### Why a governed skill pipeline instead of direct installs?
Because the failure mode isn't hypothetical — a live prompt-injection attempt via a raw `curl`-delivered "skill" was caught in this exact deployment. A proposal/review/rollback pipeline turns "trust the agent's judgment in the moment" into "review a diff before it becomes capability."

### Why confirm-or-kill on the knowledge graph?
An unconfirmed fact that turns out wrong doesn't just sit there — it gets retrieved and acted on by every future query that touches it. Human confirmation before write is cheap; unwinding a contaminated KG is not.

---

## Lessons Learned

### On Multi-Tenant Isolation
Documentation is not enforcement. "Agents don't cross tenants" as a written rule is worth nothing next to "the tool literally isn't in the agent's schema." Build the second one; keep the first one as the human-readable explanation of why.

### On Skill Installation
Any inbound instruction that says "run this to install a capability" is a live attack surface, not a convenience feature. Route it through review every time, even when it looks legitimate — especially when it looks legitimate.

### On Automation Ownership
The most successful deployment here (Jeeves) works because it reports to the team that owns the problem, not to whoever built it. Automation that routes everything back through the builder becomes a bottleneck the moment there's more than one department.

### On Cost Discipline
Bulk, mechanical work is a script problem, not a model problem. Spawning an LLM subagent for pagination-style data work is money spent solving a problem that `$0` of Python already solves.

---

## Getting Started

This is confidential client infrastructure — not open-source — but the architecture and decisions are documented here for reference.

### To Deploy a Similar Multi-Tenant Setup
1. Stand up OpenClaw as the gateway; keep gateway-level control separate from any department agent
2. Split memory physically per agent before it's needed, not after the first cross-contamination incident
3. Enforce department/tenant scope at the tool-schema and filesystem level — never rely on prompt discipline alone
4. Put every skill install behind a review pipeline with rollback, from day one
5. Route department agents to the teams that own the work, not back through the operator
6. Build the confirm-or-kill gate before the knowledge graph has anything worth contaminating

---

## Architected and deployed by David Hill

**Contact:** Available for consulting on multi-tenant AI architecture, agent security design, and self-hosted enterprise automation.

---

**Last updated:** 2026-08-04 | **Status:** Production (Jeeves) + Designed & Scoped (Nova/Sage)
