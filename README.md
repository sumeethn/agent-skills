# agent-skills

Claude Code skills for model inference and system-level inference performance work (latency,
throughput, memory, kernels, batching, scheduling, quantization, parallelism, serving). They are
hardware- and framework-agnostic, and built on Orca orchestration (the `orchestration` skill)
without adding an orchestration framework of their own.

| Skill | Produces | Human gate |
| --- | --- | --- |
| [`design-convergence`](skills/design-convergence/SKILL.md) | An approved design grounded in a measured baseline and bottleneck analysis, with a performance contract, committed as a document in a merge request. Claude writes it, a read-only Codex critic challenges it. | Design approval |
| [`phased-implementation`](skills/phased-implementation/SKILL.md) | One merged MR per phase of an approved design. Claude plans and reviews, Codex implements and measures each phase with an A/B benchmark on the target hardware; each merge is automatic and fail closed. | Plan approval |

The **performance contract** in the design fixes the workload, the metrics and their targets, the
quality gate that proves outputs are unchanged or acceptable, the environment manifest and the
measurement protocol. `phased-implementation` enforces it: a phase merges only if quality holds,
no guardrail metric regresses against its base or the original baseline, the phase's minimum gain
passes in a paired benchmark on the same hosts, and on the last phase every goal is met. A gain
that does not materialize stops the run for a decision rather than being coded around.

The handoff between them is the design document on the target branch, identified by its SHA-256.
`phased-implementation` never edits the design; a needed design change stops it and goes back
through `design-convergence`. Either skill can be used alone.

Project-specific settings, credentials and run history are not stored here. Run state lives in each
project under `$(git rev-parse --git-common-dir)/agent-runs/`.

## Install (user scope)

```bash
git clone https://github.com/sumeethn/agent-skills ~/src/agent-skills
mkdir -p ~/.claude/skills
for skill in design-convergence phased-implementation; do
  ln -s ~/src/agent-skills/skills/$skill ~/.claude/skills/$skill
done
```

Update with `git -C ~/src/agent-skills pull`.

## Invoke

In Claude Code inside any project repository, with Orca running:

```
/design-convergence <problem statement>
/phased-implementation <path to the approved design document>
```

`design-convergence` opens the design MR but never merges it. Merge it under your repository's
normal policy before running `phased-implementation`, which only starts from a design that is on
the target branch.

Each skill inspects the project's instructions first and asks only for what is missing.

## Migrating from `two-phase-development`

That skill is replaced by the two above. Remove its link with
`rm ~/.claude/skills/two-phase-development` and install the new ones. Run records under
`.git/two-phase/` are not read by the new skills; finish or abandon those runs first.
