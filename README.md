# spine

An AI-assisted development workflow with teeth: research → plan → human
approval → implement → deterministic floor → adversarial review → ship.
Every gate is a real script or a real hook, not a prompt asking an agent to
please be careful. See `core/ADAPTER-CONTRACT.md` for the contract
everything else is built on, and `docs/tradeoffs.md` for what this costs,
where it's the wrong tool, and what it concedes by design.

This repository (`spine/`) is the portable core — scripts, hooks, skills,
agents, rules, artifact templates. It is never installed *into* a project;
projects reference it. Nothing in `spine/` names a language, framework, or
tool — stack specificity lives only in a project's own `.spine/adapters/`,
generated at install time.

## Why this exists

Building almost entirely with AI agents fails in a small number of
specific, repeatable ways. Each mechanism in this system traces to one of
them — nothing here exists for its own sake:

- **Intent evaporates.** Steering happens in chat; only code gets
  committed; the "why this shape" is gone by next week. → `research.md`,
  `plan.md`, and the delta briefing are real files, not scrollback.
- **Agents add, never consolidate.** Duplication compounds weekly. →
  `clone-scan` fails the floor on new duplication touching changed code,
  every task.
- **Agent-written tests routinely assert nothing.** No acceptance
  criteria means review is vibes. → the falsifier subagent stubs out the
  feature logic and re-runs the tests; if they still pass, that's a
  reported finding, not a shrug.
- **Models see source text, not runtime behavior.** Authorization and
  data-integrity bugs dominate production failures for a reason. → the
  security adversary and the smoke/migration lane exist specifically for
  the categories a text-only reviewer can't see.
- **The engineer stops knowing what their product is.** Named first,
  because it's the one that matters most. → the three touchpoints
  (classify, approve the plan, read the briefing) are designed to be the
  minimum that keeps a human oriented, not a rubber stamp.
- **Agents fix symptoms in the wrong layer on a codebase they don't
  understand.** → `docs/map.md` and per-task research exist so a change is
  grounded in how *this* system actually works, not a guess.
- **When plan meets reality, an unsupervised agent improvises — and each
  improvisation is locally reasonable, globally corrosive, and nobody
  decided it.** → the latitude table and the halt tier mean a real
  surprise stops and asks, instead of getting quietly papered over.

None of this removes the need for an engaged engineer — see
`docs/tradeoffs.md` for where this system is the wrong tool, what it
costs, and what it concedes by design. It's a floor, not a substitute for
judgment.

## Install → first task, in one sitting

**1. Clone this repo once per machine, anywhere stable.** Its path is
load-bearing: the install mechanism symlinks a project's
`.claude/{skills,agents,rules}` and `.claude/hooks` to real paths inside
this checkout. Those symlinks are generated locally, on every machine —
never committed (`docs/tradeoffs.md` states the cost: moving this checkout
after installing means re-running `core/scripts/setup`, once, on that
machine — see step 2 and "Staying installed" below).

**2. From a session in *this* repo, install into a project:**

```
/bootstrap --project /path/to/new-project     # nothing there yet
/adopt     --project /path/to/existing-project # real code already there
```

Both ask three layers of questions once (build prompt §0): who you are as
an engineer (asked once, ever, on this machine — stored at
`~/.spine/user-config.json`), what this repo's protected paths and risk
posture are, and what stack it runs (consumed only by adapter generation —
nothing else in the spine ever sees these answers). Answer them; both
skills generate real adapters, run them through conformance, wire the
hooks, and write a starting `CLAUDE.md`, `docs/charter.md` (draft — read
and edit it), and `docs/map.md`.

**3. Start a fresh session inside the installed project.** Hook and skill
wiring doesn't take effect mid-session — this is the one unavoidable
restart in the whole flow.

**A second engineer joining an already-installed project** clones the
project (its `.claude/skills/`, `.claude/agents/`, `.claude/rules/`,
`.claude/hooks` are gitignored — nothing to clone there), clones this repo
once on their own machine, then runs the one per-machine step:

```
/path/to/spine/core/scripts/setup --project /path/to/project
```

Idempotent — safe to re-run any time symlinks look wrong, or after moving
this checkout. Reports a core-version-skew warning if this machine's spine
is ahead or behind the project's recorded pin (see "Staying installed"
below); otherwise silent success.

**Before that first `setup` run, every Edit/Write/Bash is denied outright
— on purpose, confirmed empirically, not assumed.** A committed
`.claude/settings.json` wiring a hook at a path that doesn't exist yet
(`.claude/hooks/<name>`, gitignored, generated only by `setup`) is
*silently skipped* by Claude Code's own hook runner, not blocked — tested
directly: a fresh clone with no `setup` run let a real `Write` through
with no denial and no warning. `.claude/settings.json` therefore routes
every hook through a small, always-committed `.claude/hook-guard` instead
of the generated path directly — it exists on the very first clone, checks
whether the real hook is present, and fails loudly (denying the tool call,
with the fix command in the message) if it isn't. Re-tested the same fresh
clone with the guard in place: the write was correctly denied, with the
exact remediation shown. The pre-`setup` state is meant to be unusable,
loudly — never silently unenforced.

