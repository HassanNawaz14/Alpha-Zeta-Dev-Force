# 04-cli-adapters

## Scope

This document defines the `CLIAdapter` interface and the required adapter behavior for:

- Claude Code
- OpenCode CLI

Both adapters must support:

- final-tier interactive PTY attachment
- V1-tier one-shot headless execution

## Core Interface

```ts
export type AgentRole = "manager" | "planner" | "builder" | "tester" | string;

export type AdapterMode = "interactive" | "headless";

export interface AttachInteractiveOptions {
  agentId: string;
  role: AgentRole;
  taskId: string;
  cwd: string;
  env: Record<string, string>;
  ptyCols: number;
  ptyRows: number;
  prompt: string;
  onData(chunk: string): void;
  onExit(code: number | null, signal: string | null): void;
}

export interface RunHeadlessOptions {
  agentId: string;
  role: AgentRole;
  taskId: string;
  cwd: string;
  env: Record<string, string>;
  promptFilePath: string;
  logPath: string;
}

export interface AdapterRunResult {
  pid: number;
  exitCode: number | null;
  signal: string | null;
  logPath: string;
}

export interface CLIAdapter {
  readonly name: string;
  attachInteractive(options: AttachInteractiveOptions): Promise<{
    pid: number;
    send(input: string): void;
    resize(cols: number, rows: number): void;
    interrupt(): void;
    dispose(): Promise<void>;
  }>;
  runHeadless(options: RunHeadlessOptions): Promise<AdapterRunResult>;
}
```

## Interface Rules

### `attachInteractive()`

This method is required for the final architecture.

It must:

- start the CLI in native interactive mode
- bind the process to the orchestrator-owned PTY
- forward PTY output to the caller
- accept injected input for later prompts and interrupts

### `runHeadless()`

This method is required for V1.

It must:

- launch the CLI once
- read the prompt from `promptFilePath`
- stream logs to `logPath`
- return exit metadata when the process ends

## Shared Adapter Responsibilities

Every adapter must:

- resolve the executable path
- build the command line
- set the working directory
- pass environment variables
- normalize exit status
- keep prompt content out of shell strings when a temp-file input path is available

## Claude Code Adapter

### Interactive mode

The Claude Code adapter must:

- launch Claude Code in its supported interactive CLI mode
- connect that process to the PTY returned by the orchestrator
- keep the session alive across multiple prompts
- preserve session context within the same worker identity

### Headless mode

The Claude Code adapter must:

- launch the one-shot process for V1
- read prompt content from a temporary file
- write logs to the configured run log
- surface the final exit code without reinterpretation

### Adapter-specific notes

- command construction must be isolated in the adapter, not the Manager
- the Manager supplies only task context and file paths

## OpenCode CLI Adapter

### Interactive mode

The OpenCode adapter must:

- launch OpenCode in interactive CLI mode
- attach the process to the PTY owned by the Agent Runner
- support pause, resume, and re-prompt by PTY input injection

### Headless mode

The OpenCode adapter must:

- launch the one-shot V1 process
- consume prompt input from the temp file path
- emit a machine-readable log and exit code

### Adapter-specific notes

- output parsing must not rely on exact user-facing phrasing
- the adapter may normalize known success/failure patterns, but only as a supplement to exit code and filesystem diff

## Mode Selection

The Manager never chooses CLI-specific flags directly.

Instead:

- the Manager requests `interactive` for final product workers
- the Manager requests `headless` for V1 workers
- the adapter owns the exact CLI invocation

## Error Handling

The adapter must treat these as hard failures:

- missing executable
- unsupported mode
- PTY allocation failure
- prompt-file read failure
- unexpected process exit before state confirmation

