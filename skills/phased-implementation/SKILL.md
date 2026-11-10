---
name: phased-implementation
description: >-
  Build an already-approved design in phases using Orca workers: Claude writes
  the software structure and a phase plan, a read-only Codex critic checks it,
  the user approves it, then for each phase Codex implements, tests, pushes a
  branch and opens an MR while Claude reviews (max 5 rounds), and the MR is
  merged automatically if every gate passes. Use when the user asks for phased
  implementation of an approved design, Codex implementation with Claude review,
  or an MR-per-phase workflow with automatic fail-closed merge. Needs an
  approved design document (for example from `design-convergence`); it does not
  create or change the design. Project-agnostic. Requires the `orchestration`
  skill.
---

# Phased implementation

You are the **software architect and reviewer (Claude)**. Codex is the **plan critic, then the
implementer**, run as Orca workers. Load the `orchestration` skill first and use its commands for
every worker (`worker-start`, `send`, `check`). Never poll, never steer through a worker's
terminal, and do not build another orchestration layer. Deterministic gates below are yours to
check, not the agents'.

**This skill owns** how an approved design is built: code structure behind the design's interfaces,
the slicing into phases, implementation, review and merge. **It does not own** the design. A change
to scope, an interface, the data model or agreed behaviour is a design change (see Changes).

## 0. Intake (at invocation)

Needed: the design document path, repository and host (e.g. GitLab project), target branch, where
code runs and tests run (local or remote, and how it is reached), build/test commands, and required
CI jobs. **Inspect first**: `AGENTS.md`, `CLAUDE.md`, `README`, `CONTRIBUTING`, CI config,
`Makefile`/package scripts, `git remote -v`, default branch. Ask the user only for what is still
missing, in one batch. Do not assume any framework, hardware, path or dependency.

**Design gate.** Fetch the target branch and compute the SHA-256 of the design file as it exists
there (`git show <target>:<path> | sha256sum`). Proceed only if the file is on the target branch
and either the run record's `design.approved_hash` equals that hash, or the user confirms, when
shown the path and hash, that this is the approved design. Otherwise stop and say what is missing.
Do not write or repair a design here.

Create or extend the run record and freeze: design path and hash, target branch and its SHA, test
commands and required CI jobs. If any of these change later, treat it as a plan change.

## Run record

Directory `$(git rev-parse --git-common-dir)/agent-runs/<task-id>/` (inside `.git`, never
committed, shared across worktrees and with `design-convergence`). Task ID: short, `[a-z0-9-]`
only; reuse the design's task ID when its run record exists. Files: `state.json`, `plan-r<N>.md`,
`findings.md`, `evidence/<phase-id>/` (test logs, CI links).

`state.json` has `"schema": 1`, `task_id`, `target_branch`, and an `implementation` object that
this skill owns: `phase` (`intake`, `plan`, `build`, `complete` or `blocked`), `design_path`,
`design_hash`, `plan_round`, `plan_hash`, `approved_plan_hash`, `current`, `blocked_reason`, and
`phases`, a list with one entry per phase: `id`, `status` (`pending`, `active`, `merged` or
`blocked`), `branch`, `target_sha`, `head_sha`, `tested_sha`, `reviewed_sha`, `signed_off_sha`,
`mr`, `review_round`, `open_findings`. Leave every other key untouched. Update after every step. On
resume, read it, then reconcile with real git and MR state before acting; repeat only unfinished
steps. Evidence is tied to a SHA and is void when the head SHA changes.

## 1. Plan (max 2 rounds)

1. Write `plan-r<N>.md` with two parts.
   - **Structure**: the modules or files to add or change, what each is responsible for, and which
     may depend on which. It must realise the design's interfaces exactly, without adding to them.
   - **Phases**: an ordered list. Each phase has an ID (`[a-z0-9-]`), scope, allowed files,
     acceptance criteria, the tests that prove them, and the earlier phases it depends on. Each
     phase is one merge request that leaves the target branch building and passing on its own.
     Prefer few phases; one is fine. Put the riskiest assumption in the earliest phase that can
     test it.
2. Start a **read-only** Codex critic (`worker-start --agent codex --worktree current`). The spec
   must name the plan file, the design file and the code paths it may read; forbid edits, builds
   and network; ask for ranked findings (blocker/major/minor, concrete failure scenario, specific
   fix) on whether the plan covers the whole design, each phase is independently mergeable and
   testable, the order is sound and nothing exceeds the design, ending in APPROVED,
   CHANGES_REQUESTED or ESCALATE. Then end your turn; Orca wakes you.
3. Revise for each valid finding; state which you reject and why. **Agreement** = APPROVED on the
   `plan_hash` you hold. After round 2 without agreement, or on any ESCALATE, stop and give the
   user the disagreement and your recommendation.
4. Present the plan and **stop for explicit user approval**. Record `approved_plan_hash` and create
   the `phases` entries. Do not start a phase without it.

## 2. Build each phase, in order (max 5 review rounds per phase)

Run one phase at a time. Start a phase only when every phase it depends on is `merged`.

1. Fetch the target branch and record `target_sha`. Start a Codex implementer in its own branch
   worktree (`worker-start --agent codex --worktree new-top-level --repo id:<id> --name <branch>`),
   with the branch named by the project's convention. The spec must be self-contained: the design
   path, this phase's section of the plan, the plan's structure, build/test commands and
   environment, allowed files, branch name, and what to report. It must: implement only this
   phase, run the tests, push the branch, and open the MR with the **actual** test commands and
   results. It must never claim a test that did not run, and must report an environment blocker
   rather than guess around it.
