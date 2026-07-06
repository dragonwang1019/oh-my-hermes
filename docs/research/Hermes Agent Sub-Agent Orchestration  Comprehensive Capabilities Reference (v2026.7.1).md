# Hermes Agent Sub-Agent Orchestration: Comprehensive Capabilities Reference (v2026.7.1)

**Release:** v0.18.0 — "The Judgment Release" (July 1, 2026)[^1]
**Repository:** github.com/NousResearch/hermes-agent  
**Scope:** Sub-agent dispatch, orchestration topology, third-party coding agent integration, ACP compatibility, and the Kanban multi-agent platform

***

## Executive Summary

Hermes Agent (by Nous Research) functions as a fully capable outer-loop orchestrator that can dispatch, supervise, and synthesize results from multiple classes of sub-agents. As of v2026.7.1, the system supports three complementary sub-agent patterns: (1) **synchronous `delegate_task` fan-out** for in-turn parallel workstreams, (2) **background `delegate_task` fan-out** introduced in the v0.18.0 window for non-blocking async delegation, and (3) the **Kanban multi-agent board** for durable, cross-turn, cross-profile pipelines. Layered on top of these native sub-agent primitives is first-class orchestration of third-party coding agents — Claude Code (Anthropic), Codex CLI (OpenAI), and OpenCode — each delivered as a bundled skill that Hermes loads and drives from its terminal tool. Additionally, Hermes exposes itself as an **ACP (Agent Client Protocol) server**, making it composable within any ACP-compatible orchestration ecosystem.[^2][^3][^4][^5][^6]

***

## 1. Architecture: Hermes as the Outer Loop

### 1.1 Core Design

Hermes' orchestration capability is rooted in a single, platform-agnostic `AIAgent` class (`run_agent.py`) that handles prompt construction, provider resolution, tool dispatch, retries, context compression, and session persistence. This same core is re-used by every entry point: CLI, messaging gateway, ACP adapter, batch runner, and API server — meaning orchestration behaviors are identical regardless of surface.[^7]

The codebase was substantially refactored in v0.15.0, collapsing the original 16,083-line monolithic `run_agent.py` to 3,821 lines (-76%) redistributed across 14 cohesive modules under `agent/`. This refactor did not change behavior but made the architecture far more navigable for contributors and power operators.[^8]

```
Entry Points:
   CLI (cli.py) | Gateway (gateway/run.py) | ACP (acp_adapter/) | API Server | Batch Runner
                                           ↓
                               AIAgent (run_agent.py)
                     ┌──────────────┬────────────────┬──────────────┐
                     │ Prompt       │ Provider       │ Tool         │
                     │ Builder      │ Resolution     │ Dispatch     │
                     └──────────────┴────────────────┴──────────────┘
                                     ↓                   ↓
                           Session Storage (SQLite)    Tool Backends
                                                 (Terminal, Browser, Web, MCP, File…)
```


### 1.2 Tool Registry and Delegation Surface

The central tool registry (`tools/registry.py`) manages 70+ tools across ~28 toolsets. Sub-agent orchestration is implemented in `tools/delegate_tool.py`; kanban state in `~/.hermes/kanban.db`; ACP wiring in `acp_adapter/`.[^9][^7]

***

## 2. `delegate_task` — Synchronous In-Turn Sub-Agents

### 2.1 What It Is

`delegate_task` spawns child `AIAgent` instances with isolated context, restricted toolsets, and their own terminal sessions. Each child begins with a completely fresh conversation — zero knowledge of the parent's history, tool calls, or prior context. The parent must pass everything the subagent needs through the `goal` and `context` fields.[^10]

Only the child's **final summary** re-enters the parent's context, keeping token usage efficient. Intermediate tool calls and reasoning never enter the parent's context window.[^11]

### 2.2 Single-Task Delegation

```python
delegate_task(
    goal="Debug why tests fail",
    context="Error: assertion in test_foo.py line 42. Project at /home/user/myproject.",
    toolsets=["terminal", "file"]
)
```


### 2.3 Parallel Batch Delegation

Up to **3 concurrent subagents by default** (configurable, no hard ceiling), executed via a `ThreadPoolExecutor`:

```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Research topic B", "toolsets": ["web"]},
    {"goal": "Fix the build",    "toolsets": ["terminal", "file"]}
])
```


Key batch behaviors:
- **Maximum concurrency:** 3 by default, configurable via `delegation.max_concurrent_children` or `DELEGATION_MAX_CONCURRENT_CHILDREN` env var (floor of 1, no hard ceiling)[^10]
- **Progress display:** In CLI mode, a tree-view shows tool calls from each subagent in real-time with per-task completion lines[^10]
- **Result ordering:** Results sorted by task index regardless of completion order[^10]
- **Interrupt propagation:** Interrupting the parent interrupts all active children, including grandchildren[^10]

