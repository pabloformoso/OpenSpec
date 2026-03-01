# Multi-Agent Workflow Guide

OpenSpec's **distributed spec-driven** (`dispec-driven`) workflow enables parallel implementation across multiple Claude Code agents. Instead of one agent working through tasks sequentially, a team of agents works simultaneously — each with its own isolated worktree, assigned files, and task list.

## Overview

The multi-agent workflow has two phases:

1. **Planning** (`/opsx:multiagent`) — Creates all artifacts including dependency analysis and agent distribution plan
2. **Execution** (`/opsx:multiagent-apply`) — Orchestrates a Claude Code agent team from the distribution plan

```
/opsx:multiagent                    /opsx:multiagent-apply
┌─────────────────────────┐         ┌──────────────────────────┐
│ proposal.md             │         │ TeamCreate               │
│ specs/*.md              │         │ TaskCreate (all tasks)   │
│ design.md               │  ────>  │ Agent spawn (worktrees)  │
│ tasks.md                │         │ Monitor via TaskList     │
│ dependencies.md         │         │ Shutdown when done       │
│ distribution.md         │         └──────────────────────────┘
└─────────────────────────┘
```

## Quick Start

### 1. Plan the change

```
/opsx:multiagent add-user-auth
```

This creates a change using the `dispec-driven` schema and walks you through 6 artifacts:

| Artifact | Purpose |
|----------|---------|
| `proposal.md` | Why this change is needed, scoped for parallelism |
| `specs/*.md` | What the system should do (requirements + scenarios) |
| `design.md` | How to implement it, with parallelism considerations |
| `tasks.md` | 15-30 tasks with `Files:` annotations for ownership |
| `dependencies.md` | Task dependency matrix, critical path, execution waves |
| `distribution.md` | Agent assignments, file isolation, team setup instructions |

You'll be asked how many agents to use (recommended: 3-5).

### 2. Execute with a team

```
/opsx:multiagent-apply add-user-auth
```

This reads `distribution.md` and automatically:
- Creates a Claude Code team
- Populates a shared task list with dependencies
- Spawns agents in isolated worktrees
- Monitors progress and shuts down agents when done

## The `dispec-driven` Schema

The schema defines 6 artifacts in a dependency chain:

```
proposal ──> specs ──> tasks ──> dependencies ──> distribution
     └─────> design ────┘
```

- `proposal` must be created first (no dependencies)
- `specs` and `design` can be created in parallel (both depend only on `proposal`)
- `tasks` requires both `specs` and `design`
- `dependencies` requires `tasks`
- `distribution` requires `dependencies`
- The `apply` phase requires `distribution` to be complete

### Creating a change manually

If you prefer not to use the `/opsx:multiagent` skill:

```bash
openspec new change "my-change" --schema dispec-driven
openspec status --change "my-change"
```

Then create each artifact file in dependency order under `openspec/changes/my-change/`.

## Artifact Details

### proposal.md

Establishes **why** the change is needed. Unique to `dispec-driven`:

- **Capabilities** must be scoped for independent work by different agents
- Identify modules or subsystems that are naturally isolated

### specs/*.md

One spec file per capability. Each requirement uses `SHALL/MUST` language with `WHEN/THEN` scenarios.

Multi-agent addition: note the primary files and modules each capability touches — this information feeds into task distribution.

### design.md

Technical design with a dedicated **Parallelism Considerations** section:

- Which components can be built independently?
- What shared interfaces must be defined before parallel work begins?
- Serialization points where one piece must finish before another starts

### tasks.md

15-30 tasks sized for multi-agent distribution. Each task **must** include a `Files:` annotation:

```markdown
## 1. Core Implementation

- [ ] 1.1 Implement auth middleware
  <!-- Files: src/middleware/auth.ts, src/middleware/types.ts -->
- [ ] 1.2 Add JWT token validation
  <!-- Files: src/auth/jwt.ts -->
```

Tasks should be self-contained — an agent should complete one without mid-task coordination.

### dependencies.md

