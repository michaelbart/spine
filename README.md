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

**1. Clone this repo once, anywhere stable.** Its path is load-bearing: the
install mechanism symlinks a project's `.claude/{skills,agents,rules}` and
`.claude/hooks` to real paths inside this checkout (`docs/tradeoffs.md`
states the cost — moving this checkout after installing breaks every
project that references it).

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

**4. Run your first task:**

```
/task <describe the change>
```

You'll be asked to confirm its class (trivial / standard / high blast
radius), shown a plan to approve before any code changes, and handed a
one-page briefing when it ships. That's the entire recurring interaction
this system asks of you — everything else (research, the deterministic
floor, adversarial review) happens without you in the loop, by design.

## Staying installed

Updating the core is `git pull` in this checkout — zero commits in any
installed project, because nothing there is a copy. `/ratchet` is the only
thing allowed to grow a project's `CLAUDE.md` or `core/rules/`, and it pays
for every addition with a deletion. `/costs` reports what this is actually
costing, untracked-commit ratio first, so calibration can be revised from
data instead of guesses.

## Layout

```
core/scripts/    the deterministic layer — q, floor, conformance, ledger, ...
core/hooks/      the three PreToolUse gates (phase, protected-path, dependency)
core/rules/      path-scoped discipline (currently: migrations)
core/skills/     /bootstrap /adopt /task /verify /ship /ratchet /remap /costs
core/agents/     researcher, falsifier, security — fresh-context, read-only
core/templates/  every artifact format the skills above produce
work/.build/     this build's own phase handoffs — the install's decision record
```

## If something feels like it's fighting you

It might be right to fight you — re-read `core/ADAPTER-CONTRACT.md` and the
hook you hit before assuming it's a bug. If it's genuinely wrong for your
situation, `/ratchet` is how a real, recurring friction becomes a fix
instead of a workaround; `/ship --bypass <reason>` exists for the one
situation where you can't wait even for that (a bypass is loud, recorded,
and visible in `/costs` — never silent).