### 2.4 Background Fan-Out (New in v0.18.0)

The v0.18.0 "Judgment Release" introduced a critical UX improvement: `delegate_task` can now fan out multiple subagents that all run **in the background** — the chat is never blocked, and when every subagent finishes, their results come back as a single consolidated turn.[^6][^1]

The new async delegation surface includes the following tools (in the `async_delegation` toolset):

| Tool | Purpose |
|------|---------|
| `delegate_task_async` | Spawn a background agent; returns a task identifier immediately |
| `check_task` | Non-blocking status + recent output from a running task |
| `steer_task` | Inject a message mid-flight for real-time course correction |
| `collect_task` | Block until task completes, retrieve full result |
| `cancel_task` | Stop a running task immediately |
| `list_tasks` | See all async tasks currently running in the session |

[^6]

Background agents run as in-process threads, reusing the same `AIAgent` machinery, credentials, and toolsets as synchronous `delegate_task`. The current limitation is single-session durability — async delegation does not yet persist across turn boundaries by default; cross-turn durability is targeted through an ongoing ACP initiative.[^6]

The TUI ships a `/agents` overlay (alias `/tasks`) that turns recursive `delegate_task` fan-out into a first-class audit surface with live tree view, per-branch cost/token/file rollups, kill and pause controls, and post-hoc per-subagent turn-by-turn review.[^10]

### 2.5 Toolset Isolation and Blocked Capabilities

```
Toolsets for leaf sub-agents (default):
    terminal, file, web, browser, vision, memory (read-only)

Always blocked for leaf subagents:
    delegation   — prevents runaway recursive spawning
    clarify      — subagents cannot interact with the user
    memory       — no writes to shared persistent memory
    code_execution — children reason step-by-step

Orchestrator subagents (role="orchestrator"):
    retain:  delegate_task
    blocked: clarify, memory, execute_code
```


### 2.6 Toolset Selection Guidance

| Toolset Pattern | Use Case |
|-----------------|----------|
| `["terminal", "file"]` | Code work, debugging, file editing, builds |
| `["web"]` | Research, fact-checking, documentation lookup |
| `["terminal", "file", "web"]` | Full-stack tasks (default) |
| `["file"]` | Read-only analysis, code review without execution |
| `["terminal"]` | System administration, process management |

[^10]

### 2.7 Depth Limit and Nested Orchestration

By default, delegation is **flat**: a parent (depth 0) spawns children (depth 1), and those children cannot delegate further. This is controlled by `delegation.max_spawn_depth` (default: **1**).[^10]

For multi-stage workflows, a parent can spawn **orchestrator** children:

```python
delegate_task(
    goal="Survey three code review approaches and recommend one",
    role="orchestrator",  # This child can spawn its own workers
    context="...",
)
```

- `role="leaf"` (default): child cannot delegate further
- `role="orchestrator"`: child retains the `delegation` toolset, gated by `max_spawn_depth`
- `delegation.orchestrator_enabled: false`: global kill switch forcing every child to leaf
- With `max_spawn_depth: 3` and `max_concurrent_children: 3`, up to 27 concurrent leaf agents are possible

[^10]

### 2.8 Model Override for Sub-Agents

Sub-agents can be routed to a different (e.g., cheaper) model than the parent:

```yaml
# ~/.hermes/config.yaml
delegation:
  model: "google/gemini-flash-2.0"  # cheaper model for subagents
  provider: "openrouter"             # optional: different provider
  api_mode: anthropic_messages       # optional; auto-detected for /anthropic endpoints
```


This enables cost-efficient architectures where inexpensive models handle boilerplate sub-tasks and expensive models handle hard reasoning.

### 2.9 Durability Constraints

`delegate_task` is **synchronous and not durable** — it runs inside the parent's current turn. If the parent is interrupted (user sends a new message, `/stop`, `/new`), all active children are cancelled. For work requiring durability, the alternatives are:[^10]
- `cronjob (action="create")` — schedules a separate agent run; immune to parent-turn interrupts[^10]
- `terminal(background=True, notify_on_complete=True)` — long-running shell commands that keep running independently[^10]
- **Kanban** — the primary durable multi-agent primitive (see Section 4)[^3]

***

## 3. Third-Party Coding Agents: Bundled Skills

Hermes ships a category of bundled skills under `skills/autonomous-ai-agents/` that teach it how to orchestrate commercial coding agent CLIs as sub-agents. These skills are loaded into the agent's context when needed and define the precise orchestration patterns, CLI flags, PTY handling, and monitoring strategies for each tool.[^5][^2]

### 3.1 Claude Code (Anthropic)

