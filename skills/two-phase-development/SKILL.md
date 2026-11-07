---
name: two-phase-development
description: >-
  Run a development task as two gated phases using Orca workers: Claude and
  Codex converge on a design (max 3 rounds, then user approval), then Codex
  implements, tests, pushes a branch and opens an MR while Claude reviews (max
  5 rounds), then the MR is merged automatically if every gate passes. Use when
  the user asks for "two-phase development", design convergence with Codex
  before coding, Codex implementation with Claude review, or an MR workflow with
  automatic merge. Project-agnostic: it collects repo, environment, test
  commands and acceptance criteria at invocation. Requires the `orchestration`
  skill.
---

# Two-phase development

You are the **architect and reviewer (Claude)**. Codex is the **critic, then implementer**, run as
Orca workers. Load the `orchestration` skill first and use its commands for every worker
(`worker-start`, `send`, `check`). Never poll, never steer through a worker's terminal, and do not
build another orchestration layer. Deterministic gates below are yours to check, not the agents'.

## 0. Intake (at invocation)

Needed: task statement, repository and host (e.g. GitLab project), target branch, where code runs
and tests run (local or remote, and how it is reached), build/test commands, required CI jobs, and
acceptance criteria. **Inspect first**: `AGENTS.md`, `CLAUDE.md`, `README`, `CONTRIBUTING`, CI
config, `Makefile`/package scripts, `git remote -v`, default branch. Ask the user only for what is
still missing, in one batch. Do not assume any framework, hardware, path or dependency.

Create the run record (see Run record) and freeze: task text, target branch and its SHA, the test
commands, required CI jobs, acceptance criteria. If any of these change later, treat it as a
design change (return to Phase 1).

## Run record

Directory `$(git rev-parse --git-common-dir)/two-phase/<task-id>/` (inside `.git`, never committed,
shared across worktrees). Task ID: short, `[a-z0-9-]` only.
Files: `state.json`, `design-r<N>.md`, `findings.md`, `evidence/` (test logs, CI links).
`state.json` keys: `phase`, `design_rev`, `design_hash`, `approved_hash`, `design_round`,
`review_round`, `target_sha`, `branch`, `head_sha`, `tested_sha`, `reviewed_sha`, `mr`,
`open_findings`, `blocked_reason`. Update after every step. On resume, read it, then reconcile with
real git and MR state before acting; repeat only unfinished steps. Evidence is tied to a SHA and is
void when the head SHA changes.

## Phase 1: design convergence (max 3 rounds)

1. Write the design as `design-r<N>.md`: scope and non-goals, approach, interfaces, failure
   handling, test plan, open questions. Keep it minimal; cut anything the task does not need.
2. Start a **read-only** Codex critic (`worker-start --agent codex --worktree current`). Spec must
   name the exact file(s) to review and exclude unrelated docs; forbid edits, builds and network;
   ask for ranked findings (blocker/major/minor, concrete failure scenario, specific fix) judged on
   correctness, feasibility, maintainability and simplicity, ending in APPROVED, CHANGES_REQUESTED
   or ESCALATE. Then end your turn; Orca wakes you.
3. Revise for each valid finding; state which you reject and why. Increment the round.
4. **Agreement** = Codex verdict APPROVED on the same `design_hash` you hold, and you have no open
   objection. A verdict on an older revision does not count.
5. After round 3 without agreement, or on any ESCALATE, stop and give the user the alternatives,
   the disagreement, and your recommendation.
6. On agreement, present the final design and **stop for explicit user approval**. Record
   `approved_hash`. Do not start implementation without it. Any later change to scope or design
   invalidates approval.

## Phase 2: implementation and review (max 5 review rounds)

1. Start a Codex implementer in its own branch worktree
   (`worker-start --agent codex --worktree new-top-level --repo id:<id> --name <branch>`). Spec must
   be self-contained: approved design, build/test commands and environment, allowed files, branch
   name, and what to report. It must: implement, run the tests, push the feature branch, and open the
   MR with the **actual** test commands and results. It must never claim a test that did not run,
   and must report an environment blocker rather than guess around it.
2. Verify, don't trust: confirm the pushed SHA, branch, and MR exist; read the evidence.
3. Review as Claude, against the exact head SHA, for correctness against the approved design and
   acceptance criteria, tests, safety, and simplicity. Record findings with stable IDs in
   `findings.md`. Blocking findings become MR discussions when the host supports it.
4. Send blocking findings to the implementer with `send`. It fixes, reruns affected tests, pushes,
   and reports the new SHA and results. Review that SHA. A prior finding counts as resolved only
   when you explicitly dispose of it.
5. After 5 unsuccessful rounds, stop and escalate to the user with the open findings.
6. A **significant design change** (scope, interfaces, approach) returns to Phase 1: stop the
   implementer, revise the design, repeat convergence and user approval, then resume.

## Completion (automatic, fail closed)

Merge only if all hold at the moment of merging, re-checked from live state:
- Claude approved and Codex agrees on the **same latest head SHA**; no open blocking findings.
- Required tests passed on that SHA; CI succeeded for that SHA with every required job passing
  (a pipeline merely existing is not success; missing, pending, failed, skipped or stale is not).
- Target branch freshly fetched: if it moved since testing/review, integrate it, then repeat
  tests, CI and review on the new head. Do not merge stale evidence.
- The host reports the MR mergeable under repository policy. Merge with a SHA-conditional merge
  (check the host CLI's help for the flag; do not guess) and **without** the delete-branch option.
- Do not ask for separate human merge approval and never call an approval API or bypass
  protection. If policy requires approvals or blocks the merge, stop, report the exact policy
  message, and mark the run blocked.

After any merge call, fetch the MR and verify state is merged and the merge matches the expected
SHA, retrying only after a fresh GET. Then delete the source branch only if the merge is verified
and the branch head still equals the expected SHA; confirm it is gone. Report done only when merge
and deletion are both verified.

## Hard rules

- Keep Codex critic read-only and every worker spec self-contained and narrowly scoped.
- Report only what was actually run. Mocked results are labelled as mocked.
- No credentials, tokens, or project config in the skill, run record, MR text or worker specs.
- No force-push, no hook skipping, no protection bypass, no unrelated branch changes.
- Do not add features, abstractions, or flags beyond the approved design.
- Wait for the user's explicit design approval; a Codex APPROVED is not user approval.
