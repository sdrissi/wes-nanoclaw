# Agent Team Architecture — Implementation Blueprint

Reference for implementing autonomous product teams on NanoClaw. For a concise overview, see [docs/AGENT_TEAMS.md](../AGENT_TEAMS.md).

---

## Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Autonomy model | Fully autonomous 24/7 | System discovers work from data signals, never waits for human input |
| Parallelism | NanoClaw scheduler + containers | No Agent Teams needed — scheduler spawns parallel containers per group |
| Agent nesting | 1 level only (container → Task agents) | Proven, reliable, no cascading failure from deep nesting |
| Team structure | Team lead (container session) + 3 disposable roles | Architect, developer, reviewer — all fresh per invocation |
| Developer freshness | Disposable per task, not per mission | Prevents context exhaustion; learnings loaded via files, not accumulated |
| TDD approach | RED/GREEN/REFACTOR per task | Developer writes test first, implements minimum to pass, refactors |
| Task breakdown | DDD-aware behavioral specs | Architect maps domain boundaries, scopes tasks to bounded contexts |
| Knowledge file | Single AGENTS.md with sections | No fragmentation; readers skip sections they don't need |
| Container model | One group per product, multiple container types | Orchestrator + mission + message containers all share one group |
| Separation enforcement | Skill instructions, not mount differences | Simpler infrastructure; orchestrator skill says "don't read code" |
| Human role | Merge gate + occasional strategic direction | Three modes: merge/reject PRs, set priorities, inject context |

---

## The Two-Tier Context Split

Context windows are zero-sum. Fill with business context → no room for code. Fill with code → no room for business context.

```
ORCHESTRATOR (per product container)        CODING AGENTS (disposable Task agents)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Knows:                                      Knows:
├── Product strategy & roadmap              ├── The codebase (one repo)
├── Customer conversations                  ├── AGENTS.md codebase patterns section
├── PostHog metrics & trends                ├── progress.txt (this mission's learnings)
├── Sentry error patterns                   ├── The specific behavioral spec
├── What was tried before (mission history) └── Test suite & framework docs
├── What shipped & what failed
├── Which prompt patterns work              Doesn't know:
└── Current priorities                      ├── Why this task matters
                                            ├── Customer context
Doesn't know:                               ├── Business priorities
├── Implementation details                  └── Other active missions
├── Codebase internals
└── How to write code

Translates WHY into mission briefs          Translates briefs into code
```

---

## Container Types Per Group

One NanoClaw group per product. One WhatsApp chat per product. Three container types, all sharing the same group folder, MCP servers, and chat.

```
GROUP: fc
CHAT: ForwardCompute (WhatsApp/Telegram)
FOLDER: groups/fc/

┌────────────────────┬──────────────────────────┬──────────────────────┐
│  Container Type    │  Trigger                 │  What It Does        │
├────────────────────┼──────────────────────────┼──────────────────────┤
│  Message handler   │  You text the fc chat    │  Regular NanoClaw    │
│                    │                          │  flow: responds to   │
│                    │                          │  your message.       │
│                    │                          │  Handles: merge PRs, │
│                    │                          │  update priorities,  │
│                    │                          │  save context.       │
├────────────────────┼──────────────────────────┼──────────────────────┤
│  Orchestrator      │  Scheduled: every 10 min │  Invoke /orchestrator│
│                    │                          │  skill. Scan signals,│
│                    │                          │  prioritize, schedule│
│                    │                          │  missions, monitor   │
│                    │                          │  outcomes, notify.   │
│                    │                          │  Never reads code.   │
├────────────────────┼──────────────────────────┼──────────────────────┤
│  Mission           │  On demand (scheduled    │  Invoke /mission-lead│
│                    │  by orchestrator)        │  skill. IS the team  │
│                    │                          │  lead. Spawns Task   │
│                    │                          │  agents: architect,  │
│                    │                          │  developer, reviewer.│
│                    │                          │  Creates PR.         │
└────────────────────┴──────────────────────────┴──────────────────────┘

All three container types:
├── Read/write groups/fc/ (CLAUDE.md, memory/, autonomous/)
├── Use fc's MCP servers (PostHog fc project, Sentry fc org)
├── Send messages to the fc WhatsApp chat
└── Mount fc's code repos

Separation between orchestrator and mission is enforced by skill
instructions, not by mount configuration.
```

