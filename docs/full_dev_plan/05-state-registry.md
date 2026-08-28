# 05-state-registry

## Scope

This document defines the Agent Runner and State Registry mechanism for both tiers.

It covers:

- spawning
- PTY or process output streaming
- heartbeat tracking
- marker parsing
- filesystem diff verification
- Manager polling

## State Registry Purpose

The State Registry is the canonical machine-readable source of truth for worker progress.

The Manager must read only the State Registry, plus the filesystem diff, to decide the next action.

## State Registry Files

### Final tier

- `state/<agent_id>.json`
- `runs/<agent_id>/<timestamp>.log`

### V1 tier

- `state/<agent_id>.json`
- `runs/<agent_id>/<timestamp>.log`

The file layout is shared across tiers. The meaning of the state values differs by tier.

## Agent Runner Responsibilities

The Agent Runner owns:

- process spawn and teardown
- PTY allocation for final workers
- stdout/stderr capture for V1 workers
- log file creation
- state file creation and updates
- periodic diff snapshots
- status normalization

The Agent Runner is the only component allowed to update `state/<agent_id>.json`.

## Final Tier Mechanism

### Output stream handling

For each final-tier worker:

1. The Agent Runner receives every byte of PTY output.
2. The Agent Runner forwards the bytes to the UI terminal pane.
3. The Agent Runner appends the bytes to the run log.
4. The Agent Runner timestamps the last observed output activity.

### Heartbeat and idle detection

Idle detection is based on output silence.

Rule:

- if no output has arrived for the configured idle threshold, the worker becomes a candidate for `idle-waiting-for-input`

The Agent Runner may not mark the worker idle unless the current task is otherwise stable.

### Marker parsing

The Agent Runner must scan the live output for recognizable task markers such as:

- explicit success text
- explicit failure text
- file-write confirmations
- test-pass or test-fail phrases

Marker parsing is advisory. It may accelerate state updates, but it may not override filesystem evidence.

### Filesystem diff verification

The Agent Runner must periodically compare the workspace against the previous snapshot.

Required verification sources:

- `git status`
- `git diff`

The diff is the tie-breaker when output and reality disagree.

### Final state derivation

The Agent Runner combines:

- PTY heartbeat
- marker parsing
- filesystem diff

to derive:

- `queued`
- `starting`
- `running`
- `idle-waiting-for-input`
- `succeeded`
- `failed`
- `timeout`

## V1 Tier Mechanism

### Process handling

For each V1 worker:

1. Write the prompt to a temporary file.
2. Launch the CLI as a one-shot process.
3. Stream stdout/stderr to `runs/<agent_id>/<timestamp>.log`.
4. Capture PID, exit code, and duration.
5. Compare the filesystem diff after exit.
6. Write `state/<agent_id>.json`.

### V1 state derivation

V1 derives state from:

- process running or exited
- exit code
- filesystem diff

V1 does not require PTY heartbeat logic or marker parsing.

## State Update Rules

### Required write timing

The Agent Runner must update the state file:

- on spawn
- on first output
- on idle detection
- on success
- on failure
- on timeout
- on exit

### State write format

The state file must be written atomically.

The write pattern must avoid partial reads by the Manager.

### Terminal states

Terminal states are:

- `succeeded`
- `failed`
- `timeout`

Once a terminal state is written for the current task, the Manager may advance the graph.

## Manager Polling Contract

### Required behavior

The Manager must poll the State Registry on a fixed interval or subscribe to file changes with equivalent semantics.

The required compatibility contract is:

- read current state file
- compare against the last observed state
- react only to durable changes

### Polling guarantees

The Manager must:

- tolerate transient incomplete writes by retrying the read
- never infer state from UI text
- never infer state from worker self-reporting alone

### Recommended order of inspection

1. Read `state/<agent_id>.json`
2. Confirm task-specific artifact paths
3. Confirm filesystem diff
4. Decide next action

## Failure Rules

A worker must be marked `failed` when:

- the process exits nonzero before success
- the adapter reports an irrecoverable startup failure
- the diff verification contradicts the claimed success

A worker must be marked `timeout` when:

- the configured maximum execution window is exceeded

