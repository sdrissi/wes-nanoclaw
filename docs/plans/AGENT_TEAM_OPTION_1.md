# Agent Team Architecture Blueprint

A pattern for running multiple AI startup teams on NanoClaw, where each product gets a dedicated central intelligence that coordinates specialized agents (dev, marketing, product analytics, etc.).

---

## Core Idea

One NanoClaw group per product. Each group is an autonomous startup team. You talk to the product's chat, and a central intelligence (orchestrator) delegates work to specialized agents using Claude Code's agent swarm (TeamCreate/SendMessage/Task).

Reusable role definitions live in `container/skills/` and are shared across all products. Product-specific context lives in each group's `CLAUDE.md` and `memory/`. Adding a new capability (e.g., marketing) = adding a skill file. Adding a new product = registering a new group.

Zero changes to NanoClaw source code. Only markdown skills, CLAUDE.md files, and JSON config.

---

## Alignment with NanoClaw Principles

| Principle | How this architecture honors it |
|-----------|-------------------------------|
| Small enough to understand | No code changes. Only adds skills (markdown) and group configs |
| Secure by isolation | Each product team runs in its own container, can only see its own repos and memory |
| Customization = code changes | Product context lives in CLAUDE.md, not config files |
| Skills over features | Roles (dev, marketer, analyst) are skills, not baked-in features |
| Built for one user | One chat per product, natural conversation, main for admin |

---

## Architecture Overview

### Three Layers

```
LAYER 1: Reusable Role Skills (shared across all products)
  container/skills/
  ├── dev/SKILL.md         HOW to implement code (TDD, commits)
  ├── planner/SKILL.md     HOW to break work into tasks
  ├── analyst/SKILL.md     HOW to analyze PostHog/Sentry/metrics
  ├── marketer/SKILL.md    HOW to create content and do outreach
  ├── reviewer/SKILL.md    HOW to review code quality
  └── agent-browser/       HOW to browse the web (already exists)

  Synced to ALL containers automatically (container-runner.ts:125-135)

LAYER 2: Product Context (per product)
  groups/fc/CLAUDE.md      WHAT ForwardCompute is, its stack, audience, team config
  groups/moonbeam/CLAUDE.md WHAT Moonbeam is, different stack, same role skills

LAYER 3: Accumulated Knowledge (per product)
  groups/fc/memory/
  ├── posthog-insights/    What analytics revealed
  ├── sentry-insights/     What errors occurred
  ├── marketing/           What was posted, engagement results
  ├── patterns/            What worked and failed in dev
  └── decisions/           Architectural decisions made
```

### System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           YOUR PHONE                                │
│                                                                     │
│   ┌─────────┐    ┌──────────┐    ┌──────────┐                     │
│   │  main   │    │    fc    │    │ moonbeam │    ...               │
│   │  chat   │    │   chat   │    │   chat   │                      │
│   │ (admin) │    │ (daily)  │    │ (daily)  │                      │
│   └────┬────┘    └────┬─────┘    └────┬─────┘                     │
└────────┼──────────────┼───────────────┼────────────────────────────┘
         │              │               │
         │              │               │