---

## Work Discovery Loop (Orchestrator Container)

Runs every 10 minutes per product. Light token usage — scans signals, reads JSON, schedules tasks.

```
Orchestrator container starts (scheduled, every 10 min)
│
├── Read CLAUDE.md → current priorities, product context
├── Read AGENTS.md § Mission History → what was tried, what failed
├── Read AGENTS.md § Prompt Patterns → how to write effective briefs
├── Read active-tasks.json → what's currently running
├── Read memory/ → recent customer feedback, decisions
│
├── Scan data signals:
│   ├── PostHog API → metric drops, funnel breaks, unused features
│   ├── Sentry API → new/trending errors, spikes, regressions
│   ├── memory/customers/ → unaddressed feedback, feature requests
│   └── git log (via IPC) → merged PRs needing docs/changelog
│
├── Prioritize:
│   ├── Rank by impact (metric severity × user count)
│   ├── Weight by current priorities from CLAUDE.md
│   ├── Filter: already in progress (active-tasks.json)
│   ├── Filter: already failed recently (AGENTS.md § Mission History)
│   └── Filter: beyond agent capability → flag for human
│
├── Monitor existing missions:
│   ├── Check PR status via IPC (gh pr checks)
│   ├── Timed out? → mark failed, log to AGENTS.md
│   ├── PR passed all checks? → update status, notify human via chat
│   ├── PR failed CI? → schedule retry with failure context
│   └── Container crashed? → schedule retry
│
├── Schedule new missions (up to available slots):
│   ├── Write mission brief to runs/{id}/mission.md
│   ├── Register in active-tasks.json (status: "scheduled")
│   └── schedule_task(prompt, group) via IPC
│
├── Post-merge curation:
│   ├── For missions that were merged since last scan:
│   ├── Read the mission's progress.txt
│   ├── Curate proven patterns into AGENTS.md § Codebase Patterns
│   └── Log success to AGENTS.md § Mission History
│
└── Container exits.
```

---

## Mission Execution (Mission Container)

Each mission container IS the team lead. It spawns Task agents directly — one level of nesting only.

