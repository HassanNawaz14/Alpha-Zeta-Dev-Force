# 01-final-architecture

## Scope

This document defines the **final product architecture** only.

It covers:

- The Manager
- Persistent worker sessions
- Orchestrator-owned PTYs
- In-platform terminal-pane UI
- Prompt injection, pause, resume, and re-prompt behavior
- The final-product control flow and state transitions

It does not define the V1 one-shot demo execution model. That is documented in `02-v1-demo-milestone.md`.

## Top-Level Architecture

### Fixed structural rule

The final product has exactly one orchestration authority:

- `Manager`: a single persistent process that decides what each worker should do next
- `Agent Runner`: the process-management layer that owns worker PTYs and produces state records
- `Worker Agent`: a persistent interactive CLI agent session bound to one PTY
- `UI`: a local app surface that renders one terminal pane per active worker session

The Manager never interacts with a worker by reading chat text alone. It uses the State Registry produced by the Agent Runner.

### Required process ownership model

For each worker:

1. The Agent Runner spawns the worker CLI in interactive mode.
2. The Agent Runner allocates a PTY and attaches the worker to that PTY.
3. The PTY belongs to the orchestrator, not to the worker.
4. The PTY output is streamed to the UI terminal pane.
5. The Manager sends control input to the worker by writing into the PTY input stream.

The UI is a viewer/controller of orchestrator-owned PTYs. It is not the process owner.

## Final Product Responsibilities

### Manager

The Manager:

- Reads project docs and task artifacts
- Creates the task graph
- Starts workers when work becomes available
- Pauses workers when a task is complete or superseded
- Resumes workers when the next step requires the same persistent session
- Re-prompts workers with new instructions
- Reads the State Registry to determine progress
- Enforces retry limits
- Advances from docs to plan to build to test to commit

The Manager never assumes completion from an agent message if the State Registry does not confirm it.

### Agent Runner

The Agent Runner:

- Spawns and supervises worker processes
- Owns PTY creation and teardown
- Streams output to the UI and log storage
- Maintains live worker state records
- Detects idle, success, failure, and timeout conditions
- Provides the Manager with canonical status data

### UI

The UI:

- Renders each worker session in its own terminal pane
- Mirrors the PTY output stream
- Sends user-visible terminal sizing and focus events to the PTY layer when needed
- Does not interpret task state on its own

## Worker Session Model

### Persistent session definition

A persistent worker session is a long-lived worker process attached to a single PTY that may receive multiple task instructions across its lifetime.

Session identity is stable for the life of the process:

- one `agent_id`
- one PTY
- one live `state/<agent_id>.json`
- one log stream

### Session lifecycle

1. `queued`: the worker has been selected but not yet spawned
2. `starting`: the PTY and process are being created
3. `running`: the worker is actively processing the current instruction
4. `idle-waiting-for-input`: the worker is alive and waiting for the next instruction
5. `succeeded`: the current task completed successfully
6. `failed`: the current task failed irrecoverably or hit a task-level error condition
7. `timeout`: the worker exceeded the allowed execution window for the current task

The Manager may keep a session alive after `succeeded` if the same worker is intended to receive a follow-up instruction later. In that case, `succeeded` is a task result, not necessarily a process shutdown signal.

## Control Flow

### Canonical control loop

1. The Manager reads the current task graph and the State Registry.
2. The Manager selects the next required role.
3. The Manager ensures the required worker session exists.
4. If the session does not exist, the Agent Runner spawns it.
5. The Manager injects the next instruction into the worker PTY.
6. The Agent Runner observes PTY output and filesystem changes.
7. The Agent Runner updates `state/<agent_id>.json`.
8. The Manager waits for the relevant session state to become terminal for the current task.
9. The Manager advances the graph or retries according to policy.

### Pause semantics

Pause means:

- do not send new task instructions to that worker
- if the worker is currently executing and must stop, inject an interrupt into the PTY
- update state so the worker is no longer treated as active for the current task

Pause is a Manager decision. It is not a UI action.

### Resume semantics

Resume means:

- keep the same persistent worker session
- send the next instruction into the same PTY
- continue with the same worker identity and log stream
- do not create a new worker process unless the existing session has failed or exited

### Re-prompt semantics

Re-prompt means:

- write a new instruction block into the same PTY
- include only the missing or corrected task context
- preserve any shared session context already established in the persistent worker

The re-prompt must not rely on free-form chat memory. It must be backed by explicit task context and file paths.

## PTY Ownership And Input Injection

### PTY requirements

Each worker PTY must support:

- bidirectional text I/O
- resize events
- interrupt injection
- live output capture

### Prompt injection behavior

The Manager sends instructions by writing to the PTY input stream, including:

- the next task prompt
- follow-up constraints
- retry instructions
- interruption sequences when required

The Manager may inject control characters such as `Ctrl+C` only through the PTY layer and only when the current task must be paused or aborted.

### Safety rule

No code path may write directly to a worker process stdin unless that path is mediated by the Agent Runner PTY owner. This keeps the PTY stream authoritative.

## State Transition Rules

### State machine

| Current state | Event | Next state |
|---|---|---|
| `queued` | worker process spawn begins | `starting` |
| `starting` | first live PTY output arrives | `running` |
| `running` | no output for configured idle threshold | `idle-waiting-for-input` |
| `idle-waiting-for-input` | Manager sends new prompt | `running` |
| `running` | task completes successfully | `succeeded` |
| `running` | task fails | `failed` |
| `running` | execution window exceeded | `timeout` |
| `idle-waiting-for-input` | worker exits cleanly after completion | `succeeded` |
| `idle-waiting-for-input` | worker exits nonzero | `failed` |
| any live state | process terminates unexpectedly | `failed` or `timeout`, based on cause |

### Transition rules

- `queued` and `starting` are orchestration states only.
- `running` means the worker is actively processing the current instruction or is still producing task-relevant output.
- `idle-waiting-for-input` means the worker is alive, has no pending failure, and is waiting for a new Manager instruction.
- `succeeded` is emitted only when the task-level success conditions are confirmed by state evidence.
- `failed` is emitted when the task-level failure conditions are confirmed by state evidence.
- `timeout` is emitted when the task exceeds the configured maximum allowed duration.

### Invariant

The Manager may advance the graph only when the State Registry reports a terminal result for the current task.

## UI Architecture

### Terminal panes

The UI must render:

- one pane per active worker session
- live PTY output
- pane status labels from the State Registry
- focus and resize behavior typical of a terminal surface

### UI data flow

1. PTY output is captured by the Agent Runner.
2. Output is streamed to the frontend.
3. The frontend renders the output inside `xterm.js` or an equivalent terminal component.
4. The frontend never fabricates output or state.

### Visual requirement

The UI must present separate terminals as separate panes or tabs inside the app. Detached native terminal windows are not part of the final product.

## Final Product Task Graph

The final product always follows a graph with these logical stages:

1. Read source docs
2. Produce `plan.md`
3. Implement code changes
4. Run tester validation
5. Apply git management steps
6. Finish or retry

This graph is fixed, but the set of workers may be extended with custom roles.

## Persistence And Logs

The final product must keep:

- live state files under `state/`
- per-session logs under `runs/<agent_id>/`
- task outputs under their designated plan/report locations

The log stream is append-only for the life of a session.

## Deferred Item

The seed spec’s `OPEN (V2)` local IPC replacement for polling is deferred. This document does not define that future mechanism.