Analyzes task dependencies using standard project management notation:

| Type | Meaning |
|------|---------|
| **FS** (Finish-to-Start) | Task B cannot start until Task A finishes |
| **SS** (Start-to-Start) | Task B can start once Task A starts |
| **FF** (Finish-to-Finish) | Task B cannot finish until Task A finishes |
| **SF** (Start-to-Finish) | Task B cannot finish until Task A starts |

Produces:
- **Dependency Matrix** — all task relationships
- **Critical Path** — longest chain determining minimum duration
- **Parallel Execution Waves** — groups of tasks that can run simultaneously
- **Float/Slack** — how much non-critical tasks can be delayed
- **Text DAG** — ASCII visualization of the dependency graph

### distribution.md

The agent assignment plan. Contains:

- **Configuration** — agent count, tasks per agent ratio
- **Token Cost Warning** — N agents = ~N x token usage
- **Feasibility Assessment** — whether the agent count works given dependencies
- **Agent Assignment Cards** — per-agent task list, file ownership, execution order, cross-agent deps
- **File Ownership Isolation** — verifies no two agents write the same file
- **Cross-Agent Dependencies** — sync points where one agent waits for another
- **Claude Code Team Setup** — actionable instructions using `TeamCreate`, `TaskCreate`, and `Agent`

## Execution Model

### How agents coordinate

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Agent 1   │     │   Agent 2   │     │   Agent 3   │
│  (worktree) │     │  (worktree) │     │  (worktree) │
│             │     │             │     │             │
│ Tasks: 1-7  │     │ Tasks: 8-15 │     │ Tasks: 16-22│
│ Files: A, B │     │ Files: C, D │     │ Files: E, F │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────┬───────┘───────────────────┘
                   │
            ┌──────┴──────┐
            │  Shared     │
            │  Task List  │
            │  (TaskCreate│
            │   /Update)  │
            └─────────────┘
