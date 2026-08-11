# Extension C, Phase A — primitive verification, reconciliation, build plan

Build prompt: `spine/work/.build/build-prompt-ext-c.md` (not yet written to
disk as a separate file — this handoff is the record; the prompt text
lives in the session that kicked this off). Scope: §0 primitive
verification, §1–2 read against actual code, reconciliation with
`README.md`/`docs/tradeoffs.md`, and a build plan for Phases B–D.

## State found at session start

Extensions A (`/design`, decision store, milestones) and B (`workspace.json`,
contracts, staged multi-repo ship) are **both already installed** in this
`spine/` checkout — `git log` shows both merged (`2376be1`, `5c4d9ad`,
`d1736e3`), and `work/.build/ext-phase-{A..E}-handoff.md` document their own
five-phase build. This build prompt's "may or may not be installed" framing
is conservative language for *other* projects, not this repo. One
uncommitted change was already sitting in the working tree at session
start: an addition to `docs/tradeoffs.md`'s classifier-wall section (a
prior, unfinished pass at this same primitive-verification work, narrowing
the wall's scope to nested `claude -p` invocation specifically, from
inside any session's own Bash tool). Left as-is, uncommitted, per "only
commit when asked" — it's correct content, not scratch to discard, and
Phase D's commit step should pick it up along with everything else.

## §0 primitive verification — findings

**0.1 Install mechanism + reproduction.** Read `README.md` and
`core/skills/bootstrap/SKILL.md` §5 / `core/skills/adopt/SKILL.md` §5
directly (not from memory). Confirmed mechanism: `.claude/skills/<name>`,
`.claude/agents/<name>.md`, `.claude/rules/<name>.md` are per-entry
symlinks to `<spine-checkout-absolute-path>/core/...`; `.claude/hooks` is
one whole-directory symlink to `<spine>/core/hooks`. All are **committed**
(git stores the literal absolute target string in the symlink blob). A
project's committed `.claude/settings.json` additionally carries a
`permissions.allow` entry `Bash(/Users/michaelbart/spine/core/scripts/*:*)`
— also an absolute path baked in at install time, also committed.

Reproduced the breakage rather than trusting the tradeoffs doc's existing
"known cost" note: bundled `~/bgr`'s real git history
(`git bundle create`), ran it inside a fresh `debian:bookworm-slim` Docker
container as a new user `alice` (`/home/alice`, no relation to
`/Users/michaelbart`) with **no spine clone present at all** — the "second
engineer's first checkout, before running any per-machine step" case.
Result, exactly as predicted:

```
readlink .claude/skills/task  →  /Users/michaelbart/spine/core/skills/task
test -e .claude/skills/task   →  BROKEN_DANGLING_SYMLINK
readlink .claude/hooks        →  /Users/michaelbart/spine/core/hooks
test -e .claude/hooks         →  HOOKS_BROKEN
```

