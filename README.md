# spine

An AI-assisted development workflow: research → plan → human approval →
implement → deterministic floor → adversarial review → ship. Part of it is
mechanical (three hooks that block bad writes, a floor of scripts that run the
project's own checks, scripts that compute answers a model would otherwise
guess). Part of it is instructions a model follows. `docs/enforcement-map.md`
says, rule by rule, which is which; `docs/tradeoffs.md` says what it costs and
where it is the wrong tool.

This repository is the portable core — scripts, hooks, skills, agents,
rules, templates. It's never copied into a project; projects just point at
it. Nothing in here names a language, framework, or tool — that specificity
lives only in a project's own `.spine/adapters/`, generated at install time.

## Why

A handful of specific, repeatable ways AI-driven development goes wrong,
and what here exists to catch each one:

- Steering happens in chat, only code gets committed, and the "why" is gone
  by next week → `research.md`, `plan.md`, and `docs/decisions/` are real
  files, not scrollback.
- Duplication compounds quietly → the floor can fail on new duplication in
  changed code, **if the project has a `clone-scan` adapter**. Neither
  turnpilot nor tgml has one yet, so that check has never run there (`/spine`
  reports checks that keep getting skipped).
- Agent-written tests routinely assert nothing → the falsifier subagent
  stubs out the feature and re-runs them; if they still pass, that's a
  finding.
- Models see source text, not runtime behavior → a security adversary and a
  smoke/migration lane exist for the bugs a text-only reviewer can't see.
- An unsupervised agent improvises when reality diverges from the plan, and
  nobody decided that → real surprises halt and ask instead of getting
  quietly papered over.
- The engineer stops knowing what their own product is → three touchpoints
  (classify, approve the plan, read the briefing) keep a human oriented,
  without turning into a rubber stamp.
- A large, foggy effort gets committed to in one blind pass →
  `/wayfinder` charts it as a map of decision tickets instead, resolved
  one at a time.
- A spike whose whole point is "I don't know what I want yet" gets forced
  through research → plan → implement anyway → `/prototype` is a disposable,
  ungated escape hatch built for exactly that.

None of this replaces an engaged engineer — it's a floor, not a substitute
for judgment.

## How it's built

Three plain kinds of files, wired together by one convention:

1. **Skills** — the slash commands (`/task`, `/verify`, `/ship`, ...). Each
   is a markdown instruction file a model reads and follows step by step.
   The big ones keep the common path in `SKILL.md` and load conditional
   material from a `reference/` folder only when its trigger applies.
2. **Agents** — fresh, isolated sessions with a narrow job and no memory of
   the conversation that spawned them (`researcher`, `falsifier`,
   `security`). Independence is the point: a fresh session can't be swayed
   by the reasoning that wrote the code.
3. **Hooks + scripts** — real shell code, not prompts. Three hooks block a
   write before it happens (outside the task folder during research and plan;
   to a protected path unless the task is Class 2; a package install or manifest
   edit without asking). Scripts do mechanical work: run the floor, check that
   research is still current, filter adversary findings, gate phase changes.

What that buys, honestly: the hooks and scripts can't be talked out of
their answer, but a skill still has to tell the model to run most scripts, and
the hooks read `state` and `class` files the model itself writes. So the system
catches mistakes and drift well, and is not a defence against a model that is
trying to get around it. See `docs/enforcement-map.md`.

## Get started

**1. Clone this repo once per machine, anywhere stable.** Its path is
load-bearing — installing a project symlinks its `.claude/` wiring to real
paths inside this checkout. **Keep this checkout on `main`:** every installed
project runs whatever branch is checked out here. Develop changes to spine in a
separate `git worktree`, and never run `core/scripts/setup` from one against a
real project.

**2. Install into a project, from a session in this repo:**

```
/bootstrap --project /path/to/new-project      # nothing there yet
/adopt     --project /path/to/existing-project  # real code already there
```

Answer the setup questions once; both generate real adapters for your
stack, wire the hooks, and write a starting `CLAUDE.md`, `docs/charter.md`,
and `docs/map.md`.

**3. Start a fresh session inside the installed project** — hook/skill
wiring doesn't take effect mid-session.

**4. Run your first task:**

```
/task <describe the change>
```

You'll confirm its class, approve a plan before any code changes, and get a
one-page briefing when it ships. That's the whole recurring interaction —
research, the floor, and adversarial review all happen without you in the
loop.