```
Mission container starts (scheduled by orchestrator)
│
├── Read runs/{id}/mission.md (the brief)
├── Read AGENTS.md (all sections)
├── Read CLAUDE.md (product context)
│
├── Set up worktree:
│   git worktree add ../fix-auth-error -b fix/auth-error origin/main
│
│
│  STEP 1: ARCHITECT
│  ─────────────────
├── Task(subagent_type: "Explore",
│     prompt: "Mission: [brief]. Explore codebase. Map domain
│     boundaries (DDD). Break into testable behavioral specs.
│     Known patterns: [AGENTS.md § Codebase Patterns]")
│
│   Architect returns:
│   ├── domain-mapping (bounded contexts, aggregates, layers)
│   └── ordered task list (each task = one behavioral spec,
│       scoped to one bounded context)
│
│
│  STEP 2: FOR EACH TASK → DEVELOPER + REVIEWER
│  ─────────────────────────────────────────────
│
│   ┌─── Task N ─────────────────────────────────────────────┐
│   │                                                        │
│   │  Developer:                                            │
│   │  Task(subagent_type: "general-purpose",                │
│   │    prompt: "Invoke /dev. TDD RED/GREEN.                │
│   │    Task: [one behavioral spec from architect].         │
│   │    Learnings: [progress.txt contents].                 │
│   │    Patterns: [AGENTS.md § Codebase Patterns].")        │
│   │                                                        │
│   │  Developer does:                                       │
│   │  RED → write failing test for the behavioral spec      │
│   │  GREEN → minimum code to pass                          │
│   │  REFACTOR → clean up, run full suite, commit           │
│   │                                                        │
│   │  Returns: { pass, summary, learnings }                 │
│   │                                                        │
│   │  Team lead appends learnings to progress.txt           │
│   │                                                        │
│   │  Reviewer:                                             │
│   │  Task(subagent_type: "general-purpose",                │
│   │    prompt: "Review diff. Check:                        │
│   │    1. DDD: bounded context boundaries, dependency      │
│   │       direction, aggregate scope, naming.              │
│   │    2. Test quality: tests behavior not implementation, │
│   │       would catch regression, is independent.          │
│   │    3. Architecture: follows existing patterns,         │
│   │       deps flow inward, change is minimal.             │
│   │    Domain map: [from architect].                       │
│   │    Patterns: [AGENTS.md § Codebase Patterns].")        │
│   │                                                        │
│   │  Returns: APPROVE or CHANGE_REQUEST + feedback         │
│   │                                                        │
│   │  If CHANGE_REQUEST:                                    │
│   │    Team lead appends feedback to progress.txt          │
│   │    Spawn fresh developer with feedback in prompt       │
│   │    Spawn fresh reviewer to re-check                    │
│   │                                                        │
│   │  If APPROVE: move to next task                         │
│   └────────────────────────────────────────────────────────┘
│
│   Repeat for each task in the architect's list.
│
│
│  STEP 3: PR
│  ──────────
├── Push branch
├── gh pr create (title + body with rationale, task checklist)
├── Update active-tasks.json via IPC: status → "pr_created"
│
└── Container exits.

Next orchestrator cycle detects the PR, checks CI + AI reviews,
and notifies you when all checks pass.
```

---

## Developer Agent: Fresh Context Per Task

Each developer agent starts clean. No accumulated diffs from previous tasks.

```
Developer agent prompt (assembled by team lead for Task N):

┌────────────────────────────────────────────────────────────┐
│ 1. /dev skill (TDD process, commit conventions)            │  Always same
│ 2. AGENTS.md § Codebase Patterns                           │  Cross-run
│ 3. progress.txt (learnings from Tasks 1..N-1 this mission) │  This mission
│ 4. ONE behavioral spec from architect                      │  This task
└────────────────────────────────────────────────────────────┘

No Task 1 diffs. No Task 2 diffs. No stale implementation context.
Full context budget available for the codebase.
Agent dies after returning result. Context freed.
```

---

## Knowledge System

Single file (AGENTS.md) with sections. One file per product, persistent across missions.

### AGENTS.md Structure

```markdown
# AGENTS.md

## Mission History
<!-- Orchestrator reads: avoid repeats, learn what works -->

### Merged
- 2026-02-27: fix-auth-error (PR #43) — 3 tasks, 1 retry on task 2
  Brief style: specific Sentry event + acceptance criteria
- 2026-02-27: fix-search-perf (PR #42) — 3 tasks, 0 retries
  Brief style: PostHog metric + user complaint quotes

### Failed
- 2026-02-25: migrate-design-system — too large for agent scope
  Reason: requires cross-repo coordination + breaking changes
  Action: flagged for human

## Codebase Patterns
<!-- Architect + developer read: write better code -->

- All service access routes through domain layer. Never import
  infrastructure directly from components. (Learned: 2026-02-27)
- Input components use forwardRef pattern. (Learned: 2026-02-27)
- Use MSW for API mocks, not jest.fn() on fetch. (Learned: 2026-02-27)
- Stripe SDK v12. Use TEST_STRIPE_KEY env var. (Learned: 2026-02-26)
- Error types live in domain/errors.ts. (Learned: 2026-02-27)

## Prompt Patterns
<!-- Orchestrator reads: write better mission briefs -->

- Search domain tasks: mention SearchService.ts is the entry point
- Billing tasks: mention Stripe SDK version + test key
- Auth tasks: mention TokenRepository uses dependency injection
- General: include acceptance criteria as measurable thresholds,
  not descriptions ("< 200ms" not "should be fast")
```

### Knowledge Flow