**4. Run your first task:**

```
/task <describe the change>
```

You'll be asked to confirm its class (trivial / standard / high blast
radius), shown a plan to approve before any code changes, and handed a
one-page briefing when it ships. That's the entire recurring interaction
this system asks of you — everything else (research, the deterministic
floor, adversarial review) happens without you in the loop, by design.

**When a plan or briefing confuses you, that's a template defect — say so,
and fix the template, not the instance.** A one-off rewrite of a
confusing plan evaporates the next time one ships; a fix to
`core/templates/plan.md` or `briefing.md` (or the writing mandate both
draw on) persists for every artifact after it — the same compounding
`/ratchet` gives the rest of this system, applied to the two documents you
actually read every task.

## Staying installed

Updating the core is `git pull` in this checkout — zero commits in any
installed project, because nothing there is a copy. `/ratchet` is the only
thing allowed to grow a project's `CLAUDE.md` or `core/rules/`, and it pays
for every addition with a deletion. `/costs` reports what this is actually
costing, untracked-commit ratio first, so calibration can be revised from
data instead of guesses.

**Version pin, for a team of engineers each on their own local spine
checkout (Extension C §2.1).** Every installed project carries
`.spine/core-pin.json` — `{"sha": "<core commit>", "mode": "warn|strict"}`
— committed, ordinary project state, the same as `.spine/capabilities.json`.
`core/scripts/setup --check` (cheap, called at the start of every `/task`)
compares this machine's resolved spine `HEAD` against the pin: a match is
silent; a mismatch prints loudly under `warn`, blocks under `strict` — one
engineer's hooks silently enforcing something a colleague's don't is the
same "manufactures confidence" failure shape as a hook that doesn't fire at
all, arriving through the update channel instead of a broken symlink.

**Upgrade workflow:** a maintainer tests the new core for real —
`adapter-conformance --all` against a real install, the throwaway-hook
fire-test, one trivial task through the full research→ship loop — then
bumps `.spine/core-pin.json`'s `sha` in an ordinary commit to the project's
shared mainline. Every other engineer's next `/task` (or `setup --check`)
announces the mismatch; they resolve it with `git pull` in their own spine
checkout, then `core/scripts/setup --project <project>` to confirm. Nothing
about a project's own files — `.spine/adapters/`, `.spine/capabilities.json`,
`.spine/protected-paths.conf`, `docs/charter.md` — ever requires a pin
bump; those update through the project's normal commits like any other
file, independent of which core commit the project is pinned to. The pin
only ever tracks `spine/`'s own `core/` — the shared enforcement layer,
not project-owned state.

**`.claude/hook-guard` maintenance — the one piece of core that's
committed, not generated, and how it stays in sync.** Everything else
under `.claude/` (skills, agents, rules, hooks) is gitignored and always
resolves live to whatever spine core this machine has checked out;
`hook-guard` exists specifically so a pre-`setup` state still enforces
(see the fresh-clone section above), which requires it to be committed —
and a committed file doesn't refresh itself the way a symlink does. It's
regenerated verbatim from `core/templates/hook-guard` every time `setup`
runs (never hand-edited — if it needs to change, that change happens in
the template, in this repo, like everything else). `setup --check`
compares the committed copy's actual bytes against this machine's
resolved template on every run, **independent of pin match** — a pin
match only proves `.claude/hooks`' generated content would resolve
correctly, it says nothing about whether the committed `hook-guard` file
itself is current, since nothing regenerates a committed file just
because the pin says the core version matches. A stale or hand-edited
guard is caught by this comparison and reported the same way a pin
mismatch is: loud, with the fix command, never silent.

## Layout

```
core/scripts/    the deterministic layer — q, floor, conformance, ledger, ...
core/hooks/      the three PreToolUse gates (phase, protected-path, dependency)
core/rules/      path-scoped discipline (currently: migrations, contracts)
core/skills/     /bootstrap /adopt /design /workspace /task /verify /ship
                 /ratchet /remap /costs /tasks
core/agents/     researcher, falsifier, security — fresh-context, read-only
core/templates/  every artifact format the skills above produce
work/.build/     this build's own phase handoffs — the install's decision record
```

## Command reference

Every command below is a skill under `core/skills/`, resolved live from
whatever this machine's spine checkout has — nothing here is copied into
an installed project. Most take `disable-model-invocation: true`, meaning
you type them; the model doesn't reach for one on its own.

