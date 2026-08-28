# Prompt for Codex — Full Documentation Generation

Use this verbatim as the instruction to Codex, pasted together with `alpha-zeta-dev-force-spec.md` as attached/reference context.

---

You are generating the **complete technical documentation** for a project called Alpha-Zeta Dev Force, based on the attached seed specification (`alpha-zeta-dev-force-spec.md`).

**Audience:** This documentation will be read directly by multiple autonomous coding agents, operated by multiple different people, running in parallel on separate parts of the codebase. They will not talk to each other except through what you write. Any ambiguity, inconsistency, or unstated assumption in your output will cause different agents to implement conflicting things. Treat precision and internal consistency as the top priority — above completeness, above prose quality.

**Critical framing — read before writing anything:** The seed spec describes **two tiers**, and both must be fully documented:
- The **Final Product Architecture** (persistent, interactive CLI agents, each in its own orchestrator-owned PTY, rendered as an in-platform terminal pane) is the real target. Document it in full depth — this is not the optional part.
- The **V1 Demo Milestone** (one-shot headless CLI invocations, native OS terminal windows for visual effect) is a smaller, faster first step used to validate the Manager's decision logic before the harder PTY-based work — it is not a simplification of the final product's *goal*, only of its *execution layer*. Do not merge the two or treat V1 as "the real thing but simpler" — they have genuinely different worker-execution and state-detection mechanisms, both specified in the seed doc.

**Hard constraints:**
1. Every section in the seed doc marked **LOCKED** is a fixed architectural decision. Expand it with full implementation detail; do not alter, soften, merge, or offer alternatives to it.
2. Every section marked **OPEN** is out of scope for this document — do not invent V2 detail for it beyond a one-line "deferred" note.
3. Do not introduce new architectural decisions the seed doc didn't make (e.g. don't add a database, don't add auth, don't add GPU-capping) unless a section is genuinely unspecified and an agent could not proceed without it — in that case, make the smallest decision necessary, mark it `[DECISION ADDED]`, and justify it in one sentence.

**Required output structure** (one file per section, all in Markdown, under a `/docs` folder):

1. `01-final-architecture.md` — the complete final-product architecture: Manager, persistent worker sessions, PTY ownership, in-platform terminal-pane UI, and the pause/resume/prompt-injection control flow, expanded from Section 3 and 7 of the seed doc into precise state-transition rules.
2. `02-v1-demo-milestone.md` — the complete V1 demo architecture and its own end-to-end demo script: one-shot invocation flow, native terminal window launch (per OS), and how it maps onto the same task graph as the final product. State explicitly that this document does not replace `01`.
3. `03-agent-contracts.md` — for each role (Manager, Planner, Builder, Tester, custom), the exact prompt/session contract for **both** tiers: required inputs, required output artifact(s), file paths, and the `status.json` schema each must satisfy.
4. `04-cli-adapters.md` — the `CLIAdapter` interface in full (TypeScript signatures) with both `attachInteractive()` and `runHeadless()` methods, plus one concrete adapter spec each for Claude Code and OpenCode CLI covering both modes.
5. `05-state-registry.md` — the Agent Runner and State Registry mechanism in full implementation detail for both tiers: process/PTY spawning, output streaming, heartbeat/idle detection, marker parsing, filesystem-diff verification, and the polling contract the Manager uses.
6. `06-resource-governance.md` — the free-hardware equal-division formula and its live enforcement via Windows Job Objects / Linux-macOS cgroups v2, with exact pseudocode, explicitly noting no GPU capping is implemented.
7. `07-repo-and-ui-setup.md` — full repo scaffold file-by-file, including the in-platform terminal-pane frontend component and its wiring to each worker's PTY.
8. `08-build-plan.md` — an ordered, numbered implementation plan broken into discrete tasks small enough for one coding agent to complete independently, phased as Phase 0 (V1 demo) then Phase 1 (final architecture), each task stating its exact inputs, exact deliverable file(s), and dependencies on other tasks — so parallel agents can be assigned non-conflicting tasks.

**Style requirements:**
- No marketing language, no filler, no restating the pitch — this is an implementation spec, not a pitch deck.
- Prefer schemas, interfaces, numbered steps, and pseudocode over prose paragraphs.
- Every claim that constrains implementation must be unambiguous enough that two different coding agents reading it independently would produce compatible code.
- Keep each file focused on its single stated purpose; cross-reference other files by name rather than duplicating content.

Begin only after confirming you have the full content of `alpha-zeta-dev-force-spec.md`.