```
WHEN                           WHO WRITES          WHERE               WHO READS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

After each task completes       Team lead           progress.txt        Next developer
(within a mission)              (extracts from                          agent in this
                                 dev agent result)                      mission

After reviewer feedback         Team lead           progress.txt        Next developer
(within a mission)                                                      agent + next
                                                                        reviewer

After PR merges                 Orchestrator        AGENTS.md           All future
(across missions)               (curates from       (all 3 sections)    missions for
                                 progress.txt)                          this product

Periodic promotion              You (or scheduled   global/CLAUDE.md    All products
(across products)               task)               then skills/

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

progress.txt ──► AGENTS.md ──► global/CLAUDE.md ──► skills/
(within mission)  (across missions)  (across products)   (permanent)
```

---

## File Structure

```
groups/fc/
├── CLAUDE.md                              Product context, strategy, priorities
├── memory/
│   ├── posthog-insights/                  Metrics trends, funnel analysis
│   ├── sentry-insights/                   Error patterns, regressions
│   ├── customers/                         Customer conversations, requests
│   ├── patterns/                          What worked in dev, marketing
│   └── decisions/                         Architectural decisions made
│
└── autonomous/
    ├── AGENTS.md                          Cross-run knowledge (3 sections)
    ├── active-tasks.json                  Currently running/completed missions
    └── runs/
        ├── fix-auth-error/
        │   ├── mission.md                 Brief from orchestrator
        │   ├── progress.txt               Learnings within this mission
        │   └── result.json                Outcome, timing, PR number
        ├── fix-search-perf/
        └── (archived after merge + curation)
```

---

## Human Interaction: Three Modes

All through the same WhatsApp chat per product.

**Mode 1 — Merge gate (daily, ~5 min per product)**
System: "PR #43 ready to merge: fix auth TypeError. ✓ CI ✓ 3 AI reviews."
You: "merge" or "reject — [feedback]"
Rejection feedback gets saved, orchestrator retries with updated brief.

**Mode 2 — Strategic direction (occasional)**
You: "focus on billing this week" / "ignore CSS warnings"
Updates CLAUDE.md or memory/. Orchestrator adjusts prioritization on next cycle.

**Mode 3 — Context injection (rare)**
You: "the onboarding drop-off is because step 3 requires a credit card"
Business context the system can't derive from metrics. Saved to memory/, used in future mission briefs.

---

## What to Build

| Item | Type | Purpose |
|------|------|---------|
| `container/skills/orchestrator/SKILL.md` | Skill | Scan signals, prioritize, schedule missions, monitor, curate AGENTS.md. Never read code. |
| `container/skills/mission-lead/SKILL.md` | Skill | Run architect → developer → reviewer loop. Manage progress.txt. Create PR. |
| `container/skills/dev/SKILL.md` | Skill | TDD RED/GREEN/REFACTOR. Receive one behavioral spec, implement, return learnings. |
| `container/skills/architect/SKILL.md` | Skill | Explore codebase, map DDD domain boundaries, produce testable behavioral specs. |
| `container/skills/reviewer/SKILL.md` | Skill | Check DDD compliance, test quality, architectural integrity. APPROVE or CHANGE_REQUEST. |
| `container/skills/analyst/SKILL.md` | Skill | Query PostHog/Sentry, format insights for orchestrator consumption. |
| `groups/{product}/CLAUDE.md` | Product context | Strategy, stack, repos, customers, current priorities. |
| `groups/{product}/autonomous/AGENTS.md` | Knowledge | Mission history + codebase patterns + prompt patterns. |
| `groups/{product}/autonomous/active-tasks.json` | State | Running/completed mission registry. |
| Scheduler configuration | Config | 10-min orchestrator task per product group. |
| Per-product MCP config | Config | `data/sessions/{product}/.claude/settings.json` with PostHog + Sentry credentials. |

No changes to NanoClaw source code for the core architecture. Skills (markdown), CLAUDE.md (markdown), memory (directories + files), scheduled tasks (existing scheduler), MCP config (existing settings.json).
