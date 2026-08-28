# 02-v1-demo-milestone

## Scope

This document defines the **V1 demo architecture** only.

V1 is a real implementation path for the same task graph, but it uses:

- one-shot worker invocations
- native OS terminal windows for visual effect
- exit-code-based task completion
- filesystem diff verification

It does not replace the final architecture in `01-final-architecture.md`.

## V1 Purpose

V1 exists to validate:

- Manager sequencing
- artifact handoffs
- retry policy
- plan/build/test/commit progression
- resource allocation formula

V1 explicitly does **not** validate the final PTY-based persistent session model.

## V1 Worker Model

### One-shot definition

A V1 worker is:

- spawned for one task
- given a complete prompt once
- allowed to run to exit
- discarded after completion or failure

There is no persistent worker session in V1.

### Execution characteristics

- interactive mode is still preferred where the CLI supports it
- the invocation is still a real CLI process
- the process is treated as task-scoped, not session-scoped
- the Manager does not resume the same worker process after exit

## V1 Terminal Behavior

### Visual requirement

For V1, the platform opens a native OS terminal window per worker invocation for visual effect.

The terminal window is not the authoritative state source. The authoritative state source is:

- process exit code
- filesystem diff
- captured stdout/stderr log

### OS launch requirements

#### Windows

Preferred launcher:

- `wt.exe` when available

Fallback launcher:

- `start cmd /k <command>`

#### macOS

Preferred launcher:

- `osascript` to request a Terminal.app window

Fallback launcher:

- `open -a Terminal <command>`

#### Linux

Preferred launcher:

- `x-terminal-emulator -e <command>`

Fallback launcher:

- the distro-local terminal emulator configured by the environment

The implementation must isolate shell-escaping so the worker prompt itself is not embedded directly into a shell command string when a temp-file prompt can be used instead.

## V1 Prompt Delivery

### Required prompt transport

The Manager writes the prompt to a temporary file and passes the file path to the worker command.

This is required to avoid shell-escaping problems, especially on Windows.

### Prompt content

The prompt must include:

- the role
- the task ID
- required input files
- required output file paths
- the exact success criteria

## V1 Control Flow

### Canonical sequence

1. Manager reads project docs.
2. Planner runs once and writes `plan.md`.
3. Manager verifies `plan.md`.
4. Builder runs once for the current plan step.
5. Manager verifies the code diff.
6. Tester runs once and writes `test-report.md`.
7. Manager evaluates the report.
8. If the report passes, Git Manager runs once.
9. Manager records completion.

### Retry behavior

If a step fails:

- the Manager may re-run the relevant role
- retries are bounded
- the Manager must not assume a failure can be resolved by the same worker process because the process does not persist in V1

## V1 State Model

### State set

V1 uses a simpler state set than the final architecture:

- `queued`
- `running`
- `succeeded`
- `failed`
- `timeout`

### State source

V1 state is derived from:

- process lifecycle
- exit code
- filesystem diff

No PTY heartbeat or marker parsing is required in V1.

## V1 Artifacts

### Required outputs

| Role | Required output |
|---|---|
| Planner | `plans/plan.md` |
| Builder | code changes in the repository |
| Tester | `reports/test-report.md` |
| Git Manager | committed changes |

### Shared logs

Each run writes logs under:

- `runs/<agent_id>/<timestamp>.log`

### Shared status

Each run writes:

- `state/<agent_id>.json`

## V1 End-To-End Demo Script

The demonstration must prove the full task graph with no manual terminal interaction after start.

### Demo inputs

- one small project brief in `docs/`
- one clean working tree
- one configured set of CLI adapters

### Demo steps

1. Start the Manager.
2. The Manager reads the seed docs.
3. The Manager launches the Planner as a one-shot process.
4. The Planner writes the plan artifact.
5. The Manager reads the plan and selects the first build step.
6. The Manager launches the Builder as a one-shot process.
7. The Builder writes code changes.
8. The Manager verifies the diff.
9. The Manager launches the Tester as a one-shot process.
10. The Tester writes the test report.
11. The Manager launches the Git Manager as a one-shot process.
12. The Git Manager commits the final result.

### Demo success condition

The demo succeeds only if:

- each role completes in sequence
- the expected artifacts exist
- the code diff is real
- the test report is present
- the final commit exists

## Relationship To The Final Architecture

V1 and the final architecture share:

- the same task graph
- the same artifact contracts
- the same resource-division formula

V1 differs only in execution and state sensing:

- one-shot process exit vs persistent PTY session
- native OS terminal window vs in-platform terminal pane
- exit-code state vs continuous PTY-based state registry