| Property | Value |
|----------|-------|
| Skill path | `skills/autonomous-ai-agents/claude-code` |
| Skill version | 2.2.0 |
| Author | Hermes Agent + Teknium |
| Prerequisite | `npm install -g @anthropic-ai/claude-code` (v2.x+) |
| Related skills | `codex`, `hermes-agent`, `opencode` |

[^2]

Hermes orchestrates Claude Code in **two distinct modes**:

#### Mode 1: Print Mode (`-p`) — Non-Interactive (Preferred)

One-shot execution, returns result, exits. No PTY needed. Preferred for most orchestration because it skips all interactive dialogs:

```python
terminal(
    command="claude -p 'Add error handling to all API calls in src/' --allowedTools 'Read,Edit' --max-turns 10",
    workdir="/path/to/project",
    timeout=120
)
```


Supports structured JSON output (`--output-format json`), streaming JSON (`stream-json`), piped input, `--json-schema` for structured extraction, session continuation (`--resume <id>`, `--fork-session`), and a `--fallback-model` for overload handling.[^2]

#### Mode 2: Interactive PTY via tmux — Multi-Turn Sessions

For iterative refactor → review → fix → test cycles, Claude Code runs inside a `tmux` session that Hermes controls via `send-keys` and `capture-pane`:

```python
# Start a tmux session
terminal(command="tmux new-session -d -s claude-work -x 140 -y 40")
# Launch Claude Code inside it
terminal(command="tmux send-keys -t claude-work 'cd /path/to/project && claude' Enter")
# Send task
terminal(command="sleep 5 && tmux send-keys -t claude-work 'Refactor the auth module to use JWT tokens' Enter")
# Monitor progress
terminal(command="sleep 15 && tmux capture-pane -t claude-work -p -S -50")
```


Hermes' Claude Code skill provides full PTY dialog-handling patterns, including the workspace trust dialog (Enter for default "Yes") and the `--dangerously-skip-permissions` bypass dialog (Down then Enter), plus a robust combined handler.[^2]

**Custom subagents within Claude Code:** Claude Code v2.x itself supports `--agents '<json>'` for defining custom subagents that can be invoked within Claude sessions (e.g., `@security-reviewer review the auth module`). These Claude-native subagents can be defined in `.claude/agents/` (project-level) or `~/.claude/agents/` (user-level) and are an additional level of agent nesting beneath Hermes.[^2]

**Full CLI flag reference exposed to Hermes:**

| Category | Notable Flags |
|----------|---------------|
| Session | `-p` (print), `-c` (continue), `-r` (resume), `--fork-session`, `--session-id` |
| Model | `--model`, `--effort`, `--max-turns`, `--max-budget-usd`, `--fallback-model` |
| Permissions | `--dangerously-skip-permissions`, `--permission-mode`, `--allowedTools`, `--disallowedTools` |
| Output | `--output-format`, `--json-schema`, `--verbose`, `--include-partial-messages` |
| Context | `--append-system-prompt`, `--bare`, `--agents`, `--mcp-config` |
| Worktree | `-w / --worktree`, `--tmux`, `--from-pr` |
| Agent Teams | `--teammate-mode`, `--brief` |

[^2]

**Rules Hermes follows when orchestrating Claude Code:**

1. Prefer print mode (`-p`) for single tasks — cleaner, no dialog handling, structured output
2. Use tmux for multi-turn interactive work
3. Always set `workdir` — keeps Claude focused on the right project directory
4. Set `--max-turns` in print mode to prevent infinite loops
5. Monitor tmux sessions with `tmux capture-pane -t <session> -p -S -50`
6. Look for the `❯` prompt indicating Claude is waiting for input
7. Clean up tmux sessions when done to avoid resource leaks

[^2]

### 3.2 Codex CLI (OpenAI)

| Property | Value |
|----------|-------|
| Skill path | `skills/autonomous-ai-agents/codex` |
| Skill version | 1.0.0 |
| Prerequisite | `npm install -g @openai/codex`; must run inside a git repository |
| Related skills | `claude-code`, `hermes-agent` |

[^12]

Codex is orchestrated via Hermes' terminal tool. Unlike Claude Code's dual-mode approach, Codex is always an interactive PTY app (`pty=true` required).[^12]

```python
# One-shot task
terminal(command="codex exec 'Add dark mode toggle to settings'", workdir="~/project", pty=True)

# Background mode for long tasks
terminal(command="codex exec --full-auto 'Refactor the auth module'", workdir="~/project", background=True, pty=True)
# Returns session_id; monitor with:
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")
process(action="submit", session_id="<id>", data="yes")  # for interactive prompts
process(action="kill", session_id="<id>")
```


Hermes can be configured to use Codex OAuth directly as a model provider (`model.provider: openai-codex`) after `hermes auth add openai-codex`, using Hermes-managed OAuth from `~/.hermes/auth.json`.[^12]

