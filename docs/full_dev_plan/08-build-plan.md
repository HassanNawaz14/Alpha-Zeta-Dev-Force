# 08-build-plan

## Scope

This document breaks implementation into discrete tasks small enough for one coding agent to complete independently.

The plan is phased:

- Phase 0: V1 demo
- Phase 1: final architecture

Each task lists exact inputs, deliverables, and dependencies.

## Phase 0: V1 Demo

### Task 0.1 - Create repo scaffold

- **Inputs:** `alpha-zeta-dev-force-spec.md`, `02-v1-demo-milestone.md`, `07-repo-and-ui-setup.md`
- **Deliverables:** root workspace files, `orchestrator/` scaffold, `ui/` scaffold, runtime folders
- **Dependencies:** none

### Task 0.2 - Define shared TypeScript types

- **Inputs:** `03-agent-contracts.md`, `04-cli-adapters.md`, `05-state-registry.md`
- **Deliverables:** `orchestrator/src/types.ts`, shared state and artifact interfaces
- **Dependencies:** Task 0.1

### Task 0.3 - Implement V1 CLI adapter interface and two adapters

- **Inputs:** `04-cli-adapters.md`
- **Deliverables:** `orchestrator/src/adapters/cli-adapter.ts`, `orchestrator/src/adapters/claude-code.ts`, `orchestrator/src/adapters/opencode.ts`
- **Dependencies:** Task 0.2

### Task 0.4 - Implement V1 process runner and log capture

- **Inputs:** `02-v1-demo-milestone.md`, `05-state-registry.md`
- **Deliverables:** `orchestrator/src/agent-runner.ts`, `runs/` log wiring, `state/` write wiring
- **Dependencies:** Task 0.2, Task 0.3

### Task 0.5 - Implement V1 Manager task graph

- **Inputs:** `01-final-architecture.md`, `02-v1-demo-milestone.md`, `03-agent-contracts.md`
- **Deliverables:** `orchestrator/src/manager.ts`, `orchestrator/src/task-graph.ts`
- **Dependencies:** Task 0.2, Task 0.4

### Task 0.6 - Implement V1 native terminal launcher

- **Inputs:** `02-v1-demo-milestone.md`, `07-repo-and-ui-setup.md`
- **Deliverables:** OS launch helpers under `orchestrator/src/os/`, V1 terminal launch behavior
- **Dependencies:** Task 0.4

### Task 0.7 - Implement V1 verification and reports

- **Inputs:** `03-agent-contracts.md`, `05-state-registry.md`
- **Deliverables:** `reports/test-report.md` generation path, diff verification logic, exit decision logic
- **Dependencies:** Task 0.4, Task 0.5

### Task 0.8 - Add V1 smoke tests

- **Inputs:** Tasks 0.1 through 0.7
- **Deliverables:** automated smoke tests for Planner → Builder → Tester → Git Manager flow
- **Dependencies:** Task 0.7

## Phase 1: Final Architecture

### Task 1.1 - Implement PTY-owned session layer

- **Inputs:** `01-final-architecture.md`, `04-cli-adapters.md`, `05-state-registry.md`
- **Deliverables:** final PTY spawn and ownership logic in `orchestrator/src/agent-runner.ts`
- **Dependencies:** Task 0.2, Task 0.3

### Task 1.2 - Implement final State Registry state machine

- **Inputs:** `01-final-architecture.md`, `05-state-registry.md`
- **Deliverables:** `orchestrator/src/state-registry.ts`, final state transitions and atomic writes
- **Dependencies:** Task 1.1

### Task 1.3 - Implement in-platform terminal pane UI

- **Inputs:** `01-final-architecture.md`, `07-repo-and-ui-setup.md`
- **Deliverables:** `ui/src/components/TerminalGrid.tsx`, `ui/src/components/TerminalPane.tsx`, `ui/src/services/pty-stream-client.ts`
- **Dependencies:** Task 1.1

### Task 1.4 - Implement live prompt injection and resize handling

- **Inputs:** `01-final-architecture.md`, `04-cli-adapters.md`
- **Deliverables:** PTY input forwarding, interrupt handling, resize propagation
- **Dependencies:** Task 1.1, Task 1.3

### Task 1.5 - Implement final Manager pause/resume/re-prompt control loop

- **Inputs:** `01-final-architecture.md`, `03-agent-contracts.md`, `05-state-registry.md`
- **Deliverables:** final Manager control logic in `orchestrator/src/manager.ts`
- **Dependencies:** Task 1.2, Task 1.4

### Task 1.6 - Implement resource governance enforcement

- **Inputs:** `06-resource-governance.md`
- **Deliverables:** `orchestrator/src/resource-governor.ts`, `orchestrator/src/os/windows-job-object.ts`, `orchestrator/src/os/cgroup.ts`
- **Dependencies:** Task 1.1

### Task 1.7 - Wire final UI to live registry data

- **Inputs:** `01-final-architecture.md`, `05-state-registry.md`, `07-repo-and-ui-setup.md`
- **Deliverables:** `ui/src/components/AgentStatusBadge.tsx`, `ui/src/services/registry-client.ts`, app integration wiring
- **Dependencies:** Task 1.2, Task 1.3

### Task 1.8 - Add final architecture integration tests

- **Inputs:** Tasks 1.1 through 1.7
- **Deliverables:** end-to-end tests for persistent sessions, state updates, terminal panes, and resource cap recomputation
- **Dependencies:** Task 1.7

## Assignment Rules

- No task may change another task’s deliverables without updating the dependency graph.
- Tasks in Phase 0 and Phase 1 may be assigned in parallel only when their dependencies are satisfied.
- The Manager implementation task must remain the only task that changes the task-graph rules.