## The short path

Most days you need four commands and three decisions:

| Command | When |
|---|---|
| `/spine` | You're not sure where you are or what to do next. Also shows a few recent numbers. |
| `/task <description>` | Start a change. Decisions: confirm the class, approve the plan. |
| `/verify <task-id>` | `/task` asks you to run it when the code is built. |
| `/ship <task-id>` | After a passing verify. Decision: read the briefing. |

Everything else below is setup, planning, or occasional.

## Staying installed

Updating the core is `git pull` in this checkout — zero commits in any
installed project. Run `/update` in a project afterward to sync it and see
what changed (it also removes links to skills that spine no longer ships).

Task folders (`work/<task-id>/`): a new install commits everything in a task folder
except raw tool output in `artifacts/` (older installs may also ignore the state
files). `/ship` commits them together with the change, so task history lands in one
commit at ship time instead of one per phase. Nothing coordinates two engineers on
one repo — spine is built for one person working one task at a time per checkout.

## Layout

```
core/scripts/          the deterministic layer — floor, check-stale, set-state, spine-stats, ...
core/hooks/            the three PreToolUse gates (phase, protected-path, dependency) and their helpers
core/skills/           every slash command; big ones have a reference/ folder loaded on demand
core/agents/           researcher, falsifier, security, ui-fidelity, surveyor — fresh-context
core/rules/            path-scoped discipline (migrations, auth, UI design system)
core/templates/        every artifact format the skills above produce
core/prompt-templates/ ready-to-paste prompts for generating that material elsewhere (see `/prompts`)
core/budget.json       size caps for skills, scripts and hooks, enforced by core/scripts/repo-lint
docs/                  tradeoffs, enforcement map, event schema, baseline numbers, design/ and history/
```

## Commands

