# 03-agent-contracts

## Scope

This document defines the per-role session contract for both tiers.

It covers:

- required inputs
- required outputs
- artifact locations
- state reporting requirements

It does not redefine orchestration mechanics already covered in `01-final-architecture.md` and `02-v1-demo-milestone.md`.

## Shared Contract Terms

### Required input categories

Every role receives:

- role name
- task ID
- workspace root
- explicit source documents
- explicit output paths
- success criteria
- retry context if the task is a retry

### Shared artifact locations

| Artifact type | Path |
|---|---|
| Plan | `plans/plan.md` |
| Test report | `reports/test-report.md` |
| Live state | `state/<agent_id>.json` |
| Run log | `runs/<agent_id>/<timestamp>.log` |

### Shared identity fields

Every worker contract uses:

- `agent_id`
- `role`
- `task_id`
- `tier` equal to `v1` or `final`

## Shared `status.json` Schema

The state file written by the Agent Runner must satisfy the following structure.

### Final tier schema

```json
{
  "agent_id": "builder-3",
  "role": "builder",
  "task_id": "plan-step-2",
  "tier": "final",
  "pid": 18234,
  "state": "queued | starting | running | idle-waiting-for-input | succeeded | failed | timeout",
  "started_at": "2026-08-28T10:00:00Z",
  "ended_at": "2026-08-28T10:12:00Z",
  "exit_code": 0,
  "artifacts": ["src/api/routes.py"],
  "log_path": "runs/builder-3/20260828T100000Z.log"
}
```

### V1 tier schema

```json
{
  "agent_id": "builder-3",
  "role": "builder",
  "task_id": "plan-step-2",
  "tier": "v1",
  "pid": 18234,
  "state": "queued | running | succeeded | failed | timeout",
  "started_at": "2026-08-28T10:00:00Z",
  "ended_at": "2026-08-28T10:12:00Z",
  "exit_code": 0,
  "artifacts": ["src/api/routes.py"],
  "log_path": "runs/builder-3/20260828T100000Z.log"
}
```

### Required fields

Every `status.json` must include:

- `agent_id`
- `role`
- `task_id`
- `tier`
- `pid`
- `state`
- `started_at`
- `ended_at`
- `exit_code`
- `artifacts`
- `log_path`

## Manager Contract

### Final tier

#### Inputs

- project docs
- current `state/*.json`
- current filesystem diff
- current task graph

#### Required outputs

- task assignment decisions
- pause/resume/re-prompt decisions
- next-step selection
- retry decision or halt decision

#### Required side effects

- spawn or resume workers through the Agent Runner
- never bypass the State Registry
- never infer completion from chat content alone

### V1 tier

#### Inputs

- project docs
- current run logs
- current `state/*.json`
- current filesystem diff

#### Required outputs

- one-shot task launch
- retry or halt decision
- artifact verification result

## Planner Contract

### Final tier

#### Inputs

- source docs under `docs/`
- Manager task prompt
- relevant repo context requested by the Manager

#### Required output artifact

- `plans/plan.md`

#### Completion requirement

The Planner session is considered complete when `plan.md` is written and its contents satisfy the task prompt.

### V1 tier

#### Inputs

- source docs under `docs/`
- one-shot prompt from the Manager

#### Required output artifact

- `plans/plan.md`

#### Completion requirement

The process exits successfully after writing the plan and producing a verifiable log.

## Builder Contract

### Final tier

#### Inputs

- `plans/plan.md`
- current repo diff
- specific plan step assigned by the Manager
- any review feedback or retry notes

#### Required output artifact

- code changes in the repository

#### Completion requirement

The Builder session is considered complete when the requested code changes exist in the filesystem and the Manager can verify them through diff inspection.

### V1 tier

#### Inputs

- `plans/plan.md`
- one build step prompt

#### Required output artifact

- code changes in the repository

#### Completion requirement

The process exits successfully after the changes are written and the diff is visible.

## Tester Contract

### Final tier

#### Inputs

- code diff or repo snapshot
- expected test scope
- current build result

#### Required output artifact

- `reports/test-report.md`

#### Completion requirement

The Tester session is considered complete when the report exists and clearly marks pass or fail.

### V1 tier

#### Inputs

- code diff or repo snapshot
- one-shot test prompt

#### Required output artifact

- `reports/test-report.md`

#### Completion requirement

The process exits successfully after the report is written.

## Custom Role Contract

### Final tier

Custom roles follow the same persistent-session rules as the core roles.

#### Required inputs

- user-defined role name
- task ID
- explicit artifact contract
- explicit file paths

#### Required outputs

- role-specific artifact(s) declared by the Manager

#### Required completion signal

- state file transition to a terminal result for the assigned task

### V1 tier

Custom roles are one-shot processes with the same prompt and artifact rules.

## Prompt Format Requirements

Every role prompt must explicitly include:

- role label
- task ID
- exact file paths to read and write
- whether the task is V1 or final
- the success condition

## Contract Invariants

- No role may choose its own output path.
- No role may omit `status.json`.
- No role may invent new stages in the task graph.
- The Manager is the only actor allowed to advance the graph.

