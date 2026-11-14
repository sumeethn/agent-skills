---
name: design-convergence
description: >-
  Produce an approved design for model inference or system-level inference
  performance work (latency, throughput, memory, kernels, batching, scheduling,
  quantization, parallelism, serving) using Orca workers: Claude measures a
  baseline, locates the bottleneck and writes the design with a performance
  contract, a read-only Codex critic challenges it (max 3 rounds), the user
  approves it, and it is published as a committed design document in a merge
  request. Use when the user asks for an inference optimization or inference
  system design converged with Codex before any code is written, or an approved
  design to hand to `phased-implementation`. Writes no production code.
  Hardware- and framework-agnostic. Requires the `orchestration` skill.
---

# Design convergence

You are the **architect (Claude)**. Codex is the **critic**, run as a read-only Orca worker. Load
the `orchestration` skill first and use its commands for every worker (`worker-start`, `send`,
`check`). Never poll, never steer through a worker's terminal, and do not build another
orchestration layer. The gates below are yours to check, not the agents'.

**This skill owns** what the system is and what "faster" means: scope, boundaries, interfaces,
behaviour, failure handling and the performance contract. **It does not own** how the code is
arranged behind those interfaces, how the work is sliced into phases, or any implementation; those
belong to `phased-implementation`.

## Performance contract

Every design defines one, and `phased-implementation` enforces it unchanged:

- **Workload**: model and checkpoint, precision, input and output length distribution, batch size or
  concurrency, and request pattern (offline batch, or online at a stated arrival rate).
- **Metrics**: only those that matter here, each with direction and target, for example time to
  first token, inter-token latency, end-to-end latency percentiles, throughput at a latency
  objective, and peak memory.
- **Quality gate**: how outputs are proven unchanged or acceptable. Exact token match under greedy
  decoding for numerically equivalent changes; a named evaluation and tolerance for lossy ones
  (quantization, speculative or approximate methods).
- **Environment manifest**: hardware model, count and interconnect, driver, runtime and library
  versions, container image digest, and clock or power settings when they matter.
- **Measurement protocol**: benchmark and evaluation commands, warmup, repetitions, the statistic
  reported (median with spread), and the noise threshold from repeated runs of the same build. A
  change counts as a gain only when it exceeds that threshold, measured on the same host.

## 0. Intake (at invocation)

Needed: problem statement, the workload and metric targets, target hardware and how it is reached,
the reference implementation or model to compare against, the quality tolerance, constraints
(compatibility, deadlines), repository and host, target branch, and where design documents live.
**Inspect first**: `AGENTS.md`, `CLAUDE.md`, `README`, `CONTRIBUTING`, existing design documents,
existing benchmark, profiling and evaluation tooling, `git remote -v`, default branch. Read the code
the design touches before designing; a design that contradicts the existing system is a blocker.
Ask the user only for what is still missing, in one batch. Do not assume any framework, hardware,
path or dependency.

Create the run record and freeze the problem statement, workload, metric targets, quality
tolerance, target hardware, target branch and the design document path. If any of these change
later, restart convergence from round 1.

## Run record

Directory `$(git rev-parse --git-common-dir)/agent-runs/<task-id>/` (inside `.git`, never
committed, shared across worktrees and with `phased-implementation`). Task ID: short, `[a-z0-9-]`
only. Files: `state.json`, `design-r<N>.md`, `design-findings.md`, `baseline/` (environment
manifest, raw benchmark and profile output, commands).

`state.json` has `"schema": 1`, `task_id`, `target_branch`, and a `design` object that this skill
owns: `phase` (`intake`, `review`, `approval`, `publish`, `published`, `abandoned` or
`blocked`), `round`, `hash`, `approved_hash`, `path`, `branch`, `commit_sha`, `mr`,
`blocked_reason`. Leave every other key untouched. Update after every step. On resume, read it,
then reconcile with real git and MR state before acting; repeat only unfinished steps.

`hash` is always the SHA-256 of the design file's exact bytes.

## 1. Baseline and bottleneck

Optimize from evidence, not intuition. If a benchmark harness and the target hardware are
available, run the protocol on the current target branch: record the environment manifest, the
baseline numbers with their spread, and a profile showing where time and memory go (kernels,
memory bandwidth, communication, host overhead, scheduling or batching, data movement). Estimate
the headroom with roofline-style reasoning: what the hardware could do for this workload versus
what it does now. Save everything under `baseline/`. These are measurements, not code changes;
commit nothing.