```

- Each agent runs in an **isolated git worktree** — no file conflicts
- Agents coordinate through a **shared task list** (`TaskCreate`/`TaskUpdate`/`TaskList`)
- Tasks have **`blockedBy` relationships** — agents automatically wait for dependencies
- Agents claim tasks with `TaskUpdate` (`owner`) and mark them `completed` when done
- The team lead monitors progress via `TaskList` and sends `shutdown_request` when finished

### File ownership isolation

The distribution plan enforces **one agent per file**. If two agents need to write to the same file, the plan documents the conflict and specifies ordering. The `/opsx:multiagent-apply` skill warns about violations before spawning agents.

### Token cost scaling

Multi-agent execution scales linearly:

| Agents | Estimated Cost Multiplier |
|--------|--------------------------|
| 2 | ~2x |
| 3 | ~3x |
| 4 | ~4x |
| 5 | ~5x |

Each agent maintains its own context window. The `/opsx:multiagent-apply` skill always displays a cost warning and asks for confirmation before proceeding.

## Manual Team Setup

If you prefer to set up the team manually instead of using `/opsx:multiagent-apply`:

### 1. Create the team

Use `TeamCreate` with the change name:
- `team_name`: `"my-change"` (the change name, kebab-case)
- `description`: brief description from the proposal

### 2. Populate the task list

For each task in `tasks.md`, use `TaskCreate`:
- `subject`: task description (e.g., `"1.1 Implement auth middleware"`)
- `description`: include the `Files:` annotation content
- `activeForm`: present-continuous form (e.g., `"Implementing auth middleware"`)

Then set up relationships:
- `TaskUpdate` with `addBlockedBy` to link dependent tasks
- `TaskUpdate` with `owner` to pre-assign tasks per the distribution plan

### 3. Spawn teammates

For each agent in the distribution plan, use the `Agent` tool:
- `name`: agent name from the plan (e.g., `"auth-core"`)
- `team_name`: the change name
- `subagent_type`: `"general-purpose"`
- `isolation`: `"worktree"`
- `prompt`: include assigned tasks, file ownership, execution order, and instructions to use `TaskList`/`TaskUpdate`

### 4. Monitor and shutdown

- Teammate messages are delivered automatically
- Use `TaskList` to check overall progress
- When all tasks are complete, send `shutdown_request` via `SendMessage` to each teammate
- Review and merge the worktree branches

## Comparison: Single-Agent vs Multi-Agent

| | Single-Agent (`spec-driven`) | Multi-Agent (`dispec-driven`) |
|---|---|---|
| **Schema** | `spec-driven` | `dispec-driven` |
| **Artifacts** | 4 (proposal, specs, design, tasks) | 6 (+ dependencies, distribution) |
| **Planning skill** | `/opsx:propose` | `/opsx:multiagent` |
| **Execution skill** | `/opsx:apply` | `/opsx:multiagent-apply` |
| **Parallelism** | Sequential | 2-5 agents in parallel |
| **Isolation** | Single branch | Worktree per agent |
| **Coordination** | None needed | Shared task list |
| **Token cost** | 1x | ~Nx (N = agent count) |
| **Best for** | Small-medium changes | Large changes with independent subsystems |

## When to Use Multi-Agent

Use `dispec-driven` when:
- The change touches **3+ independent modules** or subsystems
- There are **15+ tasks** that can be parallelized
- You want to **reduce wall-clock time** for large implementations
- The codebase has clear **file ownership boundaries**

Stick with `spec-driven` when:
- The change is small or tightly coupled
- Tasks are mostly sequential (long critical path)
- Token cost is a primary concern
- You're making quick fixes or focused feature additions

## Workflow Diagram

```
User                    Claude Code                 Agent Team
  │                         │                           │
  │  /opsx:multiagent       │                           │
  │────────────────────────>│                           │
  │                         │  create artifacts         │
  │                         │  (proposal → specs →      │
  │   how many agents?      │   design → tasks →        │
  │<────────────────────────│   dependencies)           │
  │   3 agents              │                           │
  │────────────────────────>│                           │
  │                         │  create distribution.md   │
  │   planning complete     │                           │
  │<────────────────────────│                           │
  │                         │                           │
  │  /opsx:multiagent-apply │                           │
  │────────────────────────>│                           │
  │                         │  read distribution.md     │
  │   cost warning (3x)     │                           │
  │<────────────────────────│                           │
  │   yes, proceed          │                           │
  │────────────────────────>│                           │
  │                         │  TeamCreate               │
  │                         │  TaskCreate (all tasks)   │
  │                         │  spawn Agent 1 ──────────>│ (worktree)
  │                         │  spawn Agent 2 ──────────>│ (worktree)
  │                         │  spawn Agent 3 ──────────>│ (worktree)
  │                         │                           │
  │                         │  monitor TaskList    <────│ task updates
  │                         │                      <────│ completions
  │                         │                           │
  │                         │  all tasks done           │
  │                         │  shutdown_request ───────>│
  │   implementation done   │                           │
  │<────────────────────────│                           │
  │                         │                           │
  │  merge branches         │                           │
  │  /opsx:archive          │                           │
```

## Troubleshooting

### "Schema 'dispec-driven' not found"

Make sure you're using OpenSpec v1.2.0 or later. The `dispec-driven` schema was added in that version.

```bash
openspec --version
```

### Agent has file conflicts

The distribution plan should prevent this via file ownership isolation. If conflicts occur:
1. Check `distribution.md` for the File Ownership Isolation table
2. Reassign tasks so each file has a single owner
3. Re-run `/opsx:multiagent-apply`

### Cross-agent deadlock

If agents are waiting on each other in a cycle:
1. Check `dependencies.md` for the dependency matrix
2. The dependency analysis should have caught cycles — if it didn't, update the dependency matrix
3. Break the cycle by having one agent complete its task first (change the ordering)

### Token costs too high

- Reduce the agent count (2 agents is the minimum)
- Ensure the distribution plan groups dependent tasks into the same agent to minimize cross-agent sync points
- Consider using `/opsx:apply` (single-agent) for smaller changes