┌────────┼──────────────┼───────────────┼────────────────────────────┐
│        ▼              ▼               ▼        NanoClaw Host       │
│                                                                    │
│   ┌────────────────────────────────────────────┐                   │
│   │           Node.js Orchestrator              │                   │
│   │                                             │                   │
│   │  index.ts    — message loop (2s poll)       │                   │
│   │  ipc.ts      — routes IPC commands          │                   │
│   │  scheduler   — runs cron tasks              │                   │
│   │  group-queue — concurrency control          │                   │
│   │                                             │                   │
│   │          ┌──────────────┐                   │                   │
│   │          │   SQLite DB  │                   │                   │
│   │          │  messages    │                   │                   │
│   │          │  sessions    │                   │                   │
│   │          │  tasks       │                   │                   │
│   │          │  groups      │                   │                   │
│   │          └──────────────┘                   │                   │
│   └──────┬─────────────┬──────────────┬────────┘                   │
│          │             │              │                             │
│          ▼             ▼              ▼                             │
│   ┌───────────┐ ┌───────────┐ ┌───────────┐                       │
│   │ Container │ │ Container │ │ Container │  (up to 5 concurrent)  │
│   │   main    │ │    fc     │ │  moonbeam │                        │
│   │  (admin)  │ │  (team)   │ │  (team)   │                        │
│   └───────────┘ └───────────┘ └───────────┘                        │
│                                                                    │
│   Shared assets (synced at container launch):                      │
│   ├── container/skills/*  → .claude/skills/ in every container     │
│   └── groups/global/      → /workspace/global/ in every container  │
└────────────────────────────────────────────────────────────────────┘
```

### Inside a Product Container

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Container: nanoclaw-fc-*                          │
│                                                                      │
│  Filesystem:                                                         │
│  /workspace/group            ← groups/fc/ (rw, memory + conversations)│
│  /workspace/global           ← groups/global/ (ro, shared persona)   │
│  /workspace/extra/fc-frontend← ~/Repos/fc-frontend (rw)             │
│  /workspace/extra/fc-backend ← ~/Repos/fc-backend (rw)              │
│  /home/node/.claude          ← sessions + synced skills + settings   │
│  /workspace/ipc              ← IPC namespace for fc only             │
│                                                                      │
│  Skills loaded: /dev /planner /analyst /marketer /reviewer           │
│  MCP servers: nanoclaw (IPC) + PostHog + Sentry (from settings.json)│
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    ORCHESTRATOR (team lead)                     │  │
│  │                                                                │  │
│  │  Reads CLAUDE.md → product context, stack, audience, team      │  │
│  │  Reads memory/   → past insights, patterns, decisions          │  │
│  │  Reads global/   → shared persona, cross-project learnings     │  │
│  │                                                                │  │
│  │  Classifies request → delegates to appropriate role            │  │
│  │                                                                │  │
│  │              Persistent Team (TeamCreate)                       │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │  │
│  │  │ researcher │ │  planner   │ │   tester   │ │  analyst   │ │  │
│  │  │  (haiku)   │ │  (sonnet)  │ │  (haiku)   │ │  (sonnet)  │ │  │
│  │  │            │ │            │ │            │ │            │ │  │
│  │  │ Finds code │ │ Designs    │ │ Validates  │ │ PostHog    │ │  │
│  │  │ Reads docs │ │ plans      │ │ tests      │ │ Sentry     │ │  │
│  │  │ Explores   │ │ Breaks     │ │ Checks     │ │ Metrics    │ │  │
│  │  │ context    │ │ into tasks │ │ quality    │ │ Funnels    │ │  │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘ │  │
│  │                                                                │  │
│  │              Disposable Agents (Task tool)                      │  │
│  │  ┌────────────┐ ┌────────────┐                                 │  │
│  │  │implementer │ │  marketer  │  Spawned per task.              │  │
│  │  │  (sonnet)  │ │  (sonnet)  │  Invoke role skill.             │  │
│  │  │ /dev skill │ │ /marketer  │  Die when done.                 │  │
│  │  │ TDD cycle  │ │ skill      │  Fresh context each time.       │  │
│  │  │ Commits    │ │ Drafts,    │                                 │  │
│  │  │            │ │ posts      │                                 │  │
│  │  └────────────┘ └────────────┘                                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  MCP Server: nanoclaw (always present)                         │  │
│  │  send_message  → IPC → host → WhatsApp/Telegram               │  │
│  │  schedule_task → IPC → host → SQLite                          │  │
│  │  list_tasks, pause_task, resume_task, cancel_task              │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Isolation Model

```
                    main container              fc container            moonbeam container

Project root        /workspace/project (RO)     not mounted             not mounted
Own group folder    /workspace/group (RW)       /workspace/group (RW)   /workspace/group (RW)
Global memory       (under project mount)       /workspace/global (RO)  /workspace/global (RO)
FC repos            not mounted                 /workspace/extra/* (RW) not mounted
Moonbeam repo       not mounted                 not mounted             /workspace/extra/* (RW)
IPC scope           all groups                  own chat only           own chat only
Task scope          all tasks                   own tasks only          own tasks only
Database            read-only                   not mounted             not mounted
MCP servers         nanoclaw only               nanoclaw + PostHog +    nanoclaw + PostHog
                                                Sentry (FC creds)       (Moonbeam creds)
```

---

## Permission Hierarchy

```
YOU (Claude Code on host)
│  Full control over:
│  ├── container/skills/*         role definitions
│  ├── groups/*/CLAUDE.md         product context
│  ├── registered_groups.json     group config, mounts
│  ├── mount-allowlist.json       security policy
│  ├── groups/global/CLAUDE.md    cross-project learnings
│  ├── data/sessions/*/settings   per-group MCP servers
│  └── src/*                      NanoClaw source code
│
├── MAIN AGENT
│   │  Can read:   everything (project root, read-only)
│   │  Can write:  groups/main/ only
│   │  Via IPC:    register groups, send to any chat,
│   │              schedule/manage tasks for any group
│   │  Cannot:     modify skills, mounts, source, global, settings
│   │
│   └── PRODUCT AGENTS (fc, moonbeam, ...)
│       Can read:   own group folder + global + mounted repos
│       Can write:  own group folder + mounted repos
│       Via IPC:    send to own chat, schedule tasks for self
│       Cannot:     see other groups, database, project root,
│                   modify skills, send to other chats
```

---

## Groups and Their Roles

### main — Portfolio Admin (rarely used)

You message main for:
- Cross-product queries: "how are all my products doing?"
- Creating new products: "@Wes create a new product called Solaris"
- Cross-product task management: "list all tasks across products"

Main can register new groups via IPC but cannot write their CLAUDE.md, set up mounts, or configure MCP servers (those are host-level actions via Claude Code).

### fc — ForwardCompute Team (daily use)

You message fc for all ForwardCompute work:
- "fix the autocomplete bug"
- "what did PostHog show this week?"
- "post about the new feature on Reddit"
- "check Sentry for new errors"

The orchestrator classifies your request, gathers context from memory, and delegates to the right subagents.

### moonbeam — Another Product (daily use)

Same structure, different product context, different repos mounted, different MCP credentials. Completely isolated from fc.

---

## Two Types of Subagents

### Persistent (TeamCreate + SendMessage)

Created once per session. Stay alive, accumulate context. Good for roles that need ongoing awareness.

| Role | Agent Type | Model | Purpose |
|------|-----------|-------|---------|
| researcher | Explore | haiku | Find code, read docs, build context |
| planner | Plan | sonnet | Design approaches, create task breakdowns |
| tester | general-purpose | haiku | Run tests, validate builds, check quality |
| analyst | general-purpose | sonnet | Query PostHog/Sentry, analyze metrics |

Coordinated via `SendMessage`:
```
SendMessage(recipient: "researcher", content: "Find the autocomplete code...")
```

### Disposable (Task tool)

Spawned fresh per task. Die when done. Fresh context prevents stale assumptions.

| Role | Model | When to use |
|------|-------|-------------|
| implementer | sonnet | Each coding task in a plan. Invokes /dev skill. |
| marketer | sonnet | Content creation, outreach. Invokes /marketer skill. |
| other one-offs | varies | Any task that benefits from a clean slate |

Spawned via `Task`:
```
Task(subagent_type: "general-purpose", prompt: "Invoke /dev skill. Implement: [plan]...")
```

---

## Per-Group MCP Servers

Each group gets its own MCP server configuration via `data/sessions/{group}/.claude/settings.json`. No code changes needed.

Example for fc (`data/sessions/fc/.claude/settings.json`):
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD": "1",
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  },
  "mcpServers": {
    "posthog": {
      "command": "npx",
      "args": ["-y", "@anthropic/posthog-mcp-server"],
      "env": {
        "POSTHOG_API_KEY": "phx_fc_key",
        "POSTHOG_PROJECT_ID": "12345"
      }
    },
    "sentry": {
      "command": "npx",
      "args": ["-y", "@sentry/mcp-server"],
      "env": {
        "SENTRY_AUTH_TOKEN": "sntrys_fc_token",
        "SENTRY_ORG": "forwardcompute"
      }
    }
  }
}
```

Moonbeam gets different credentials pointing to its own PostHog project. Each product team queries only its own data.

The `nanoclaw` MCP server (IPC tools: send_message, schedule_task, etc.) is always added programmatically by the agent-runner and doesn't need to be in settings.json.

---

## Multiple Repos Per Product

Products with multiple repositories use `additionalMounts` in `registered_groups.json`:

```json
{
  "12036...@g.us": {
    "name": "ForwardCompute",
    "folder": "fc",
    "trigger": "@Wes",
    "containerConfig": {
      "additionalMounts": [
        { "hostPath": "~/Repos/fc-frontend",  "containerPath": "fc-frontend",  "readonly": false },
        { "hostPath": "~/Repos/fc-backend",   "containerPath": "fc-backend",   "readonly": false },
        { "hostPath": "~/Repos/fc-infra",     "containerPath": "fc-infra",     "readonly": true  }
      ]
    }
  }
}
```

The agent sees:
```
/workspace/extra/
├── fc-frontend/     (rw)
├── fc-backend/      (rw)
└── fc-infra/        (ro)
```

CLAUDE.md files in each repo are loaded automatically via `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD`. The product's CLAUDE.md should include a repo map so the orchestrator knows what's where.

All mounts are validated against `~/.config/nanoclaw/mount-allowlist.json` (stored outside the project, tamper-proof from containers). Sensitive paths (.ssh, .aws, .env, etc.) are blocked by default.

---

## Cross-Project Learnings

When an agent discovers something generalizable (e.g., "benchmark posts get 3x Reddit engagement"), it tags the insight with `[GENERALIZABLE]` in the product's memory.

Periodically, you (via Claude Code on the host) or a main scheduled task can:
1. Read all products' memory for `[GENERALIZABLE]` tags
2. Synthesize into `groups/global/CLAUDE.md` or a cross-project learnings file
3. All product groups read global on every invocation — learnings propagate

When an insight is proven reliable across multiple products, codify it into the skill itself (`container/skills/marketer/SKILL.md`). Now every product, including future ones, starts with that knowledge.

Propagation speeds:
- **Product memory**: instant, only that product
- **Global memory**: next invocation, all products
- **Skill update**: permanent, all current and future products

---

## Scheduled Tasks

Each product group can have autonomous scheduled work:

```
Daily 9am:    Analyst checks PostHog → saves to memory/posthog-insights/
Hourly:       Check Sentry for new errors → saves to memory/sentry-insights/
Weekly Mon:   Full review — metrics + errors + marketing → priorities report
Daily 6pm:    Archive conversations, extract decisions
```

Scheduled from the product's chat:
```
fc chat: "@Wes check PostHog every morning at 9am and report key metrics"
→ schedule_task(prompt: "Invoke /analyst skill...", cron: "0 9 * * *")
```

Or from main for cross-product scheduling:
```
main: "@Wes schedule a weekly report for ForwardCompute every Monday"
→ schedule_task(..., target_group_jid: "12036...@g.us")
```

---

## Typical Message Flow

```
You (fc chat): "autocomplete is slow, users are dropping off"
  │
  ▼
WhatsApp/Telegram → NanoClaw host → SQLite
  │
  ▼
Poll loop (2s) → message for fc group, trigger matched
  │
  ▼
Spawn container nanoclaw-fc-* with fc's mounts and MCP servers
  │
  ▼
Orchestrator reads:
  ├── CLAUDE.md (product context, team config)
  ├── memory/patterns/ (what worked before)
  ├── memory/posthog-insights/ (recent analytics)
  └── global/CLAUDE.md (persona, cross-project learnings)
  │
  ▼
Classifies: engineering task → research → plan → implement → test
  │
  ├─► SendMessage → researcher: "find autocomplete code"
  │   researcher explores /workspace/extra/fc-frontend
  │   reports: "bottleneck in AutocompleteSearch.tsx:47, no debounce"
  │
  ├─► SendMessage → planner: "plan fix based on findings"
  │   planner: "1. add 200ms debounce 2. cache results 3. prefetch top items"
  │
  ├─► Task → implementer (disposable):
  │   "invoke /dev skill, implement plan, repo at /workspace/extra/fc-frontend"
  │   implements, tests, commits, dies
  │
  ├─► SendMessage → tester: "validate fix, run full suite"
  │   tester: "47 tests pass, build clean"
  │
  ├─► Saves to memory/patterns/success.json
  │
  └─► send_message (IPC): "Fixed autocomplete (8s → 200ms).
      Added debounce + caching. 3 commits, all tests pass."
  │
  ▼
IPC watcher → WhatsApp/Telegram → your phone
```

---

## Adding a New Capability

Example: adding marketing to an existing product.

**Step 1: Create the skill (one-time, shared)**
```
container/skills/marketer/SKILL.md
```
Define how to draft content, track engagement, platform guidelines, memory conventions.

**Step 2: Mention it in the product's CLAUDE.md**
```markdown
## Team
- /marketer — content, outreach, engagement tracking
```

**Step 3: Use it**
```
fc chat: "Post about the new feature on Reddit"
→ Orchestrator spawns marketer subagent
→ Subagent invokes /marketer skill
→ Reads product context from CLAUDE.md (audience, tone)
→ Reads memory/marketing/reddit/ for past posts
→ Drafts, presents for review or posts
→ Logs to memory/marketing/reddit/posts.md
```

---

## Adding a New Product

**From Claude Code on the host (recommended):**
```
You: "Set up a new product called Moonbeam.
     Stack: Python, FastAPI.
     Repo: ~/Repos/moonbeam"
```

Claude Code:
1. Registers group in `registered_groups.json` with `additionalMounts`
2. Updates `~/.config/nanoclaw/mount-allowlist.json` if needed
3. Creates `groups/moonbeam/CLAUDE.md` with product context
4. Creates `groups/moonbeam/memory/` directory structure
5. Writes `data/sessions/moonbeam/.claude/settings.json` with MCP servers
6. Done — message the moonbeam chat to start working

**From main chat (partial):**
Main can register the group via IPC but cannot write CLAUDE.md, set up mounts, or configure MCP servers. Host-level setup is still needed.

---

## When to Talk to Main vs a Product Chat

| Through main (delegation/admin) | Direct to product chat (daily work) |
|--------------------------------|-------------------------------------|
| "Create a new product" | "Fix the autocomplete bug" |
| "List all tasks across products" | "What did PostHog show this week?" |
| "How are all my products doing?" | "Post about the fix on Reddit" |
| "Schedule a weekly report for fc" | "Check Sentry for new errors" |
| | "Walk me through the auth code" |
| | "Let's brainstorm the next feature" |

Main is for cross-product admin. Product chats are for daily work. You'll mostly use product chats.

---

## What to Build

| Item | Type | Purpose |
|------|------|---------|
| `container/skills/dev/SKILL.md` | Skill | TDD process, commit conventions, failure/success logging |
| `container/skills/planner/SKILL.md` | Skill | Task breakdown, plan structure, dependency ordering |
| `container/skills/analyst/SKILL.md` | Skill | PostHog/Sentry analysis, insight formatting, memory conventions |
| `container/skills/marketer/SKILL.md` | Skill | Content creation, platform guidelines, engagement tracking |
| `container/skills/reviewer/SKILL.md` | Skill | Code review process, spec compliance, quality checks |
| `groups/fc/CLAUDE.md` | Product context | FC product description, repo map, team config, memory layout |
| `groups/fc/memory/` | Directories | posthog-insights/, sentry-insights/, marketing/, patterns/, decisions/ |
| `data/sessions/fc/.claude/settings.json` | Config | PostHog + Sentry MCP servers with FC credentials |
| `registered_groups.json` entry | Config | FC group with additionalMounts for repos |
| `mount-allowlist.json` update | Config | Allow ~/Repos/ as mount root |

No changes to NanoClaw source code. No new dependencies.

---

## Key Design Decisions

1. **One group per product, not per function.** Avoids the "always specify which product" problem. The orchestrator inside each product handles all functions (dev, marketing, analytics) via agent swarm.

2. **Skills for roles, CLAUDE.md for context.** Role definitions (HOW) are shared. Product context (WHAT) is per-group. This means adding a new product reuses all existing roles automatically.

3. **Persistent team for context, disposable agents for execution.** Researcher/planner/tester stay alive to accumulate context. Implementers/marketers are spawned fresh to avoid stale assumptions.

4. **Memory stays in NanoClaw group folders.** Simple, works today. Can be moved to product repos later if version control becomes important. Don't over-engineer now.

5. **MCP servers via settings.json, not code changes.** Per-group MCP configuration using existing Claude Code settings mechanism. Each product gets its own credentials.

6. **Host-level control for structural changes.** Skills, mounts, MCP servers, and security config are managed via Claude Code on the host, not by agents. Agents coordinate and execute, they don't reconfigure themselves.