**Key orchestration rules:**
1. Always use `pty=true` — Codex is an interactive terminal app and hangs without a PTY
2. Git repo required — Codex refuses to run outside a git directory; use `mktemp -d && git init` for scratch
3. Use `exec` for one-shots; `--full-auto` for autonomous building
4. Use `background=True` for long tasks and monitor with `process` tool
5. Parallel Codex instances are supported — run multiple background sessions simultaneously

[^12]

### 3.3 OpenCode CLI

| Property | Value |
|----------|-------|
| Skill path | `skills/autonomous-ai-agents/opencode` |
| Purpose | Delegate coding to OpenCode CLI (features, PR review) |
| Related skills | `claude-code`, `hermes-agent` |

[^13][^5]

OpenCode is the third bundled coding-agent CLI, following the same terminal-orchestration pattern as Codex and Claude Code.

### 3.4 Hermes-Agent Spawning Skill

| Property | Value |
|----------|-------|
| Skill path | `skills/autonomous-ai-agents/hermes-agent` |
| Purpose | Coordinate spawned agents for bounded subtasks |
| Description | A skill that teaches Hermes the canonical patterns for spawning and managing its own subagents via `delegate_task` |

[^14][^13]

This skill encodes the structured orchestration playbook — when to delegate vs. execute directly, how to write effective `goal`/`context` fields, toolset selection patterns, and depth control.

### 3.5 Skill Bundle: `autonomous-ai-agents`

All four of the above skills (`claude-code`, `codex`, `opencode`, `hermes-agent`) live under the `autonomous-ai-agents/` category and ship **bundled by default** in a clean Hermes install. They are loaded into context via Hermes' skills system when the task context suggests coding agent orchestration is appropriate.[^13]

***

## 4. Kanban: Durable Multi-Agent Platform

### 4.1 Overview

The Hermes Kanban board is a durable, SQLite-backed task queue (`~/.hermes/kanban.db`) that enables multiple named agents — each a full Hermes **profile** with its own model, skills, and toolset — to collaborate on work across restarts, session boundaries, and human interruptions.[^3]

Kanban is the primary orchestration primitive for workflows `delegate_task` cannot handle:

- **Research triage** — parallel researchers + analyst + writer, human-in-the-loop
- **Scheduled ops** — recurring daily briefs building a journal over weeks
- **Digital twins** — persistent named assistants (`inbox-triage`, `ops-review`) accumulating memory over time
- **Engineering pipelines** — decompose → implement in parallel worktrees → review → iterate → PR
- **Fleet work** — one specialist managing N subjects (50 social accounts, 12 monitored services)

[^3]

### 4.2 `delegate_task` vs. Kanban

| Property | `delegate_task` | Kanban |
|----------|-----------------|--------|
| Shape | RPC call (fork → join) | Durable message queue + state machine |
| Parent blocks | Yes, until child returns | No — fire-and-forget after `create` |
| Child identity | Anonymous subagent | Named profile with persistent memory |
| Resumability | None — failed = failed | Block → unblock → re-run; crash → reclaim |
| Human in the loop | Not supported | Comment / unblock at any point |
| Agents per task | One call = one subagent | N agents over task's life |
| Audit trail | Lost on context compression | Durable rows in SQLite forever |
| Coordination | Hierarchical (caller → callee) | Peer — any profile reads/writes any task |

[^3]

**They coexist:** a Kanban worker may call `delegate_task` internally during its run.[^3]

### 4.3 Task Lifecycle

Tasks follow a defined state machine: `triage → todo → ready → running → blocked → done → archived`.[^3]

- **triage**: Parking column for rough ideas. With `kanban.auto_decompose: true` (default), the dispatcher auto-decomposes triage tasks into a graph of child tasks routed to the best-fit specialist profiles.
- **todo**: Awaiting dependencies (parent tasks) to complete
- **ready**: All parents done; eligible for dispatch
- **running**: Claimed and running under a worker profile
- **blocked**: Paused, awaiting human input or an upstream capability
- **done**: Worker called `kanban_complete`
- **archived**: Swept off the board

[^3]

### 4.4 Worker Interaction via `kanban_*` Tools

Workers **do not** shell out to `hermes kanban`. The dispatcher sets `HERMES_KANBAN_TASK=t_abcd` in the child's environment, which activates a dedicated `kanban_*` toolset in the model's schema:[^3]

