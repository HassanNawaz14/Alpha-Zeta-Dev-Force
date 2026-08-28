# 07-repo-and-ui-setup

## Scope

This document defines the repo scaffold and UI wiring needed for both tiers, with emphasis on the final in-platform terminal-pane UI.

It is a file-by-file scaffold, not a package implementation.

## Repository Layout

```text
/
  package.json
  tsconfig.json
  README.md
  .gitignore
  /docs/
    alpha-zeta-dev-force-spec.md
    codex-generation-prompt.md
    /full_dev_plan/
      01-final-architecture.md
      02-v1-demo-milestone.md
      03-agent-contracts.md
      04-cli-adapters.md
      05-state-registry.md
      06-resource-governance.md
      07-repo-and-ui-setup.md
      08-build-plan.md
  /orchestrator/
    package.json
    tsconfig.json
    /src/
      index.ts
      manager.ts
      agent-runner.ts
      state-registry.ts
      resource-governor.ts
      task-graph.ts
      types.ts
      /adapters/
        cli-adapter.ts
        claude-code.ts
        opencode.ts
      /os/
        windows-job-object.ts
        cgroup.ts
  /ui/
    package.json
    tsconfig.json
    /src/
      main.tsx
      App.tsx
      /components/
        TerminalGrid.tsx
        TerminalPane.tsx
        AgentStatusBadge.tsx
      /services/
        pty-stream-client.ts
        registry-client.ts
  /state/
  /runs/
  /plans/
  /reports/
```

## File-By-File Purpose

### Root files

#### `package.json`

- workspace entry point
- shared scripts for dev, build, and test

#### `tsconfig.json`

- shared TypeScript compiler options

#### `README.md`

- short project entry point

#### `.gitignore`

- ignore generated logs, build output, and runtime artifacts

### `orchestrator/src/index.ts`

- starts the Manager
- initializes the State Registry
- wires adapters, runner, and resource governance

### `orchestrator/src/manager.ts`

- owns the task graph
- reads state files
- decides spawn, pause, resume, re-prompt, retry, and stop actions

### `orchestrator/src/agent-runner.ts`

- owns process and PTY lifecycle
- streams output
- writes state files
- records logs

### `orchestrator/src/state-registry.ts`

- atomic read/write helpers for `state/*.json`
- status normalization helpers

### `orchestrator/src/resource-governor.ts`

- recomputes live caps
- calls the platform-specific enforcement layer

### `orchestrator/src/task-graph.ts`

- encodes the docs → plan → build → test → commit flow
- provides explicit step transitions

### `orchestrator/src/types.ts`

- shared types for roles, states, task IDs, and artifact descriptors

### `orchestrator/src/adapters/cli-adapter.ts`

- defines the adapter interface used by both tiers

### `orchestrator/src/adapters/claude-code.ts`

- Claude Code adapter implementation

### `orchestrator/src/adapters/opencode.ts`

- OpenCode CLI adapter implementation

### `orchestrator/src/os/windows-job-object.ts`

- Windows resource enforcement helper

### `orchestrator/src/os/cgroup.ts`

- Linux/macOS resource enforcement helper

### `ui/src/main.tsx`

- frontend bootstrap

### `ui/src/App.tsx`

- page shell and layout

### `ui/src/components/TerminalGrid.tsx`

- lays out the terminal panes

### `ui/src/components/TerminalPane.tsx`

- binds one worker PTY stream to one terminal view

### `ui/src/components/AgentStatusBadge.tsx`

- displays state file status for a worker

### `ui/src/services/pty-stream-client.ts`

- subscribes to PTY output events from the orchestrator

### `ui/src/services/registry-client.ts`

- reads state updates for display

## Final UI Wiring

### Data flow

1. Agent Runner receives PTY output.
2. Agent Runner publishes the output stream.
3. UI subscribes to that stream by `agent_id`.
4. `TerminalPane` renders the bytes in a terminal emulator component.
5. `AgentStatusBadge` reflects the current state file.

### Pane contract

Each `TerminalPane` must know:

- `agent_id`
- `role`
- `task_id`
- `state`
- stream endpoint

### Interaction contract

The UI may:

- resize a pane, which forwards PTY dimensions to the Agent Runner
- request focus changes

The UI may not:

- invent state
- send worker prompts directly around the Agent Runner

## V1 UI Behavior

V1 does not use the in-platform terminal pane.

Instead, the orchestrator opens native OS terminal windows and still records the run in `runs/` and `state/`.

## Final UI Behavior

Final product UI:

- runs inside the app
- shows one pane per worker
- renders live orchestrator-owned PTYs
- provides visible separation of sessions

