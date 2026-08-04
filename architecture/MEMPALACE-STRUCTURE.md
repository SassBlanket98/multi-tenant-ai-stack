# MemPalace Structure — Multi-Tenant Memory Architecture

## Overview

Single source of truth for how memory is organized across the four agents (Orchestrator, Jeeves, Nova, Sage). Adapted from a simpler single-owner memory-palace design, then rebuilt specifically to solve the problem that design didn't have: **multiple agents, multiple companies, one gateway, zero tolerance for cross-contamination.**

What transferred from the single-owner version: pipeline discipline, cron-enforced automation, the knowledge-graph confirm-gate, the destructive-action gate. What didn't: a single shared palace with wing-scoped access. A single-owner deployment doesn't have a "two paying clients who must never see each other's data" problem — this one does, so the physical architecture had to change to match.

---

## 1. Palace Location & Access Model

**Three physically separate palace instances**, not one shared palace with logical partitioning:

```
palaces/
├── mempalace-orchestrator/   ← Orchestrator's private operating memory
├── mempalace-jeeves/         ← Jeeves' private operating memory
└── mempalace-shared/         ← cross-agent/department knowledge (both
                                 tenant wings) + the knowledge graph
```

Each palace is registered as its own MCP server with its own derived tool prefix (`mempalace-orchestrator__`, `mempalace-jeeves__`, `mempalace-shared__`). Agent tool configs explicitly deny the prefixes they shouldn't have:

- **Orchestrator:** full access to its own private prefix + shared; Jeeves' private prefix is structurally absent from its tool schema
- **Jeeves:** full access to its own private prefix; read-only on shared (search/get/list — no write, no delete); Orchestrator's private prefix is structurally absent
- **Nova/Sage:** no palace yet — dormant agents don't get memory infrastructure they're not using

**Verified live, not just configured:** a wrong-prefix tool call isn't caught by a runtime permission check that could theoretically be talked around — the tool simply isn't present in the calling agent's tool schema. Confirmed by direct agent-turn tests in both directions before this was considered done.

A prior single-palace version is kept registered and on disk, untouched, purely as a rollback path pending an explicit decommission decision — not deleted as part of the rebuild, in case anything downstream still depends on it.

---

## 2. Wings & Rooms

Wings live in physically separate palaces now (§1). Structure at last audit:

| Palace | Wing | Purpose |
|---|---|---|
| `mempalace-orchestrator` | `orchestrator` | Orchestrator's own diary + self-improvement notes |
| `mempalace-jeeves` | `jeeves` | Jeeves' own diary + integration notes |
| `mempalace-shared` | `ops-tenant` | Operations-side technical/operational/architecture history |
| `mempalace-shared` | `studio-tenant` | Studio-side technical history + per-client rooms |

Combined, the two tenant wings in the shared palace hold **13,000+ curated drawers** — the bulk of it technical/operational history accumulated over the engagement, with per-client rooms kept separate within the wing so a department agent's cross-department read (via search, never direct file access) stays scoped to the department it's actually allowed to see.

---

## 3. Filing Rules

- **No raw auto-mining into the palace.** Only curated content and diary summaries get filed — a prior full-palace audit found that unfiltered auto-mined content (raw session transcript, repo-mined code) made up the vast majority of a comparable single-owner palace before cleanup. Auto-mining without a filter is how a palace becomes 97% noise.
- **Ambiguous filing decisions stop and ask.** Never guess which wing or room a fact belongs to; never create a new wing without explicit approval.
- **Agent-persona content is historical reference, not live truth.** A drawer describing why an agent's design changed is framed as "decision recorded as of [date]," pointing back at that agent's live config files as the actual current source — the palace archives *why*, it doesn't replace the live config as the answer to *what's true now*.

---

## 4. Knowledge Graph — Confirm-or-Kill Protocol

1. A fact surfaces in conversation
2. Presented to a human explicitly: "Confirm or kill?"
3. Only on an explicit yes does it get written
4. When a fact changes: the old one is invalidated and the new one added — **never a silent overwrite**
5. Historical-state questions use the KG's `as_of` query, not a fresh guess

**Never auto-add facts.** This is a hard-learned rule, not a default design choice — an earlier single-owner deployment had a real incident where one person's private data leaked into another person's knowledge graph before this rule existed. The rule shipped here from day one specifically because that failure mode was already known.

---

## 5. Diary — Compressed Entity-Coded Format

Diary entries use a compressed format designed for cheap re-ingestion: short entity codes for recurring people/companies, bracketed emotion/action markers, pipe-separated fields, ISO dates, and a star-rated importance scale.