**Install & design — run once, from a session in the spine repo itself
(or per the "second engineer" flow in a project that's already installed)**

| Command | Args | What it does |
|---|---|---|
| `/bootstrap` | `--project <path>` | Install spine into a brand-new, near-empty project — charter first, then calibration, adapter generation, and your first `/task`. |
| `/adopt` | `--project <path>` | Install spine into an existing repo with real code — calibration, adapter generation, a bounded survey, and a charter draft you edit. Re-running it later recalibrates rather than reinstalling. |
| `/design` | `[--project <path>]` | Optional, greenfield-first design stage: a facilitated session turning a confirmed charter into foundational decisions, a walking-skeleton milestone, and a reviewed stopping point *before* feature code exists. |
| `/workspace` | `--root <path> [--repo <name>=<path> ...] [--from-design <path>]` | Multi-repo only. Initializes or extends a workspace root that coordinates several repositories through declared contracts. Run once per workspace, never per task. |

**The daily loop — inside an installed project, per task**

| Command | Args | What it does |
|---|---|---|
| `/task` | `<description> [--milestone <id>]` | The default way any non-trivial change gets made: classify → research → plan (you approve it) → implement → verify → ship. This is the one you actually type most days. |
| `/verify` | `<task-id>` | Runs the deterministic floor plus the adversary agents (falsifier, and security per class/ceremony) and assembles `verify.md`. `disable-model-invocation: true` means `/task` cannot call this itself — at the verify phase it asks you to type `/verify <task-id>` yourself, waits, then reads the result back from `work/<task-id>/verify.md` rather than assuming. |
| `/ship` | `<task-id> [--bypass <reason>]` | Gates (floor passed, no open deviations, second approver if Class 2), commits, and writes the delta briefing plus a PR description assembled from the task's own verified record for a task that's passed verification. Same `disable-model-invocation: true` rule as `/verify` — `/task` asks you to run it yourself once verify passes, or you can run it directly with `--bypass` for a genuine emergency that can't wait — loud and recorded, never silent. |

**Maintenance & visibility — not part of any one task**

| Command | Args | What it does |
|---|---|---|
| `/tasks` | *(none)* | Lists every open task in the registry — owner, class, phase, claims, flags. Read-only; exists so a human sees the same picture `claims-check`/`propagate` do. |
| `/costs` | `[--since <git-date>]` | Reports cost instrumentation: untracked-commit ratio first, then per-task/per-engineer ledger aggregates. An instrument for spotting drift early, not a leaderboard. |
| `/ratchet` | `<description of the recurring finding>` | Converts a finding that's genuinely recurred twice (two deviations, two adversary verdicts, two relayed review comments) into a deterministic check, deleting the prose rule it supersedes. The only command allowed to grow `CLAUDE.md` or the rules directory. |
| `/remap` | *(none)* | Regenerates `docs/map.md` from the current repository's real state, stamped with the current commit SHA. Runs isolated from the calling conversation. |
| `/task-report` | `<task-id> [--project <path>]` | Generates a self-contained HTML visualization of one task's `work/` record — timeline, deviations, adversary activity, floor results — for a quick look without reading five files by hand. Never a gate. |
| `/visualize` | `[--project <path>]` | Generates a project-wide HTML dashboard — Gantt timeline of every task, a chronological event feed, the decision store with supersession chains, the live capability matrix, milestone progress, and the /costs-style drift instrument. Never a gate. |


## Working with other engineers (2–4, on the same project)

`/task` opens a task by committing and pushing its folder to the shared
mainline immediately — before any code is written, not at ship time — so
`/tasks` (read-only, lists every open task's owner/class/phase/claims/
flags) shows the same picture to everyone. Two mechanisms catch what git
alone can't: `claims-check` blocks plan approval on a real write conflict
with another open task (a shared *read* only warns — two people reading
the same file is normal); `propagate` flags a task when something it
grounds on changes underfoot, and that task can't advance phase until the
flag is acknowledged. Class 2 needs a second approver — a different git
identity than the task's owner — with a loud, ledger-visible override for
the genuine solo/on-call case. `/costs` reports all of this per engineer,
framed as an instrument for spotting drift early, not a leaderboard —
`docs/tradeoffs.md`'s Extension C section has the full accounting,
including what's still enforced by an agent following instructions rather
than a hook, and what breaks past about 4 engineers.

## If something feels like it's fighting you

It might be right to fight you — re-read `core/ADAPTER-CONTRACT.md` and the
hook you hit before assuming it's a bug. If it's genuinely wrong for your
situation, `/ratchet` is how a real, recurring friction becomes a fix
instead of a workaround; `/ship --bypass <reason>` exists for the one
situation where you can't wait even for that (a bypass is loud, recorded,
and visible in `/costs` — never silent).
