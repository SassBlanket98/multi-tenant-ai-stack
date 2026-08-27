# multi-tenant-ai-stack

## Overview

A production multi-agent deployment built on OpenClaw, standing up department-facing AI agents for two independent companies that share infrastructure but need their own data kept separate. One gateway supports two companies' operational context while keeping access deliberately scoped.

Company names and other identifying details are anonymized throughout. The architecture and decisions described here are real.

## Architecture, in short

Each agent runs on OpenClaw with its own tool schema and its own slice of shared memory, built on MemPalace (open source). An agent only has the tools and memory access its role needs. Tools it shouldn't use aren't offered as an option, rather than relying on a policy telling it not to use them. Department agents report to the teams that own the work, not back through the operator, so approvals and corrections happen at the team level instead of bottlenecking through whoever built the system.

This scoping isn't a finished, closed problem. It's something I keep reviewing and tightening as gaps turn up, not a claim that crossover between tenants is impossible.

## Agent roster

- **Orchestrator**: gateway and systems architect. The only agent with full gateway control; never itself client-facing.
- **Jeeves**: the flagship deployment, doing real unattended work for one of the two companies: day-to-day PM and admin support, reviewed by that company's own PM before anything client-facing goes out.
- **Nova, Sage**: department-scoped agents, built and tool-scoped, currently paused pending team rollout.

## Current status

Two agents (Orchestrator, Jeeves) are live in production. Two (Nova, Sage) are built and scoped but not yet switched on.

## Key design decisions

**Why department agents talk to their own teams, not back through the operator.** An agent that routes every output through the person who built it doesn't scale past one team and doesn't save anyone time. Team leads review their own agent's output; the operator keeps gateway-level control, not day-to-day approval authority.

**Why tool scoping over policy alone.** A written rule saying "don't touch the other tenant's data" is worth less than a tool that simply isn't available to call. I lean on the second wherever I can, and treat the written rule as documentation of intent, not as the enforcement mechanism itself.

**Why this isn't described as "solved."** Any claim of total isolation on a live, evolving system is either untested or temporary. What actually holds this together is scoping decisions made deliberately, reviewed regularly, and tightened when something looks off, not a one-time design that gets assumed to be airtight forever.

## Lessons

- Documentation isn't enforcement. A written rule that agents don't cross tenants is worth much less than a tool that literally isn't available to call. Build the second one, and keep the first as the explanation of intent.
- Automation should report to whoever owns the problem, not to whoever built it. That's what keeps it from becoming a bottleneck once more than one team is involved.

## Tech stack

- OpenClaw (self-hosted AI gateway)
- MemPalace (open source) for agent memory
- Slack as the primary department-facing channel; Discord and Telegram for operator access
- Model routing tiered by task complexity

## Contact

Architected and deployed by David Hill. Available for consulting on multi-agent AI architecture and agent security design.

---

**Last updated:** 2026-08-27