| Tool | Purpose | Required |
|------|---------|----------|
| `kanban_show` | Read the current task (title, body, prior attempts, comments, full worker context) | — |
| `kanban_list` | List task summaries with filters (orchestrators) | — |
| `kanban_complete` | Finish with summary + metadata structured handoff | `summary` or `result` |
| `kanban_block` | Stop with typed reason: `dependency`, `needs_input`, `capability`, `transient` | `reason` |
| `kanban_heartbeat` | Signal liveness during long operations (required every hour to avoid reclaim) | — |
| `kanban_comment` | Append a durable note to the task thread | `task_id`, `body` |
| `kanban_create` | (Orchestrators) fan out into child tasks | `title`, `assignee` |
| `kanban_link` | (Orchestrators) add a parent→child dependency edge | `parent_id`, `child_id` |
| `kanban_unblock` | (Orchestrators) move a blocked task back to `ready` | `task_id` |

[^3]

### 4.5 Auto-Decomposition and Orchestration Topology

The v0.15.0 "Velocity Release" added **orchestrator auto-decomposition and swarm topology** as first-class Kanban primitives.[^15]

**Swarm topology** is created with a single command:

```bash
hermes kanban swarm
```

This creates a full Swarm v1 graph: root orchestrator, parallel workers, gated verifier, gated synthesizer, and a shared blackboard. Per-task model overrides (cheap models for boilerplate, expensive ones for hard sub-tasks) and per-task worktree paths/branches are configurable.[^8]

**Auto-decomposition flow:**

1. A task drops into the `triage` column
2. The dispatcher runs the `auxiliary.kanban_decomposer` model against the task body + installed profile roster
3. The LLM produces a JSON task graph: which tasks to spawn, who they go to, which depend on which
4. The original triage task becomes parent of every leaf, staying alive until the graph completes
5. The orchestrator profile (`kanban.orchestrator_profile`) judges overall completion

[^3]

**Config knobs:**

| Key | Default | Purpose |
|-----|---------|---------|
| `kanban.auto_decompose` | `true` | Dispatcher auto-runs the decomposer every tick |
| `kanban.auto_decompose_per_tick` | 3 | Cap on decompositions per tick |
| `kanban.orchestrator_profile` | `""` | Profile for the root/orchestration task |
| `kanban.default_assignee` | `""` | Where a child lands when LLM picks unknown profile |
| `kanban.dispatch_interval_seconds` | 60 | Tick interval |
| `kanban.dispatch_in_gateway` | `true` | Runs inside gateway (default) |

[^3]

### 4.6 Multi-Board Isolation

A single Hermes install supports multiple isolated boards (separate SQLite DBs, separate `workspaces/` and `logs/` directories). Workers spawned for a task see only their board's tasks via the `HERMES_KANBAN_BOARD` environment variable injected by the dispatcher.[^3]

### 4.7 Goal-Mode Cards

Kanban tasks can be run in **goal-loop mode** (`--goal` flag or `goal_mode=True`), which reuses the same Ralph-style `/goal` engine: after every turn, an auxiliary judge checks the worker's output against the card's title + body (treated as acceptance criteria), and the worker continues until the judge agrees or the turn budget runs out.[^3]

### 4.8 Pinning Extra Skills Per Task

The `kanban_create` tool's `skills` array lets an orchestrator load specialist skills onto a specific task without editing the assignee's profile:[^3]

```python
kanban_create(
    title="audit auth flow",
    assignee="reviewer",
    skills=["security-pr-audit", "github-code-review"],
)
```

***

## 5. ACP (Agent Client Protocol) Integration

### 5.1 Hermes as an ACP Server

Hermes exposes itself as an ACP server over stdio JSON-RPC, making it compatible with any ACP-aware host:[^16][^4]

```bash
hermes acp          # serve ACP on stdio
hermes acp --check  # verify installation
hermes acp --bootstrap  # print IDE install snippet
```