**Written after every significant event, not at session end.** Sessions end unpredictably — compaction, timeout, a crash — so waiting for a clean end-of-session signal means the entry never gets written. Diary writing is cron-backstopped (§6), never dependent on in-session discipline alone.

---

## 6. Automation Pipeline (Cron)

All routine memory maintenance runs on the cheapest capable model tier, in isolated sessions, with a restricted tool list — so hygiene work never touches or bills against the main session context.

| Job | Cadence | Scope |
|---|---|---|
| Auto-diary collection | Every 6h | Collects new session messages → writes a diary entry silently |
| Daily memory promotion | Once daily, end-of-day | Extracts meaningful diary items from the last 24h, proposes them for human approval before filing |
| Weekly diary cleanup | Weekly | Compresses routine low-importance entries into weekly summaries; high-importance entries stay as-is |
| Weekly KG-gate | Weekly | Proposes candidate knowledge-graph facts from the past week, tiered confirm / needs-eyes / pre-killed — **never auto-writes** |

Memory-promotion cadence was deliberately moved from weekly to daily ("a decision made Tuesday shouldn't wait until Monday to reach its room"), then the daily slot itself got moved later in the day after the early-morning timing started feeling like noise — a small scheduling correction, but one that mattered for whether the humans on the other end kept trusting the pipeline.

---

## 7. Destructive Action Gate

Before any destructive command or config-overwrite touching auth/config/system files: **search the palace first** for that system's history and any past corrections on the same issue. This exists specifically to stop a known mistake from being repeated a second time — an earlier deployment hit the same auth-system bug twice because this check wasn't enforced the first time it should have caught it.

---

## 8. Maintenance & Cleanup Discipline

A full audit of the largest wing (over 13,000 drawers) was run via direct SQLite classification scripts — **zero LLM/token cost** — rather than having a model read and judge each drawer individually. Result: roughly two-thirds of the audited volume was confirmed noise flagged for deletion, with the remainder batched for human review or flagged ambiguous for follow-up. Classifying at the database layer before spending any inference budget on judgment calls kept a large cleanup pass essentially free.

A separate full-text reindex job was found to be indexing an abandoned legacy data store rather than the live palace, and was disabled — the live MCP tools write directly to the palace; a second reindex step was never actually needed.

---

## 9. Self-Improvement Integration

A flat per-agent-workspace learnings file is the first capture point for corrections, errors, and feature requests — logged immediately, same reasoning as the diary rule in §5: don't wait for a session end that might not come cleanly.

When an entry gets distilled into a genuine standing rule, a short curated drawer gets filed into a dedicated self-improvement room in the shared palace: the distilled rule plus a pointer back to the source learnings file — not a duplicate of the full incident write-up, just the searchable summary. Cross-agent learnings (things any agent in the deployment should know) belong in the shared room; agent-specific quirks stay local unless they're genuinely broadly applicable.

---

## 10. Playbook Adoption — MemPalace as Priority Memory

A comparable single-owner MemPalace deployment (running longer, at larger scale) was used as a reference for a gap analysis against this system. Three gaps were identified and closed:

**1. Systematic KG population.** The knowledge graph had sat nearly empty because confirm-or-kill only fired when an agent happened to notice a fact worth flagging mid-conversation — there was no systematic extraction. A weekly two-stage job now reads the past week's diary and daily memory, extracts candidate facts with a mandatory verbatim source quote and an explicit ownership tag (who/what it's about, which tenant/department owns it), dedupes against existing KG entities, and produces a tiered digest for human review. **Nothing is written by this job** — it only ever proposes; a human still has to say yes.

**2. Promotion frequency: weekly → daily.** Rationale carried over directly from the reference deployment: a decision made mid-week shouldn't sit unpromoted until the next week's review. Diary cleanup itself stayed weekly — compressing low-importance noise doesn't need daily cadence, only fact promotion does.

**3. MemPalace stated explicitly as priority memory.** The core operating rules were rewritten so the memory palace (diary + search + KG) is the primary source of truth for durable, institutional facts — local memory files remain in place and still get written to, but are explicitly framed as backup/local-cache layers rather than the authoritative answer once the palace holds the same fact. Session-start read order was made mandatory: diary read, then search/KG query, then local files as the last resort — reversing what had been the de facto order before.

---

_Multi-tenant memory isolation only works if the boundary is enforced somewhere a prompt can't reach it. Every rule in this document exists because a specific failure mode — contamination, noise, a missed alert, a repeated mistake — was identified once and then closed structurally, not just documented against._
