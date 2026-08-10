<!--
One file per milestone: work/<milestone-id>/milestone.md. Milestone IDs are
`M0`, `M1`, ... — sequential, distinct in shape from task IDs
(`<YYYYMMDD>-<kebab-slug>`) specifically so `ledger scan-untracked-ratio`'s
commit-trailer grep is never confused by a milestone folder: a milestone
never ships its own commit, its member tasks do, each carrying its own
ordinary `Spine-Task: <task-id>` trailer. `M0` is reserved, always, for the
walking skeleton — design-gate's stopping-rule check #1 looks for
`work/M0/milestone.md` by exactly that path, not by inspecting content to
guess which milestone is "the first one."

`/task <description> --milestone <milestone-id>` loads this file into
planning context (core/skills/task/SKILL.md); member tasks otherwise plan,
get approved, implement, verify, and ship exactly as any task does — this
file is not a second approval gate, the human still approves each member
task's own plan. `/ship` on the *final* member task additionally checks
this file's `## Done-definition` before completing.
-->

# Milestone `<milestone-id>`: <title>

## Member tasks

<!-- Ordered — task N may assume task N-1's own Inter-task contracts
     entry held. Each is a real task-id once /task has created it, or
     `TBD` before that. -->

1. `<task-id | TBD>` — <one line: what it covers>
2. `<task-id | TBD>` — <one line>

## Inter-task contracts

<!-- What task N may assume task N-1 left true — an in-repo, sequential
     handoff between this milestone's own member tasks. Not the same thing
     as a cross-repo contract (Extension B's ws/contracts/<name>/) — those
     are between repos, these are between tasks in one repo. -->

## Capability targets

<!-- Required for M0 (design-gate check #1 reads this section, not just
     this file's existence): which capabilities must reach `implemented`
     in `.spine/capabilities.json` by the time this milestone ships, and
     their currently planned status. For M0 specifically, the walking-
     skeleton definition-of-done names `test`, `smoke-seed`, `smoke-run`,
     `smoke-golden` as the floor — list each with its planned status here,
     even if that status is currently `unavailable (no code yet — skeleton
     target)`. -->

| Capability | Planned status |
|---|---|
| test | |
| smoke-seed | |
| smoke-run | |
| smoke-golden | |

## Done-definition

<!-- Mechanical: what must be true for /ship, on the final member task, to
     consider this milestone complete. For M0: the four capabilities above
     conformance-passing and flipped to `implemented` in
     `.spine/capabilities.json` — the skeleton is done when the floor
     under it is real, not when the last line of code is written. -->
