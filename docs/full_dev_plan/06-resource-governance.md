# 06-resource-governance

## Scope

This document defines the resource governance model for both tiers.

It covers:

- equal division of currently-free CPU and RAM
- live updates when the worker count changes
- Windows Job Objects
- Linux/macOS cgroup-based enforcement
- fallback behavior

No GPU capping is implemented.

## Core Rule

When the number of active agents changes, recompute per-agent resource caps from currently-free hardware.

### Formula

```text
cap_per_agent_cpu = free_cpu_cores / active_agent_count
cap_per_agent_ram = free_ram_gb / active_agent_count
```

Where:

- `free_cpu_cores` is the currently free CPU capacity reported by the OS
- `free_ram_gb` is the currently free RAM reported by the OS
- `active_agent_count` is the number of live worker processes under orchestration

## Enforcement Requirements

### Windows

Each worker process must be placed in a Job Object.

The Job Object must enforce:

- CPU rate control
- memory limit

The configured limits must be live-updatable when the active agent count changes.

### Linux/macOS

Each worker process must be placed in a per-agent cgroup-style control boundary with:

- `cpu.max`
- `memory.max`

The limits must be live-updatable by writing new values when agent count changes.

### Fallback

If Job Objects or cgroup-style enforcement are unavailable:

- apply process priority hints only
- treat the result as best-effort
- do not claim hard enforcement

## Update Triggers

The orchestrator must recompute caps whenever:

- a worker starts
- a worker exits
- a worker is paused and fully removed from the live set
- a worker is resumed and re-added to the live set

## Pseudocode

```ts
function recomputeCaps(activeAgents: WorkerProcess[]): void {
  const freeCpuCores = os.freeCpuCores();
  const freeRamGb = os.freeRamGb();
  const count = activeAgents.length;

  if (count === 0) {
    return;
  }

  const cpuPerAgent = freeCpuCores / count;
  const ramPerAgent = freeRamGb / count;

  for (const agent of activeAgents) {
    const limits = {
      cpu: cpuPerAgent,
      ramGb: ramPerAgent,
    };

    if (hostIsWindows()) {
      applyWindowsJobObjectLimits(agent.pid, limits);
    } else if (hostSupportsCgroupStyleLimits()) {
      applyCgroupLimits(agent.controlGroupPath, limits);
    } else {
      applyBestEffortPriorityHints(agent.pid, limits);
    }
  }
}
```

## Windows Job Object Notes

The implementation must:

- create one Job Object per worker
- assign the worker process to that Job Object
- update the Job Object limits when the cap changes
- keep the manager process outside the worker Job Objects

## Linux/macOS Control Notes

The implementation must:

- create one control boundary per worker
- write the new CPU and memory limits when the cap changes
- keep the manager process outside the worker boundaries

## Non-Goals

- GPU quota management
- cross-machine scheduling
- cluster-level fairness

