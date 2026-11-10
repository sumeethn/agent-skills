---
name: design-convergence
description: >-
  Produce an approved system design using Orca workers: Claude writes the design
  (boundaries, interfaces, data model, failure handling, alternatives), a
  read-only Codex critic challenges it (max 3 rounds), the user approves it, and
  it is published as a committed design document in a merge request. Use when
  the user asks for a system design or architecture converged with Codex, a
  cross-model design review before any code is written, or an approved design to
  hand to `phased-implementation`. Writes no production code. Project-agnostic.
  Requires the `orchestration` skill.
---

# Design convergence

You are the **architect (Claude)**. Codex is the **critic**, run as a read-only Orca worker. Load
the `orchestration` skill first and use its commands for every worker (`worker-start`, `send`,
`check`). Never poll, never steer through a worker's terminal, and do not build another
orchestration layer. The gates below are yours to check, not the agents'.

**This skill owns** what the system is: scope, boundaries, interfaces, data model, behaviour and
failure handling. **It does not own** how the code is arranged behind those interfaces, how the work
is sliced into phases, or any implementation; those belong to `phased-implementation`.

## 0. Intake (at invocation)

Needed: problem statement, constraints (performance, security, compatibility, deadlines),
repository and host, target branch, and where design documents live in this repository.
**Inspect first**: `AGENTS.md`, `CLAUDE.md`, `README`, `CONTRIBUTING`, existing design documents and
their conventions, `git remote -v`, default branch. Read the code the design touches before
designing; a design that contradicts the existing system is a blocker. Ask the user only for what
is still missing, in one batch. Do not assume any framework, hardware, path or dependency.

Create the run record and freeze the problem statement, constraints, target branch and the design
document path. If any of these change later, restart convergence from round 1.

## Run record

Directory `$(git rev-parse --git-common-dir)/agent-runs/<task-id>/` (inside `.git`, never
committed, shared across worktrees and with `phased-implementation`). Task ID: short, `[a-z0-9-]`
only. Files: `state.json`, `design-r<N>.md`, `design-findings.md`.

`state.json` has `"schema": 1`, `task_id`, `target_branch`, and a `design` object that this skill
owns: `phase`, `round`, `hash`, `approved_hash`, `path`, `branch`, `commit_sha`, `mr`,
`blocked_reason`. Leave every other key untouched. Update after every step. On resume, read it,
then reconcile with real git and MR state before acting; repeat only unfinished steps.

`hash` is always the SHA-256 of the design file's exact bytes.

## 1. Write the design

Write `design-r<N>.md`. Size it to the problem and ground it in the real code, citing file paths.
Cover each of these, or say in one line why it does not apply:

- Problem, goals and non-goals.
- Constraints and assumptions, each marked as verified or unverified.
- Alternatives considered and why the chosen one wins.
- Components, their responsibilities and the boundaries between them.
- Interfaces and data model, precise enough that two people would build compatible halves.
- Failure handling, recovery and trust boundaries.
- Operation: observability, rollout, migration and rollback.
- Test strategy and acceptance criteria for the system as a whole.
- Risks and open questions.

Cut anything the problem does not need. Do not describe file layout, class structure or delivery
phases; leave them to the implementation plan.

## 2. Converge with the critic (max 3 rounds)

1. Start a **read-only** Codex critic (`worker-start --agent codex --worktree current`). The spec
   must name the exact design file and the code paths it may read, and exclude unrelated documents;
   forbid edits, builds and network; ask for ranked findings (blocker/major/minor, concrete failure
   scenario, specific fix) judged on correctness, feasibility, maintainability and simplicity,
   ending in APPROVED, CHANGES_REQUESTED or ESCALATE. Then end your turn; Orca wakes you.
2. Record each finding with a stable ID in `design-findings.md`. Revise for each valid finding;
   state which you reject and why. Increment the round and write a new `design-r<N>.md`.
3. **Agreement** = Codex verdict APPROVED on the same `hash` you hold, and you have no open
   objection. A verdict on an older revision does not count.
4. After round 3 without agreement, or on any ESCALATE, stop and give the user the alternatives,
   the disagreement and your recommendation.

## 3. User approval

On agreement, present the final design and **stop for explicit user approval**. Record
`approved_hash`. A Codex APPROVED is not user approval. Any later change to the file invalidates
approval and returns to step 2.

## 4. Publish

Run only after user approval.

1. Create a design branch from the freshly fetched target branch, named by the project's branch
   convention. If the working tree is not clean, use a separate `git worktree`; never carry
   unrelated changes.
2. Write the approved design, byte for byte, to the frozen design document path. Commit only that
   file, push the branch and open a merge request whose description states the approved hash.
3. Verify, don't trust: the pushed commit and the merge request exist, and
   `git show <commit>:<path> | sha256sum` equals `approved_hash`. Record `branch`, `commit_sha` and
   `mr`.
4. Stop. This skill does not merge. The design is ready for `phased-implementation` once that file,
   with that hash, is on the target branch.

Report the design path, the approved hash, the merge request and anything left open.

## Cleanup

After publishing, or when the run is abandoned: every worker message is handled and no gate is
pending; close a settled critic with `worker-release` (`worker-stop` on a settled dispatch is a
no-op and leaves a kept-open agent running) and confirm with `worker-list`. Set `design.phase` to
`published`. Keep the run record. Remove a design worktree you created only if
`git status --porcelain --ignored` is empty and its commit is pushed.

If a token-saving hook rewrites or compresses command output, re-run decisive checks (git status,
hashes) with the full binary path before acting on them.

## Hard rules

- Keep the critic read-only and every worker spec self-contained and narrowly scoped.
- Write no production code, tests or configuration; the only file this skill commits is the design.
- Report only what was actually checked. Mark unverified assumptions as unverified.
- No credentials, tokens or project configuration in the skill, run record, design or worker specs.
- No force-push, no hook skipping, no protection bypass, no merge, no unrelated branch changes.
- Wait for the user's explicit approval before publishing.