Every skill, agent, rule, and all three hooks are simultaneously dangling.
Per the README's own "if something feels like it's fighting you" framing
and this build prompt's §2.1 opening line ("a hook that silently doesn't
fire is spine's worst failure shape"): a hook resolved through a dangling
symlink doesn't error loudly in Claude Code's hook runner in every case —
`${CLAUDE_PROJECT_DIR}/.claude/hooks/phase-gate` pointing at nothing is
exactly the silent-non-fire shape §1 names as the thing to fix, not a
crash that gets noticed. This is the concrete case the whole extension's
install-first ordering (§2.1 "first, because everything else assumes hooks
fire everywhere") is protecting against, now confirmed instead of assumed.

**0.2 Git identity in hook/skill contexts.** Same container, before any
global git config: `git config user.name` / `user.email` both empty,
non-zero exit. On the real machine, both are set globally
(`michaelbart` / `michaeljbart@me.com`) and every repo (bgr checked
directly) inherits them with no per-repo override. Conclusion for the
ownership primitive (§2.2, §0.4): **`git config user.name`/`user.email`
(local-then-global, git's own resolution order) is the identity source —
correct, because it's the same identity that already authors every commit
this system produces, so "task owner" and "commit author" stay the same
fact read two ways, not two systems that can disagree.** It is not
universally available — a genuinely fresh machine/container has none. Any
script that needs it (registry writes, `claims-check`, `ledger init`) must
check for empty and **fail loud with a specific instruction** ("run `git
config --global user.name '<you>'` — spine's ownership model reads this,
it does not invent one"), never fall back to `$USER`, hostname, or any
other guess. This is the same "distinguish could-not-run from ran-and-
failed" discipline `core/skills/task/SKILL.md` already applies to script
invocations generally — identity resolution gets the same treatment, not
a new pattern.

**0.3 Hook JSON fields + cross-workspace task-folder reads.** Grepped all
three hooks directly: the PreToolUse payload fields actually consumed are
`.tool_name`, `.tool_input.file_path`, `.tool_input.command` — nothing
else. No hook changes are needed for this extension (the manifest already
says "no new hooks"; confirmed nothing here forces one). Read
`core/skills/ship/SKILL.md` in full: task folders already live at
**`work/<task-id>/` under the workspace root** uniformly — a multi-repo
task never had a per-repo task folder to begin with (`workspace_route`'s
own header: "task_dir is always $project/work/$task_id ... never a per-
repo one"). This means §2.2's "shared registry = the directory holding
`work/`" is not a new invariant to build, it's the existing one — B's own
design already put every task folder in exactly one place. `/ship`-side
reads across the workspace (§0.3's stated worry) were already a non-issue
before this extension started.

**0.4 Collision check with A/B.**
- **Decision store is not A-exclusive.** Read `core/templates/decision.md`'s
  own header: two entry paths write into `docs/decisions/` — `/design`
  (A) and plain `/ship` (base system, distilled from a resolved
  deviation), described as coequal, present since before A existed.
  `check-stale`'s `grounding-decisions:` handling is likewise base
  infrastructure, not gated on A being installed. **Correction to the
  build prompt's framing**: `propagate`'s decision-grounding branch does
  not need an A-presence check at all — it degrades to nothing only
  because a project with zero `docs/decisions/` records has nothing to
  flag, not because A is absent. What genuinely is A-exclusive:
  `/design`, `design-gate`, `docs/decisions/DEFERRED.md`, milestones
  (`work/<id>/milestone.md`). `propagate`'s contract-spec branch is
  correctly B-exclusive (`contract-touch` hard-requires `workspace.json`,
  confirmed by reading its usage guard).
- **Live test fixtures already exist for both presence and absence**:
  `~/bgr` (single-repo, no `workspace.json`, spine installed, real
  `docs/decisions/`-free history — B absent, A's decision store
  vacuous-but-present) and this `spine/` repo's own worked examples
  under `bookmarks`/`bookmarks-cli`/`bookmarks-workspace` (referenced in
  `docs/tradeoffs.md` — B present). No fixture needs to be built from
  scratch for degradation testing; Phase D's demo should use real ones.

## Reconciliation with README.md / docs/tradeoffs.md

Read both in full (not summarized from memory). No factual contradiction
found between this build prompt and either document on any existing-spine
claim — the install mechanism, the symlink set, the hook JSON schema, and
the task-folder location all matched what the prompt assumes. Two
**framing** corrections recorded above (0.4) are not contradictions of
fact, just precision the build prompt's own language invited getting
wrong (A-exclusivity of the decision store). Both will be written into
`docs/tradeoffs.md`'s "Extension build, Phase A" note at Phase D's
documentation pass, not now — Phase A's job per the output protocol is to
record them, not edit the tradeoffs doc into its final shape prematurely
while B/C/D might still change what needs saying.

One real design correction, not just framing: **§3's deliverable manifest
says `core/templates/state extended: owner, claims, flags,
acknowledgments`, but `core/templates/state` does not exist** — `state` is
a bare one-line phase-name file, read literally as one line by
`phase-gate` (`tr -d '[:space:]' < "$current_task_file"` pattern, same
convention elsewhere) and by every skill that does `state=$(cat
work/<id>/state)`-style reads. Cramming owner/claims/flags into that file
would break every existing reader that assumes "one line, the phase name,
nothing else." **Decision: sibling files, not a repurposed `state`.**
`work/<task-id>/owner` (one line, git identity, written once at task
creation), `work/<task-id>/claims.json` (structured, written at plan
approval), `work/<task-id>/flags.json` (array, written by `propagate`,
appended to by acknowledgment) — `state` itself stays exactly what it is
today. This preserves the zero-behavioral-change guarantee for every
existing script that reads `state` and keeps the new data mechanically
greppable/jq-able rather than shoehorned into a format that was never
meant to hold it. Recorded here as a build decision to defend in
`docs/tradeoffs.md`, not silently drifted from later.

## Build plan for Phases B–D

**Phase B** — `core/scripts/setup` (the per-machine step): resolve where
this `spine/` checkout actually lives (read `SPINE_HOME` if set, else
walk up from the running script's own path — the same self-location trick
`core/hooks/*` already use via `${BASH_SOURCE[0]}`), then (re)write this
project's `.claude/skills/*`, `.claude/agents/*`, `.claude/rules/*`,
`.claude/hooks` symlinks and the machine-local Bash-allow pattern. The
architecture change from today: those symlinks and the absolute-path
`permissions.allow` entry **stop being committed** — `bootstrap`/`adopt`
stop writing them into the setup commit; `setup` becomes the thing that
creates them locally, gitignored, on every machine including the first.
`.claude/settings.json` keeps only the already-project-relative hook
wiring (`${CLAUDE_PROJECT_DIR}/...`, confirmed portable already) and the
`.spine/adapters/*` allow patterns (already relative). Acceptance: rerun
the exact Docker reproduction above, but with a `git clone` of the *spine*
bundle into the container too, at a path that does **not** match
`/Users/michaelbart/spine`, then run `spine/core/scripts/setup --project
<bgr-clone>` and re-check `test -e .claude/hooks` — must flip from
BROKEN to resolving, with zero manual path edits. Then extend
`work/<task-id>/owner` + `.spine/`'s registry pieces, and build `/tasks`
(read-only) against real task folders in bgr and the bookmarks workspace.

**Phase C** — `claims-check` (read `research.md`'s `files:`/
`grounding-decisions:` header plus `plan.md`'s `## Predicted touch`
verbatim-parsed section, same parse convention `conformance` already
uses, to build one task's claim set; intersect against every other open
task's `claims.json` under the shared `work/`; write-write and
write-read-of-prediction block, read-read warns only), `propagate`
(scan open tasks' `claims.json` for grounding on a changed path/decision/
contract, write `flags.json` entries), ship-time re-grounding
(`check-stale` + `floor` re-run already exist as scripts — this phase
wires two new call sites into `core/skills/ship/SKILL.md` §1, doesn't
build new mechanism), second-approver check (`/ship` reads an
`approver` field this phase adds to the plan-approval record and compares
git identity against `owner`), ledger `engineer` field + `/costs`
per-engineer view (extend `ledger init`'s jq skeleton and `ledger
aggregate`'s jq reduce, both read above and both trivial additions to an
existing shape). Each gets a real invocation shown in that phase's
handoff, per the output protocol — not narrated.

**Phase D** — the two-engineer demo per §6, run against two real
checkouts + two Docker-simulated git identities as the build prompt
requires (not two directories sharing one filesystem identity — confirmed
above that git identity is the primitive, so the demo must actually vary
it, which means two containers or two `--project` trees with distinct
`git config user.name`, not just two terminal tabs as the same
`michaelbart`); solo-regression proof re-running bgr's existing
zero-diff-tree floor check (already the precedent from bgr's own install,
per memory of that session) to show byte-identical behavior; tradeoffs
additions per §5; self-red-team per the four named laziest-defeat cases.

## What Phase B needs from this handoff

- The exact symlink/settings.json shape to stop committing (0.1, listed
  above) and the exact Docker reproduction command shape to reuse for the
  acceptance test (bundle + fresh container + distinct user, not same-
  machine relocation, which doesn't actually break the absolute path).
- The sibling-files decision for task state (owner/claims.json/flags.json,
  not a repurposed `state`) — Phase B writes `owner` at task creation
  (`core/skills/task/SKILL.md` step 1, alongside the existing `class`/
  `state` writes), Phase C writes `claims.json`/`flags.json`.
- Confirmed: task folders are already workspace-root-uniform, so the
  "shared mainline" open question (build prompt §2.2) reduces to "does
  `work/` get committed at task-creation time instead of at ship time" —
  a process change in `core/skills/task/SKILL.md` step 1 (commit+push the
  new task folder there), not a location change.
- bgr and the bookmarks workspace repos as ready-made B-absent/B-present
  fixtures — no synthetic project needed for Phase D unless a genuine
  claims-conflict scenario needs two tasks invented on top of one of them
  (likely bgr, since it's smaller and has no pre-existing task history to
  route around).