If no harness exists or the hardware is unavailable, say so, mark the baseline and bottleneck
**unverified**, and require the implementation plan to build the harness and measure the baseline
before any optimization phase.

## 2. Write the design

Write `design-r<N>.md`. Size it to the problem and ground it in the real code and the baseline,
citing file paths and measured numbers. Cover each of these, or say in one line why it does not
apply:

- Problem, goals and non-goals.
- The performance contract.
- Baseline and bottleneck analysis, with evidence and headroom estimate, marked verified or
  unverified.
- Hypotheses: each change, the bottleneck it addresses, its expected effect on each contracted
  metric, and how that will be measured. Order by expected value and risk.
- Alternatives considered and why the chosen ones win.
- Components, their responsibilities and the boundaries between them.
- Interfaces and data model, precise enough that two people would build compatible halves.
- Quality and numerical risk, and how the quality gate covers it.
- Failure handling: fallback paths (for example an unsupported shape or device), out-of-memory
  behaviour, and recovery.
- Operation: observability of the contracted metrics, rollout, and rollback.
- Risks and open questions.

Cut anything the problem does not need. Do not describe file layout, class structure or delivery
phases; leave them to the implementation plan.

## 3. Converge with the critic (max 3 rounds)

1. Start a **read-only** Codex critic (`worker-start --agent codex --worktree current`). The spec
   must name the exact design file, the `baseline/` directory and the code paths it may read, and
   exclude unrelated documents; forbid edits, builds, benchmark runs and network; ask for ranked
   findings (blocker/major/minor, concrete failure scenario, specific fix) judged on correctness,
   feasibility, maintainability and simplicity, and on whether the bottleneck is supported by the
   evidence, the expected gains are physically plausible, the workload is representative, the
   measurement protocol can separate a real gain from noise, and the quality gate catches the
   numerical risks; ending in APPROVED, CHANGES_REQUESTED or ESCALATE. Then end your turn; Orca
   wakes you.
2. Record each finding with a stable ID in `design-findings.md`. Revise for each valid finding;
   state which you reject and why. Increment the round and write a new `design-r<N>.md`.
3. **Agreement** = Codex verdict APPROVED on the same `hash` you hold, and you have no open
   objection. A verdict on an older revision does not count.
4. After round 3 without agreement, or on any ESCALATE, stop and give the user the alternatives,
   the disagreement and your recommendation.

## 4. User approval

On agreement, present the final design and **stop for explicit user approval**. Record
`approved_hash`. A Codex APPROVED is not user approval. Any later change to the file invalidates
approval and returns to step 3.

## 5. Publish

Run only after user approval.

1. Create a design branch from the freshly fetched target branch, named by the project's branch
   convention. If the working tree is not clean, use a separate `git worktree`; never carry
   unrelated changes.
2. Write the approved design, byte for byte, to the frozen design document path. Commit only that
   file, push the branch and open a merge request whose description states the approved hash.
3. Verify, don't trust: the pushed commit and the merge request exist, and
   `git show <commit>:<path> | sha256sum` equals `approved_hash`. Record `branch`, `commit_sha` and
   `mr`.
4. Stop. This skill does not merge. The user or a maintainer merges the design MR under the
   repository's normal policy; the design is ready for `phased-implementation` once that file,
   with that hash, is on the target branch.

Report the design path, the approved hash, the merge request, that it must be merged before
`phased-implementation` can start, and anything left open.

## Cleanup

After publishing, or when the run is abandoned: every worker message is handled and no gate is
pending; close a settled critic with `worker-release` (`worker-stop` on a settled dispatch is a
no-op and leaves a kept-open agent running) and confirm with `worker-list`. Set `design.phase` to
`published` only after verified publication, or to `abandoned` when the run is abandoned. Keep the
run record. Remove a design worktree you created only if
`git status --porcelain --ignored` is empty and its commit is pushed.

If a token-saving hook rewrites or compresses command output, re-run decisive checks (git status,
hashes) with the full binary path before acting on them.

## Hard rules

- Keep the critic read-only and every worker spec self-contained and narrowly scoped.
- Write no production code, tests or configuration; the only file this skill commits is the design.
- Report only what was actually checked. Mark unverified assumptions as unverified.
- Every performance number names the environment manifest and protocol it came from; an estimate
  is labelled as an estimate.
- No credentials, tokens or project configuration in the skill, run record, design or worker specs.
- No force-push, no hook skipping, no protection bypass, no merge, no unrelated branch changes.
- Wait for the user's explicit approval before publishing.