2. Verify, don't trust: confirm the pushed SHA, branch and MR exist; read the evidence.
3. Review as Claude, against the exact head SHA, for correctness against the design, this phase's
   acceptance criteria and the plan's structure, and for tests, safety and simplicity. Record
   findings with stable IDs (`<phase-id>-<n>`) in `findings.md`. Blocking findings become MR
   discussions when the host supports it.
4. Send blocking findings to the implementer with `send`. It fixes, reruns affected tests, pushes,
   and reports the new SHA and results. Review that SHA. A prior finding counts as resolved only
   when you explicitly dispose of it.
5. After 5 unsuccessful rounds, stop and escalate to the user with the open findings.
6. **Sign-off.** When you have no open blocking finding on a head SHA, ask the implementer to
   confirm for that exact SHA: it is its final push, nothing is unpushed, and it knows of no
   unreported failing test or deviation from the design or plan. Record `signed_off_sha`. A
   confirmation for an older SHA does not count.
7. Merge (see Phase completion), clean up the phase, mark it `merged`, then check the remaining
   plan against what was actually merged before starting the next phase.

## Changes

- **Design change** (scope, an interface, the data model, agreed behaviour): stop the implementer,
  set `phase` to `blocked` with the reason, and tell the user exactly what in the design must
  change. The design is revised and re-approved outside this skill (`design-convergence`). Resume
  only through the design gate with the new hash, then re-plan the unmerged phases.
- **Plan change** (phase boundaries, order, acceptance criteria, structure, frozen commands): revise
  the plan for the unmerged phases, run one critic round, and get user approval again. Merged
  phases stay as they are.
- Anything else is ordinary review within the phase.

## Phase completion (automatic, fail closed)

Merge only if all hold at the moment of merging, re-checked from live state:
- Claude approved and the implementer signed off on the **same latest head SHA**; no open blocking
  findings.
- Required tests passed on that SHA; CI succeeded for that SHA with every required job passing
  (a pipeline merely existing is not success; missing, pending, failed, skipped or stale is not).
- Target branch freshly fetched: if it moved since testing/review, integrate it, then repeat
  tests, CI, review and sign-off on the new head. Do not merge stale evidence.
- The design file on the target branch still has `design_hash`. If not, treat it as a design
  change.
- The host reports the MR mergeable under repository policy. Merge with a SHA-conditional merge
  (check the host CLI's help for the flag; do not guess) and **without** the delete-branch option.
  If the CLI defaults to auto-merge or merge-when-pipeline-succeeds, disable it so the merge is
  immediate and verifiable.
- Do not ask for separate human merge approval and never call an approval API or bypass
  protection. If policy requires approvals or blocks the merge, stop, report the exact policy
  message, and mark the phase and the run blocked.

After any merge call, fetch the MR and verify state is merged and the merge matches the expected
SHA, retrying only after a fresh GET. Then delete the source branch only if the merge is verified
and the branch head still equals the expected SHA; confirm it is gone. Mark the phase `merged` only
when merge and deletion are both verified, and tell the user in one line which phase merged.

## Cleanup

**After each merged phase**, never after a blocked or escalated one:

1. **Close the phase's workers**: `worker-stop` on an already-settled dispatch is a no-op
   (`alreadySettled`) and leaves a kept-open agent running. Close a settled worker with
   `worker-release` (archives its transcript). Use `worker-stop` only for an unsettled worker.
2. **Remove the implementer worktree only after proving it is disposable**, in its own checkout:
   fetch the target branch, then require all of: empty `git status --porcelain --ignored`, no
   stash, `HEAD` is an ancestor of the freshly fetched target branch, and zero commits outside it.
   Then `orca worktree rm --worktree path:<path>` **without** `--force`. Any failed check means keep
   it and report. Do not delete the local branch unless the user asked (`--delete-branch`).

**After the last phase is merged**:

3. **Reconcile messages**: `orchestration inbox --terminal run:<run_id>`; every `worker_done`,
   question and escalation must already be handled. `gate-list` must be empty.
4. **Close out tasks and run**: `task-list` shows every task `completed`; fix any that are not with
   `task-update --status completed` only if its outcome truly succeeded. Orca has no run-complete
   command: the run is complete when all tasks are completed, no gates are pending and no worker is
   live. Confirm with `worker-list` that every worker is `exited` and `released`. Set `phase` to
   `complete` in `state.json`.
5. **Preserve**: the run record and `evidence/` (under `.git`), remote environments, caches and
   downloaded model weights. Report what was kept and what was removed.

If a token-saving hook rewrites or compresses command output, re-run decisive checks (git status,
merge help) with the full binary path before acting on them.

## Hard rules

- Keep the Codex critic read-only and every worker spec self-contained and narrowly scoped.
- Never edit the design document; a needed change stops the run.
- One phase, one branch, one MR. No phase starts before its dependencies are merged.
- Report only what was actually run. Mocked results are labelled as mocked.
- No credentials, tokens, or project config in the skill, run record, MR text or worker specs.
- No force-push, no hook skipping, no protection bypass, no unrelated branch changes.
- Do not add features, abstractions, or flags beyond the approved design and plan.
- Wait for the user's explicit plan approval; a Codex APPROVED is not user approval.