Used in production by VS Code (Zed Industries' ACP extension), Zed, and any JetBrains IDE with an ACP plugin.[^16]

**Toolset exposed in ACP mode:**
- `read_file`, `write_file`, `patch`, `search_files`
- `terminal`, `process`
- Web/browser tools
- Memory, todo, session search
- Skills
- `execute_code` and `delegate_task`
- Vision

Messaging delivery and cronjob management are intentionally excluded.[^4]

### 5.2 ACP in the Context of Sub-Agent Orchestration

ACP positions Hermes as a **sub-agent node** within a larger orchestration graph driven by an IDE or external orchestrator. In this topology:
- An IDE (VS Code, Zed, JetBrains) acts as the outer orchestrator
- Hermes runs as a stdio ACP agent server
- Hermes itself can spawn its own sub-agents via `delegate_task` internally

The ACP protocol exposes: session creation, prompt submission, streaming message chunks, tool-call events, permission requests, session fork, cancel, and authentication.[^16]

Each Hermes **profile** can be registered as a distinct ACP agent, enabling a multi-agent IDE setup where, for example, a frontend profile, backend profile, tester profile, and reviewer profile each run as separate ACP processes in VS Code.[^17]

### 5.3 ACP vs. ACP Registry

Hermes is listed in the **official ACP Registry** (via `agentclientprotocol/registry`), making it discoverable from Zed with `zed: acp registry` → search for "Hermes Agent" → one-click install. The registry entry uses `uvx --from 'hermes-agent[acp]==<version>' hermes-acp` as the launch command.[^4]

### 5.4 Programmatic Integration Options (Full Matrix)

| Protocol | Transport | Best For |
|----------|-----------|---------|
| **ACP** | JSON-RPC over stdio | IDE clients (VS Code, Zed, JetBrains) that already speak ACP |
| **TUI Gateway JSON-RPC** | JSON-RPC over stdio or WebSocket | Custom hosts wanting every Hermes feature (slash commands, approvals, multi-agent, session branching) |
| **API Server** | HTTP + Server-Sent Events | OpenAI-compatible frontends (Open WebUI, LobeChat) and language-agnostic HTTP clients |
| **Python in-process** | Direct import `run_agent.AIAgent` | Python programs that want to embed Hermes without subprocess overhead |

[^16]

**TUI Gateway sub-agent-relevant RPC methods:**

```
delegation.status     — query active subagent state
subagent.interrupt    — cancel a specific subagent mid-flight
spawn_tree.save       — persist the current subagent spawn tree
spawn_tree.list       — enumerate saved spawn trees
spawn_tree.load       — restore a saved spawn tree
session.steer         — direct the in-flight agent
```


***

## 6. Mixture-of-Agents (MoA) as a Routing Layer (New in v0.18.0)

v0.18.0 graduated Mixture-of-Agents from a toggleable mode to a **first-class model** in the Hermes model system.[^1]

Each named MoA preset appears as a selectable model under provider `moa` — pick it in any model picker and Hermes routes through the ensemble automatically. This is not sub-agent delegation in the `delegate_task` sense; it is an ensemble-inference pattern where multiple reference models deliberate in parallel before an aggregator synthesizes a final answer.[^1]

Key properties:
- Each reference model's full output renders as its own labelled block (visible in CLI, TUI, and desktop)[^1]
- The aggregator response streams live rather than appearing after a silence[^1]
- References see full tool state and fire on every user/tool response[^1]
- Opt-in full-turn trace persistence to JSONL (`moa.save_traces`) for debugging and eval[^1]

MoA is the **orchestration layer for inference diversity** — use it when you want multiple frontier models to deliberate on a hard question, not when you need parallel execution of tools or code tasks (which is `delegate_task`).

***

## 7. Verification & Goal Completion in Multi-Agent Contexts (New in v0.18.0)

v0.18.0 added evidence-based completion checking that is particularly relevant when orchestrating sub-agents:[^1]

### 7.1 Completion Contracts for `/goal`

Instead of stopping when the model "feels" done, `/goal` now judges completion against evidence you specify:

```
/goal "All API endpoints covered by integration tests"
  contract: "pytest tests/integration/ passes with 0 failures"
```


A `pre_verify` hook allows wiring in custom checks.[^1]

### 7.2 Coding Verification Evidence Ledger

A profile-scoped record of canonical project checks is maintained by `agent.coding_context`; the gateway exposes verification status. When a Kanban worker completes coding work, it can attach structured verification evidence to `kanban_complete(metadata={...})` following this recommended shape:[^1][^3]

```json
{
  "changed_files": ["path/to/file.py"],
  "verification": ["pytest tests/ -q"],
  "dependencies": ["parent task id or external issue"],
  "blocked_reason": null,
  "retry_notes": "what failed before, if this was a retry",
  "residual_risk": ["what was not tested or still needs human review"]
}
```


***

## 8. Scale-to-Zero and Production Deployment (New in v0.18.0)

v0.18.0 introduced gateway dormancy and drain coordination for operating Hermes at scale:[^1]

- **Scale-to-zero idle detection**: the gateway goes dormant when idle and wakes on demand — relevant for hosting a Hermes orchestrator on serverless infrastructure (Modal, Daytona)
- **External drain coordination (safe-shutdown Phase 2)**: quiesce cleanly before restart or migration without dropping in-flight conversations
- **Relay Phase 5/6**: wake primitives, going-idle/buffered-flip, `passthrough_forward` over WebSocket, multi-platform-per-agent identity

[^1]

***

## 9. Security Considerations in Multi-Agent Contexts

The v0.18.0 security round hardened several surfaces directly relevant to orchestration:[^1]

| Threat | Mitigation |
|--------|-----------|
| Promptware / Brainworm injection via tool output | Single source of truth threat patterns (`tools/threat_patterns.py`); tool results get delimiter markers; recalled memory scanned at load time |
| MCP-config persistence attack (malicious MCP server rewrites config) | Attack surface locked down in v0.18.0 |
| Cron `base_url` credential exfiltration | Overrides blocked |
| Credential leakage in sub-processes | `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` strips credentials from sub-processes |
| SSRF via browser tools | Cloud-metadata floor enforced on every backend |
| API server DoS from over-spawning | Configurable concurrent-run cap (`POST /v1/runs`) |
| Approval scope in ACP | Run approvals scoped by run ID |

[^2][^1]

**The `delegation` toolset is always blocked for leaf sub-agents** — this is a hard architectural guardrail against runaway recursive spawning that cannot be bypassed per-call.[^10]

***

## 10. Configuration Reference

### 10.1 Delegation Config (`~/.hermes/config.yaml`)

```yaml
delegation:
  max_iterations: 50              # Max LLM turns per child (default: 50)
  max_concurrent_children: 3     # Parallel children per batch (default: 3, floor 1)
  max_spawn_depth: 1             # Tree depth (default 1 = flat; raise to 2+ for nested)
  orchestrator_enabled: true     # Set false to force all children to leaf role
  child_timeout_seconds: 0       # 0 = no wall-clock timeout (opt-in hard cap)
  model: "google/gemini-flash-2.0"  # Optional cheaper model for subagents
  provider: "openrouter"            # Optional provider for subagents
  api_mode: chat_completions        # Wire protocol (auto-detected usually)
  # base_url: "http://localhost:1234/v1"  # For local model endpoints
```


### 10.2 Kanban Config (`~/.hermes/config.yaml`)

```yaml
kanban:
  dispatch_in_gateway: true          # Run dispatcher inside gateway process
  dispatch_interval_seconds: 60
  auto_decompose: true               # Auto-fan-out triage tasks
  auto_decompose_per_tick: 3
  orchestrator_profile: ""           # Profile for root orchestration tasks
  default_assignee: ""               # Fallback for unknown assignee names
  failure_limit: 2                   # Auto-block after N consecutive spawn failures
  dispatch_stale_timeout_seconds: 14400  # 4h: reclaim tasks with no heartbeat

auxiliary:
  kanban_decomposer:
    provider: openrouter
    model: google/gemini-flash-2.0   # Model for decomposition (cheaper is fine)
```


***

## 11. Capabilities Matrix

| Capability | Supported | Notes |
|------------|-----------|-------|
| Synchronous parallel sub-agents | ✅ | Up to N concurrent via `delegate_task(tasks=[...])` |
| Asynchronous background sub-agents | ✅ | `delegate_task(background=True)` + `async_delegation` toolset (v0.18.0+) |
| Nested orchestrator sub-agents | ✅ | `role="orchestrator"` + `max_spawn_depth ≥ 2` |
| Per-sub-agent model override | ✅ | `delegation.model` / `delegation.provider` |
| Sub-agent to different local endpoint | ✅ | `delegation.base_url`, `delegation.api_mode` |
| Claude Code delegation (print mode) | ✅ | Bundled skill `claude-code` v2.2.0 |
| Claude Code delegation (interactive PTY) | ✅ | tmux-based orchestration |
| Codex CLI delegation | ✅ | Bundled skill `codex` v1.0.0 |
| OpenCode delegation | ✅ | Bundled skill `opencode` |
| Durable multi-agent Kanban | ✅ | SQLite-backed, survives restarts |
| Auto-decomposition of tasks | ✅ | LLM-driven decomposer in Kanban triage |
| Swarm topology creation | ✅ | `hermes kanban swarm` |
| Per-task model overrides in Kanban | ✅ | Profile-level model selection |
| Per-task git worktrees | ✅ | Isolated coding sandboxes |
| Kanban goal-mode (Ralph loop) | ✅ | `--goal` / `goal_mode=True` |
| Human-in-the-loop gates | ✅ | Kanban `block`/`unblock`; comment threads |
| ACP server (IDE integration) | ✅ | `hermes acp`; Zed registry listed |
| ACP Registry listing | ✅ | Discoverable from Zed's ACP Registry |
| Multi-profile ACP (team agents in IDE) | ✅ | Per-profile ACP processes in VS Code/Zed |
| TUI sub-agent monitoring overlay | ✅ | `/agents` overlay with live tree, kill/pause controls |
| Mixture-of-Agents (ensemble inference) | ✅ | First-class MoA presets as selectable models (v0.18.0+) |
| Goal completion contracts with evidence | ✅ | Verification evidence ledger (v0.18.0+) |
| Scale-to-zero for hosted orchestrator | ✅ | Gateway dormancy + drain coordination (v0.18.0+) |
| Cross-turn durable async delegation | 🔄 In progress | ACP-based initiative; current async is session-scoped |

***

## 12. Known Limitations (as of v2026.7.1)

- **`delegate_task` is not durable across turns.** If the parent turn is interrupted, all active synchronous children are cancelled and their work is discarded. Use Kanban for work that must survive interrupts.[^10]
- **Background `delegate_task` is session-scoped.** The new async delegation is in-process; it does not persist across agent restarts. Cross-turn durability is targeted for a future release.[^6]
- **Subagents cannot interact with the user.** The `clarify` toolset is always blocked for children — subagent clarification must be anticipated and passed in via `context`.[^10]
- **Subagents cannot write to shared persistent memory.** The `memory` toolset (write operations) is blocked for leaf subagents.[^10]
- **Kanban is single-host.** The Kanban board operates on a single host's SQLite file. Cross-host Kanban (distributed multi-machine pipelines) is not supported natively.[^3]
- **Kanban attachment paths resolve on local terminal backend.** When workers run on Docker, Modal, or SSH backends, the attachment directory must be explicitly mounted.[^3]
- **ACP does not support dynamic runtime profile switching.** Each profile must be launched as a separate ACP server process.[^17]

---

## References

1. [NousResearch/hermes-agent v2026.7.1 on GitHub](https://newreleases.io/project/github/NousResearch/hermes-agent/release/v2026.7.1) - New release NousResearch/hermes-agent version v2026.7.1 Hermes Agent v0.18.0 (2026.7.1) — The Judgme...

2. [Delegate coding to Claude Code CLI (features, PRs) - Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-claude-code) - Claude Code v2.x can read files, write code, run shell commands, spawn subagents, and manage git wor...

3. [Kanban (Multi-Agent Board) | Hermes Agent](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) - Hermes Kanban is a durable task board, shared across all your Hermes profiles, that lets multiple na...

4. [ACP Editor Integration | Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp) - Use Hermes Agent inside ACP-compatible editors such as VS Code, Zed, and JetBrains. ... If you want ...

5. [Bundled Skills Catalog - Hermes Agent](https://hermes-agent.nousresearch.com/docs/reference/skills-catalog) - Configure, extend, or contribute to Hermes Agent. autonomous-ai-agents/hermes-agent. opencode, Deleg...

6. [Hermes Agent Now Runs Delegated Tasks in the Background ...](https://www.frontiernews.ai/news/article/hermes-agent-now-runs-delegated-tasks-in-the-backg-343723ce) - Hermes Agent now runs delegated tasks in the background, so your chat stays free while subagents han...

7. [Architecture | Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture) - Hermes Agent internals — major subsystems, execution paths, data flow, and where to read next

8. [Releases · NousResearch/hermes-agent - GitHub](https://github.com/NousResearch/hermes-agent/releases) - Hermes Agent v0.18.0 (v2026.7.1) Release Date: July 1, 2026 ・ 100% of them are closed. 1,693 files c...

9. [hermes-agent/tools/delegate_tool.py at main](https://github.com/NousResearch/hermes-agent/blob/main/tools/delegate_tool.py) - The agent that grows with you. Contribute to NousResearch/hermes-agent development by creating an ac...

10. [Subagent Delegation | Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation) - Subagent Delegation. The delegate_task tool spawns child AIAgent instances with isolated context, re...

11. [Delegation & Parallel Work | Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/guides/delegation-patterns) - When and how to use subagent delegation — patterns for parallel research, code review, and multi-fil...

12. [Codex CLI - hermes-agent](https://github.com/NousResearch/hermes-agent/blob/main/skills/autonomous-ai-agents/codex/SKILL.md) - The agent that grows with you. Contribute to NousResearch/hermes-agent development by creating an ac...

13. [Hermes Agent Guide | Skills System](https://hermes-agent.app/en/skills) - Structure and operate reusable Hermes Agent skills.

14. [Configure, extend, or contribute to Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-hermes-agent) - Hermes Agent is an open-source AI agent framework by Nous Research that runs in your terminal, a nat...

15. [Kanban becomes a multi-agent orchestration platform with auto ...](https://frontier.bitter.sh/signals/2026-06-03-hermes-agent-multi-agent-platform/) - Operators who ran Kanban as a task board must now decide whether to adopt orchestrator auto-decompos...

16. [Programmatic Integration | Hermes Agent - nous research](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration) - Used in production by VS Code (Zed Industries' ACP extension), Zed, and any JetBrains IDE with an AC...

17. [在VS Code 里跑一个多Agent 团队：Hermes ACP 集成实录_人工智能](https://gitcode.csdn.net/6a181c71662f9a54cb781cea.html) - 本文介绍了如何将Hermes Agent与VS Code集成，通过ACP协议实现智能开发辅助。主要内容包括：安装ACP依赖和客户端扩展；配置registryDir解决连接问题；利用profile机制实...

