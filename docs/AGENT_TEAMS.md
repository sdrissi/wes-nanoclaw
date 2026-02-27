# Autonomous Product Teams

NanoClaw can run each of your products as an autonomous AI startup team. The system discovers work from your metrics and error tracking, builds fixes and features, validates them through CI and AI code review, and notifies you on WhatsApp when PRs are ready to merge. You review and merge — everything else is autonomous.

For implementation details, see [plans/AGENT_TEAM_OPTION_2.md](plans/AGENT_TEAM_OPTION_2.md).

---

## How It Works

You have one WhatsApp group per product. Each product runs an autonomous loop:

```
Every 10 minutes:

  ┌──────────┐     ┌────────────┐     ┌──────────┐     ┌──────────┐
  │  SCAN    │────►│ PRIORITIZE │────►│  BUILD   │────►│  NOTIFY  │
  │          │     │            │     │          │     │          │
  │ PostHog  │     │ Pick work  │     │ Architect│     │ PR ready │
  │ Sentry   │     │ items by   │     │ plans,   │     │ to merge │
  │ Feedback │     │ impact     │     │ dev does │     │ on your  │
  │          │     │            │     │ TDD, rev-│     │ WhatsApp │
  │          │     │            │     │ iewer    │     │          │
  │          │     │            │     │ validates│     │          │
  └──────────┘     └────────────┘     └──────────┘     └──────────┘
```

Multiple products run independently. ForwardCompute's team can't see Moonbeam's code, metrics, or memory. Each product is a fully isolated startup team.

---

## The Two-Tier Split

The system separates **business context** from **code context** because an AI's context window can't hold both well.

**Orchestrator** (runs every 10 min, never sees code):
- Knows your product strategy, customer conversations, metrics trends, what was tried before
- Decides WHAT to build and WHY
- Writes mission briefs for coding agents

**Coding agents** (spawned per mission, fresh each time):
- Know the codebase, test patterns, framework conventions
- Decide HOW to build it
- Write code, tests, and PRs

The orchestrator translates business signals into engineering tasks. The agents translate tasks into code. Neither does the other's job.

---

## Inside a Mission

When the orchestrator finds work (e.g., "auth TypeError, 142 Sentry events"), it schedules a mission. The mission runs in its own container with three roles:

**Architect** — Explores the codebase, maps domain boundaries (DDD), breaks the mission into 8-15 small tasks. Each task is a behavioral spec ("when X happens, the system should do Y") scoped to one domain boundary. Then dies.

**Developer** — Spawned fresh for each task. Writes a failing test (RED), writes minimum code to pass (GREEN), refactors, commits. Returns learnings. Then dies. Next task gets a fresh developer with those learnings loaded — no context accumulation.

**Reviewer** — Checks DDD compliance, test quality, and architectural integrity. Approves or requests changes. Then dies.

```
Mission container (= team lead)
│
├── Architect explores → produces task list
│
├── For each task:
│   ├── Developer: RED → GREEN → REFACTOR → commit
│   ├── Team lead saves learnings to progress.txt
│   └── Reviewer: APPROVE or CHANGE_REQUEST
│       └── If change requested: fresh developer fixes → re-review
│
└── All approved → push branch → create PR
```

The mission container coordinates the flow. All the roles are disposable — knowledge lives in files (progress.txt, AGENTS.md), not in agent memory.

---

## Knowledge Compounding

The system gets smarter over time through four layers:

| Layer | Scope | Example |
|-------|-------|---------|
| **progress.txt** | Within one mission | "This codebase uses MSW for API mocks, not jest.fn()" |
| **AGENTS.md** | Across all missions for one product | "Services always route through the domain layer" |
| **global/CLAUDE.md** | Across all products | "React inputs should use forwardRef" |
| **Skills** | Permanent, all products including future | Baked into role definitions |

Knowledge flows upward as it's proven. A pattern discovered in one mission gets tested across many, then promoted. Day 1 the system knows nothing. By Day 30, it avoids known pitfalls and writes better code.

---

## Your Daily Interaction

Three modes, all through the same WhatsApp chat per product:

**Merge/reject PRs** (~5 min/day per product)
> System: "PR #43 ready to merge: fix auth TypeError. ✓ CI ✓ 3 AI reviews."
> You: "merge" or "reject — templates should reuse existing config system"

**Set priorities** (occasional)
> You: "focus on billing this week, enterprise trials start Monday"

**Add context the system can't know** (rare)
> You: "the onboarding drop-off is because step 3 asks for a credit card"

Everything else is autonomous.

---

## Multi-Product Setup

```
WhatsApp:                    NanoClaw:
┌─────────────────┐         ┌─────────────────────────────────┐
│ Main (admin)    │────────►│ main group                      │
│ ForwardCompute  │────────►│ fc group (React/Node, 3 repos)  │
│ Moonbeam        │────────►│ moonbeam group (Python/FastAPI)  │
│ Solaris         │────────►│ solaris group (Python/FastAPI)   │
└─────────────────┘         └─────────────────────────────────┘
```

Each product group has:
- Its own CLAUDE.md (product strategy, stack, repos)
- Its own memory/ (customers, decisions, insights)
- Its own AGENTS.md (codebase patterns, mission history)
- Its own MCP servers (PostHog project, Sentry org — separate credentials)
- Its own autonomous loop (orchestrator every 10 min + missions on demand)

Adding a new product = creating a group folder, writing a CLAUDE.md, configuring MCP credentials, and registering the group. The same skills (architect, developer, reviewer) work for every product.

---

## What This Runs On

NanoClaw's existing infrastructure. No new services or dependencies:

- **Scheduler** triggers the orchestrator every 10 min per product
- **Containers** provide isolated runtime for orchestrator and mission execution
- **Skills** (markdown files) define how each role behaves
- **MCP servers** connect to PostHog and Sentry per product
- **IPC** sends notifications to your WhatsApp chat
- **Git worktrees** isolate parallel missions on separate branches

The entire autonomous system is built from skills, CLAUDE.md files, scheduled tasks, and file conventions. No changes to NanoClaw source code.
