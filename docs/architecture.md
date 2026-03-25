# Agent Architecture — How Mentor & Executor Work

> This document explains the dual-agent architecture behind **The Pair**: how the Mentor (planner/reviewer) and Executor (coder) agents work, how they communicate, how the system launches them, and what happens when an agent raises a question or needs human intervention.

---

## Table of Contents

- [Overview](#overview)
- [Agent Roles](#agent-roles)
- [System Components](#system-components)
- [How Agents Are Launched](#how-agents-are-launched)
- [Agent Communication Flow](#agent-communication-flow)
- [The Handoff Mechanism](#the-handoff-mechanism)
- [Iteration Loop & Task Completion](#iteration-loop--task-completion)
- [Human Intervention & Permission Requests](#human-intervention--permission-requests)
- [Session Persistence & Recovery](#session-persistence--recovery)
- [FAQ](#faq)

---

## Overview

The Pair implements a **dual-agent cross-validation model**. Instead of relying on a single AI to write and self-review code, two separate agents work together:

| Agent | Also Known As | Purpose |
|-------|---------------|---------|
| **Mentor** | Planner / Reviewer | Plans tasks, reviews results, validates code |
| **Executor** | Coder | Implements code, runs commands, reports results |

They take turns in a loop: the Mentor plans → the Executor implements → the Mentor reviews → and so on, until the task is complete or a human steps in.

```
         ┌──────────┐        ┌──────────┐
         │  Mentor  │───────▶│ Executor │
         │ (plans)  │◀───────│ (codes)  │
         └──────────┘        └──────────┘
              │                    │
              └──── The Pair ──────┘
                  orchestrates
                  both agents
```

> **Key insight:** The Mentor and Executor never talk to each other directly. The Pair app sits in the middle, passing context from one agent to the next via structured prompts.

---

## Agent Roles

### Mentor Agent (Planner / Reviewer)

- **Access level:** Read-only — cannot modify files or run destructive commands
- **Responsibilities:**
  - Analyze the task specification provided by the human
  - Create a detailed, step-by-step plan for the Executor to follow
  - Review the Executor's work after each iteration
  - Signal task completion by including a `TASK_COMPLETE` token in its output when all requirements are met
- **Output type:** `Plan` messages

### Executor Agent (Coder)

- **Access level:** Full — can write files and run commands within the project workspace
- **Responsibilities:**
  - Execute the Mentor's plan step by step
  - Run commands (build, test, install dependencies, etc.)
  - Report progress and results
  - Does **not** self-review — the Mentor reviews instead
- **Output type:** `Result` messages

---

## System Components

The dual-agent workflow is orchestrated by four backend components working together:

```
┌─────────────────────────────────────────────────────────┐
│  Frontend (React + Zustand)                             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  usePairStore — global state & event listeners   │  │
│  └──────────────────────────────────────────────────┘  │
│                        ↕ Tauri IPC events               │
├─────────────────────────────────────────────────────────┤
│  Backend (Rust)                                         │
│  ┌──────────────┬───────────────┬──────────────────┐   │
│  │ PairManager  │ MessageBroker │ ProcessSpawner   │   │
│  │              │               │                  │   │
│  │ Creates &    │ Tracks state, │ Spawns opencode  │   │
│  │ manages      │ messages,     │ CLI processes    │   │
│  │ pair         │ iterations,   │ for each agent   │   │
│  │ lifecycle    │ status        │ turn             │   │
│  └──────────────┴───────────────┴──────────────────┘   │
│                        ↕ Child processes                │
│               ┌────────────────────┐                    │
│               │   opencode CLI     │                    │
│               │  (AI agent runner) │                    │
│               └────────────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

| Component | File | Role |
|-----------|------|------|
| **PairManager** | `src-tauri/src/pair_manager.rs` | Creates pairs, assigns tasks, coordinates lifecycle |
| **MessageBroker** | `src-tauri/src/message_broker.rs` | State machine — tracks status, messages, iterations |
| **ProcessSpawner** | `src-tauri/src/process_spawner.rs` | Spawns agent CLI processes, streams output, handles handoffs |
| **usePairStore** | `src/renderer/src/store/usePairStore.ts` | Frontend state management, event listeners, handoff orchestration |

---

## How Agents Are Launched

Agents are not long-running services. Each agent "turn" is a **single CLI process** invocation:

### Step-by-step launch sequence

1. **User creates a pair** and provides a task description (spec)
2. **PairManager** calls `pair_assign_task()`, which prepares the run
3. **MessageBroker** sets the initial status (`Mentoring`) and activity phases
4. **ProcessSpawner** builds the CLI command via `ProviderAdapter::build_turn_command()` and spawns a child process. The exact command depends on the configured provider (see [Provider-Specific Commands](#provider-specific-commands) below).

5. The spawner attaches async readers to **stdout** and **stderr**
6. As the agent produces output, the spawner:
   - Parses JSON events (session IDs, status updates, tool calls)
   - Extracts text content from the event stream
   - Emits real-time `pair:message` events to the frontend
7. When the process **exits**, the spawner finalizes the output into a `Message` and evaluates whether to hand off to the next agent

### What gets spawned

Each agent turn runs a provider CLI as a child process with:
- **stdout**: JSON event stream + text output
- **stderr**: Error logging
- **Working directory**: The project directory selected for the pair

### Provider-Specific Commands

The Pair supports multiple AI agent providers. Each provider has its own CLI and command structure. The `ProviderAdapter` (in `src-tauri/src/provider_adapter.rs`) builds the correct command for each provider:

#### opencode (default provider)

```
opencode run \
  --model <model-id> \
  --session <session-id> \
  --format json \
  "<task-prompt>"
```

- Uses `run` subcommand with the task prompt as a positional argument
- `--format json` enables JSON event streaming on stdout
- `--session` resumes an existing session (omitted on first turn)

#### codex

```
codex exec [resume <session-id>] \
  --model <model-id> \
  [--sandbox read-only] \
  --json \
  --output-last-message <temp-file-path> \
  "<task-prompt>"
```

- Uses `exec` subcommand; `resume <session-id>` continues an existing session
- `--sandbox read-only` is applied for the Mentor role to enforce read-only access
- `--output-last-message` writes the final response to a temp file for reliable extraction

#### claude

```
claude -p \
  --model <model-id> \
  --output-format stream-json \
  --permission-mode plan|auto \
  [--resume <session-id>] \
  "<task-prompt>"
```

- `-p` enables non-interactive (programmatic) mode
- `--permission-mode plan` for Mentor (read-only), `auto` for Executor (full access)
- `--output-format stream-json` enables JSON event streaming

#### gemini

```
gemini \
  --model <model-id> \
  --prompt "<task-prompt>"
```

- Simplest invocation — model and prompt as flags
- Uses plain stdio for input/output

---

## Agent Communication Flow

The Mentor and Executor **do not communicate directly**. The Pair app orchestrates their communication through a structured handoff loop:

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETE AGENT WORKFLOW                       │
│                                                                 │
│  1. User assigns task                                          │
│     ↓                                                          │
│  2. MENTOR receives task spec                                  │
│     → Produces a step-by-step PLAN                             │
│     ↓                                                          │
│  3. System validates plan, hands off to Executor               │
│     ↓                                                          │
│  4. EXECUTOR receives Mentor's plan                            │
│     → Implements code, runs commands                           │
│     → Produces RESULTS                                         │
│     ↓                                                          │
│  5. System checks iteration limits, hands off to Mentor        │
│     ↓                                                          │
│  6. MENTOR receives Executor's results                         │
│     → Reviews work                                             │
│     → Either signals TASK_COMPLETE or provides refined plan    │
│     ↓                                                          │
│  7. If not complete → back to step 4 (next iteration)          │
│     If complete → Finished ✓                                   │
└─────────────────────────────────────────────────────────────────┘
```

### What each agent sees

**Mentor's first turn** — receives the raw task specification:
```
You are the Mentor. Analyze this task and create a detailed execution plan.

Task: <user's task description>
```

**Executor's turn** — receives the Mentor's plan as a command:
```
Your mission is ONLY to EXECUTE the plan provided below.
- DO NOT create new plans.
- DO NOT review your own work.
- JUST EXECUTE THE STEPS and report results.

--- PLAN TO EXECUTE ---
<mentor's plan content>
```

**Mentor's review turn** — receives the Executor's results:
```
Your mission is ONLY to PLAN and REVIEW.
- DO NOT execute any code or tools.

The Executor has finished a turn. Review their results:

<executor's result content>

If all requirements are met, include "TASK_COMPLETE" in your response.
Otherwise, provide a refined PLAN for the next iteration.
```

---

## The Handoff Mechanism

The handoff is the core mechanism that connects the two agents. It works across both the Rust backend and the React frontend:

### Backend (ProcessSpawner)

When an agent's process exits, the spawner evaluates completion conditions:

```
Agent process exits
    ↓
┌─ Does output contain "TASK_COMPLETE"?
│   YES → Status = Finished, STOP
│
├─ Was there no text output at all?
│   YES → Status = AwaitingHumanReview, STOP
│
├─ Has the Executor reached maxIterations?
│   YES → Status = AwaitingHumanReview, STOP
│
└─ Otherwise → Emit "pair:handoff" event
                with { pairId, nextRole }
```

### Frontend (usePairStore)

The frontend listens for `pair:handoff` events and orchestrates the next turn:

1. Receives the handoff event with the next role (`"executor"` or `"mentor"`)
2. Retrieves the last message from the previous agent
3. **Validates the output** — if the Mentor's plan is malformed, triggers a repair by re-running the Mentor with a corrective prompt
4. **Builds a context-aware prompt** for the next agent (see examples above)
5. Calls `assignTask()` to trigger the next turn — which spawns a new CLI process

This frontend orchestration allows the handoff logic to be flexible and context-aware, incorporating the full conversation history.

---

## Iteration Loop & Task Completion

### State machine

Each pair follows this state machine:

```
Idle
  ↓ (user assigns task)
Mentoring  ← (Mentor is actively planning or reviewing)
  ↓ (handoff)
Executing  ← (Executor is actively implementing)
  ↓ (handoff)
Reviewing  ← (Mentor is reviewing Executor's work)
  │
  ├─ Mentor says "TASK_COMPLETE" → Finished ✓
  ├─ Max iterations reached     → AwaitingHumanReview ⏸
  ├─ No output produced         → AwaitingHumanReview ⏸
  └─ More work needed           → back to Executing (next iteration)
```

### Iteration counting

- Each Executor turn increments the iteration counter
- The `maxIterations` limit (configured per pair) acts as a safety valve
- When the limit is reached, the system pauses for human review rather than continuing indefinitely

### How a task completes

The Mentor — not the Executor — decides when a task is complete. When the Mentor is satisfied with the results, it includes the `TASK_COMPLETE` token in its response. The ProcessSpawner detects this token and sets the pair status to `Finished`.

---

## Human Intervention & Permission Requests

### When does the system pause for humans?

The system pauses and enters `AwaitingHumanReview` status in these scenarios:

| Scenario | Trigger | What the user sees |
|----------|---------|-------------------|
| **Max iterations reached** | Executor completes a turn at the iteration limit | Pair pauses with all messages visible; user can approve to continue or reject to stop |
| **No agent output** | An agent produces no text output (possible error) | Error message explaining the agent returned no output |
| **Agent encounters issues** | Agent cannot proceed (permission denied, invalid state) | Last agent message displayed with context |

### What happens when a user intervenes

The UI presents two options:

- **Approve** — The system continues with the next agent turn, picking up where it left off
- **Reject** — The system stops execution and sets the status to `Error`

```
User clicks "Approve"
    ↓
Backend: broker.record_human_feedback(pairId, approved=true)
    ↓
Adds a human feedback message to the conversation
    ↓
Returns the next role to continue
    ↓
Frontend triggers the next agent turn

---

User clicks "Reject"
    ↓
Backend: broker.record_human_feedback(pairId, approved=false)
    ↓
Sets status = Error
    ↓
Adds rejection message to conversation
    ↓
Execution stops — user can review files or start a new task
```

### How permissions work at the agent level

The underlying AI agent CLI (opencode) handles file-system and command permissions. The Pair configures this via a `PermissionStrategy`:

- **Auto** — Permissions are granted automatically (default for full automation)
- **ManualConfirm** — Each permission request is surfaced to the user
- **PreApproved** — Uses a pre-approved list of allowed operations

In the default automation mode, agents operate with workspace-scoped permissions: they can read and write files within the project directory and execute commands. If an agent hits a permission boundary enforced by the CLI, it reports back through its output, and the system surfaces this as part of the conversation.

---

## Session Persistence & Recovery

Sessions are automatically saved at key checkpoints:

- After each agent turn completes
- When a session ID changes
- On any error

### What gets saved

Each pair's state is persisted to `.pair/runtime/<pairId>/snapshot.json` inside the project directory, including:

- Full conversation history (all messages)
- Current iteration count and status
- Session IDs for both Mentor and Executor (allowing continuity)
- Agent activity state

### Recovering a session

If the app is closed or crashes during an active run:

1. On next launch, The Pair scans for recoverable sessions
2. A **Session Recovery Modal** shows available sessions
3. The user can choose to **restore** a session (optionally continuing the run) or **discard** it
4. Restored sessions resume from the exact state they were in, including the next agent's turn

---

## FAQ

**Q: Do the Mentor and Executor share memory or context?**

A: Not directly. Each agent turn is an independent CLI process. The Pair passes context between them by including the previous agent's output in the next agent's prompt. The opencode CLI may maintain its own session state across turns via session IDs, which The Pair manages and caches.

**Q: Can the Executor ignore the Mentor's plan?**

A: The Executor receives the Mentor's plan as its task prompt and is instructed to follow it. However, since the Executor is an AI model, it may deviate. The Mentor's review in the next turn catches deviations and can provide corrective feedback.

**Q: What if the Mentor's plan is malformed?**

A: The frontend validates the Mentor's plan before handing off to the Executor. If the plan doesn't meet structural expectations, the system triggers a "repair" — re-running the Mentor with a corrective prompt asking it to produce a proper plan.

**Q: Can I use different AI models for Mentor and Executor?**

A: Yes. Each pair allows independent model selection for the Mentor and Executor. This is a core feature — using different models for planning and execution increases the diversity of validation.

**Q: What happens if an agent gets stuck in an infinite loop?**

A: The `maxIterations` limit prevents infinite loops. Once the configured number of iterations is reached, the system pauses for human review. The user can then approve additional iterations or stop the run.

**Q: Can I manually send a message to an agent?**

A: The current workflow is automated — messages flow through the Mentor ↔ Executor loop. Humans interact by approving/rejecting at pause points or by assigning new tasks. The conversation history is fully visible in the Console view.