Every command is a skill under `core/skills/`, resolved live from whatever
this machine's checkout has. All of them take `disable-model-invocation:
true` — you type them, the model doesn't reach for one on its own.

**Every task — the daily loop**

| Command | Args | What it does |
|---|---|---|
| `/task` | `[description] [--milestone <id>]` | The default way work gets done: classify → research → plan (you approve it) → implement → verify → ship. Run it bare to auto-continue the next queued task in the current milestone. With a task already active it offers to resume it or abandon it (with a reason). |
| `/verify` | `<task-id>` | Runs the floor and adversary review, writes `verify.md`. You type this yourself when `/task` asks. |
| `/ship` | `<task-id> [--bypass <reason>]` | Re-grounds against what changed underfoot, gates, distills decisions, does milestone bookkeeping, writes the briefing and PR description, commits. You type this yourself when `/task` asks, or use `--bypass` for a genuine emergency (loud and recorded, never silent). |
| `/spine` | *(none)* | "Where am I, what do I do next." Read-only. |

`/task` walks through six steps, in plain terms:

1. **Classify** — you and the model agree how big a deal this change is:
   Class 0 (trivial: a couple of files, a few lines; it runs only a types and
   lint check), Class 1 (standard) or Class 2 (high-risk, protected paths). That
   sets how much process kicks in for everything after it.
2. **Research** — a fresh agent with no stake in the outcome reads the
   actual code and writes down what's really there, before anyone proposes
   how to change it.
3. **Plan** — a plan gets written from that research, and you read and
   approve it before any code changes. The default is `guided` (a stop after
   every phase), and Class 2 is always `guided`. The one opt-in exception is
   `auto`: no plan-approval stop at all, and you review the finished draft PR
   instead. `/intake`/`/task` ask which you want; neither is the norm.
4. **Implement** — the plan gets carried out. If reality doesn't match the
   plan, small surprises are just noted and it keeps going; a real one
   stops and asks instead of improvising past it. Moving to `implement` is
   refused until the plan has an approval record.
5. **Verify** — a deterministic floor (the project's typecheck, lint, tests,
   secret scan, and whatever other checks it has adapters for) runs and must
   pass. Fresh adversarial agents also try to break what was built — one tries
   to prove the tests are fake, another looks for security holes — but their
   findings surface for a human to weigh, they don't auto-block; only the floor
   and an unresolved plan-deviation do.
6. **Ship** — the change is gated (a passing verify and no open deviations are
   required), committed, and (for `auto`) a draft PR is opened, and you get a
   one-page briefing of what actually happened.

**Set up and plan — once per project, or when planning what's next**

Roughly top-to-bottom: install, then design, then figure out what to build.
`/wayfinder` and `/prototype` are the two conditional steps — reach for them
only when the plain path (`/roadmap` sequencing an already-known list; plain
discussion settling a category) genuinely isn't enough.

| Command | Args | What it does |
|---|---|---|
| `/bootstrap` | `--project <path>` | Install spine into a new, near-empty project. |
| `/adopt` | `--project <path>` | Install spine into an existing codebase. Re-run anytime to recalibrate. |
| `/design` | `[--project <path>] [--handoff <path>]` | Facilitated session turning a charter into foundational decisions and milestone 0. `--handoff` feeds it a product spec (first run) or a later design delivery to reconcile against what's already decided (re-entry). |
| `/roadmap` | `[--after M<n>]` | Sequences the next milestones from `docs/vision.md`, absorbing anything flagged along the way. You confirm the order before anything's written. |
| `/update` | *(none)* | Syncs an installed project to this checkout after a `git pull`. |
| `/wayfinder` | `[--map <map-id>]` | For an effort too large and foggy for `/design`'s six categories or `/roadmap`'s known list: charts it as a map of decision tickets, resolved one per session, until it clears into real `docs/vision.md` entries and (where warranted) real decisions. |
| `/prototype` | `<question>` | Build a concrete, disposable artifact to settle a visual/behavioral question discussion can't. No class, no plan, no floor, no ship — the declared exception to research → plan → implement, usable any time. |

**Occasional — useful, not part of the loop**

| Command | Args | What it does |
|---|---|---|
| `/intake` | `<ticket-key-or-url>` | Front door for ticketed work — sizes it, proposes a class, routes into a trace or a `/task`. |
| `/ticket` | `[<task-id>]` | Generates a copy-paste ticket description (summary, acceptance criteria, QA notes, files changed) from a verified task's own artifacts. Output only; `/ship` doesn't depend on it. |
| `/note-issue` | `<one-line description>` | Log a known issue found outside any active task or ticket — no classification, no ceremony. Written to `docs/known-issues.md`; `/roadmap` absorbs every open entry into a milestone or an explicit decline. |
| `/ratchet` | `<description>` | Turns a finding that's genuinely recurred twice into a deterministic check. |
| `/remap` | *(none)* | Regenerates `docs/map.md` from real repo state. |
| `/prompts` | `[product-spec \| feature-handoff \| ui-handoff]` | Prints a ready-to-paste prompt for generating handoff material in another session — a product spec, a mid-project feature handoff, or a Claude Design UI handoff bundle. |

**Experimental**

| Command | Args | What it does |
|---|---|---|
| `/autopilot` | `[--milestone <id>]` | Runs a whole planned milestone backlog with no human stops and one end-of-run report. Every commit stays local. See "Experimental" below. |

## Is it working?

Spine measures itself, read-only, from the project's own event log
(`.spine/events.jsonl`), Claude's session transcripts, and the task folders:

```
core/scripts/spine-stats --project <path>                # everything
core/scripts/spine-stats --project <path> friction       # hook stops, by reason
core/scripts/spine-stats --project <path> adversaries    # share of findings fixed
core/scripts/spine-stats --project <path> --brief        # the lines /spine shows
```

It reports tokens and active time per task, how often the hooks stopped work and
why, how many adversary findings got fixed, which floor checks never ran, and which
commands were used. It prints numbers only, never code or prompts, and nothing
consults it to decide anything. `docs/baseline-2026-10.md` records the first run on
two real projects; `docs/event-schema.md` documents every event. Claude Code prunes
old transcripts, so `/ship` copies a task's token totals into its `shipped` event.

For a whole-tree check (the per-task floor only sees the diff), run
`core/scripts/floor 1 --full --project <path>` in CI; nothing runs it automatically.

Spine's own size is capped (`core/budget.json`, checked by `core/scripts/repo-lint`
inside `core/scripts/core-selftest`): adding a command, a script, or skill text past a
cap means removing something first.

## Typical flows

Four common starting points, each ending in the same place: the ordinary
`/task` loop, run repeatedly until a milestone (or the backlog) is done. Run
`/spine` any time you're unsure which step comes next.

**Starting a brand-new project**

1. `/bootstrap --project <path>` — install spine; writes `CLAUDE.md`,
   `docs/charter.md`, `docs/map.md`.
2. Have real product material already? `/prompts product-spec` prints a
   prompt for drafting `docs/product-spec.md` in a separate chat session —
   save it there; `/design` auto-detects it every run, not just the first.
3. `/design` — a facilitated session turning the charter (plus the product
   spec, if any) into the six foundational decisions and milestone 0, the
   walking skeleton.
4. `/task --milestone M0` for M0's first member task, then bare `/task` to
   auto-continue its queue.
5. Once M0 ships: `/roadmap` sequences the next milestones from
   `docs/vision.md` if the shape's already known, or `/wayfinder` first if
   it's still genuinely foggy.

**Adopting spine into an existing codebase**

1. `/adopt --project <path>` — a `surveyor` agent reads the real repo and
   drafts adapters, `CLAUDE.md`, and charter material from what's actually
   there; you confirm or correct each finding, never a cold interview.
2. `/task <description>` for real work right away — `/design` is optional
   on brownfield (making implicit architecture explicit), not required to
   start.
3. Re-run `/adopt` any time to recalibrate — a new capability becomes
   available, a UI handoff bundle gets added, the stack changes.

**Starting a big feature or change mid-project**

1. `/prompts feature-handoff` prints a prompt for drafting
   `docs/features/<slug>-handoff.md` — paste in the project's real
   `docs/charter.md` and `docs/decisions/` so it cites real lines, not
   guesses.
2. That document's own "Routing" section decides what's next:
   `/design --handoff <path>` (revises or extends architecture), a
   `docs/vision.md` addition plus `/roadmap` (known milestone shape),
   `/wayfinder` (genuinely foggy), or straight to `/task` (turns out to be
   task-sized after all).
3. From there, the normal `/task` loop for whatever milestone or tasks
   resulted.

**Building UI faithful to a design**

1. `/prompts ui-handoff` prints three staged Claude Design prompts
   (design the app → extract the design system → export in spine's exact
   schema) — save the result under `docs/ui/`.
2. `core/rules/ui-design-system.md` picks it up automatically, no command
   needed — the next task that touches a declared UI path reads the
   tokens, the component library, and the relevant screen's JSON *and*
   screenshot before writing any code.
3. Sequence the design system/component library's own implementation
   *before* any screen tasks (`docs/ui/handoff.md`'s "Recommended build
   order") — screens built against an unbuilt library is exactly how
   visual drift compounds.
4. Want the deterministic `ui-conformance` check enforced at `/verify`
   time, not just grounding? Re-run `/adopt` to recalibrate once the
   bundle exists. For a real visual comparison of each built screen state
   against its screenshot, the project also needs a `ui-capture` adapter and
   `docs/ui/states/<id>.json` step files (`core/ADAPTER-CONTRACT.md` §3.10);
   Class 1 tasks run it only if `ui_fidelity_class1_optin` is `true` in
   `~/.spine/user-config.json`. That visual-fidelity path has not yet been run
   end to end on a real project.

## Experimental

Each is disclosed with its tradeoffs in `docs/tradeoffs.md` rather than folded
silently into the defaults above:

- **Ticket branches** — `/intake` checks out (or creates and pushes) a
  branch named after the ticket key before a Class 1/2 task starts, instead
  of assuming you'd already branched by hand. Automatic, no flag; Class 0's
  traced-trivial path is untouched. Naming follows this project's own
  `.spine/branch-naming.conf` template (`{ticket}`/`{slug}`/`{type}`/`{user}`
  tokens, e.g. `feature/{ticket}-{slug}`) if one was set during
  `/bootstrap`/`/adopt`, else the plain `{ticket}-{slug}` default.
- **`/autopilot`** `[--milestone <id>]` — loops an already-planned milestone
  backlog end to end with no human stops at all, including the ones Class 2
  normally forces (its guided stops, halt-tier deviations). Every override
  gets logged and reviewed once, at the end, not per task; commits stay
  local, nothing pushes. See `core/skills/autopilot/SKILL.md`.

## If something feels like it's fighting you

It might be right to — re-read `core/ADAPTER-CONTRACT.md` and the hook you
hit before assuming it's a bug. `core/scripts/spine-stats friction` shows which
rule is stopping work and how often; a recurring stop that is wrong for your
situation is a candidate for a fix, and `/ratchet` turns a real, recurring friction
into one instead of a workaround. `/ship --bypass <reason>` exists for the one case
you can't wait even for that.
