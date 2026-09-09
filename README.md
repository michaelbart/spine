# spine

An AI-assisted development workflow with teeth: research → plan → human
approval → implement → deterministic floor → adversarial review → ship.
Every gate is a real script or a real hook, not a prompt asking an agent to
please be careful.

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
- Duplication compounds quietly → the floor fails on new duplication in
  changed code, every task.
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
- A large, foggy effort gets committed to in one blind pass — a whole
  roadmap guessed from a document nobody's actually worked through yet →
  `/wayfinder` charts it as a map of decision tickets instead, resolved
  one at a time, each with the tool that actually fits it.
- A spike whose whole point is "I don't know what I want yet" gets forced
  through research → plan → implement anyway, or done off to the side
  where nothing tracks it → `/prototype` is a disposable, ungated escape
  hatch built for exactly that, instead of undocumented friction.

None of this replaces an engaged engineer — it's a floor, not a substitute
for judgment. See `docs/tradeoffs.md` for where this is the wrong tool and
what it costs.

## How it's built

Three plain kinds of files, wired together by one convention:

1. **Skills** — the slash commands (`/task`, `/verify`, `/ship`, ...). Each
   is a markdown instruction file a model reads and follows step by step.
   No code runs here — this is the "brain" layer.
2. **Agents** — fresh, isolated sessions with a narrow job and no memory of
   the conversation that spawned them (`researcher`, `falsifier`,
   `security`). Independence is the point: a fresh session can't be swayed
   by the reasoning that wrote the code.
3. **Hooks + scripts** — real shell scripts, not prompts. Hooks block a bad
   tool call before it runs; scripts do mechanical work with no judgment
   involved.

Everything a model does is steerable by a good-enough prompt; everything a
hook or script does isn't. That's the actual distinction, and it's why this
calls itself "teeth" rather than more prompting.

## Get started

**1. Clone this repo once per machine, anywhere stable.** Its path is
load-bearing — installing a project symlinks its `.claude/` wiring to real
paths inside this checkout.

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

**A second engineer joining an installed project** clones this repo on
their own machine, then runs `core/scripts/setup --project <path>` once.
Idempotent — safe to re-run any time.

## Staying installed

Updating the core is `git pull` in this checkout — zero commits in any
installed project. Run `/update` in a project afterward to sync it and see
what changed. A team pins the core version each project trusts
(`.spine/core-pin.json`); `/update` and `/task` both surface a mismatch
loudly rather than let one engineer's hooks silently enforce something a
colleague's don't. Full mechanics in `docs/tradeoffs.md`.

## Layout

```
core/scripts/          the deterministic layer — floor, conformance, claims, ...
core/hooks/            the three PreToolUse gates (phase, protected-path, dependency)
core/skills/           every slash command — see the table below
core/agents/           researcher, falsifier, security, surveyor — fresh-context
core/rules/            path-scoped discipline (migrations, contracts, auth, UI design system)
core/templates/        every artifact format the skills above produce
core/prompt-templates/ ready-to-paste prompts for generating that material elsewhere (see `/prompts`)
```

## Commands

Every command is a skill under `core/skills/`, resolved live from whatever
this machine's checkout has. All of them take `disable-model-invocation:
true` — you type them, the model doesn't reach for one on its own.

**Set up — run once per project, or when planning what's next**

Roughly top-to-bottom below: install, then design, then figure out what to
build, then build it. `/wayfinder` and `/prototype` are the two
conditional steps in that sequence — reach for them only when the plain
path (`/roadmap` sequencing an already-known list; plain discussion
settling a category) genuinely isn't enough, never by default.

| Command | Args | What it does |
|---|---|---|
| `/bootstrap` | `--project <path>` | Install spine into a new, near-empty project. |
| `/adopt` | `--project <path>` | Install spine into an existing codebase. Re-run anytime to recalibrate. |
| `/design` | `[--project <path>] [--handoff <path>]` | Facilitated session turning a charter into foundational decisions and milestone 0. `--handoff` feeds it a product spec (first run) or a later design delivery to reconcile against what's already decided (re-entry). |
| `/wayfinder` | `[--map <map-id>]` | For an effort too large and foggy for `/design`'s six categories or `/roadmap`'s known list: charts it as a map of decision tickets, resolved one per session, until it clears into real `docs/vision.md` entries and (where warranted) real decisions. |
| `/roadmap` | `[--after M<n>]` | Sequences the next milestones from `docs/vision.md`, absorbing anything flagged along the way. You confirm the order before anything's written. |
| `/prototype` | `<question>` | Build a concrete, disposable artifact to settle a visual/behavioral question discussion can't. No class, no plan, no floor, no ship — the declared exception to research → plan → implement, usable any time. |
| `/workspace` | `--root <path> [--repo <name>=<path> ...] [--from-design <project-path>]` | Multi-repo only — sets up a workspace root coordinating several repos through declared contracts. |

**Every task**

| Command | Args | What it does |
|---|---|---|
| `/intake` | `<ticket-key-or-url>` | Front door for ticketed work — sizes it, proposes a class, routes into a trace or a `/task`. |
| `/task` | `[description] [--milestone <id>]` | The default way work gets done: classify → research → plan (you approve it) → implement → verify → ship. Run it bare to auto-continue the next queued task in the current milestone. |
| `/verify` | `<task-id>` | Runs the floor and adversary review, writes `verify.md`. You type this yourself when `/task` asks. |
| `/ship` | `<task-id> [--bypass <reason>]` | Six jobs in one command: re-grounds against what changed underfoot, gates, distills decisions, does milestone bookkeeping, writes the briefing and PR description, commits. You type this yourself when `/task` asks, or use `--bypass` for a genuine emergency (loud and recorded, never silent). |

`/task` walks through six steps, in plain terms:

1. **Classify** — you and the model agree how big a deal this change is
   (trivial, standard, or high-risk). That decision sets how much process
   kicks in for everything after it.
2. **Research** — a fresh agent with no stake in the outcome reads the
   actual code and writes down what's really there, before anyone proposes
   how to change it.
3. **Plan** — a plan gets written from that research, and you read and
   approve it before any code changes — the default (`guided`), and the
   right choice for anything but a small, low-risk change. Two narrower,
   opt-in exceptions trade some of that review for less ceremony:
   `checkpointed` (approve the plan, then one finish action) and `auto` (no
   plan-approval stop at all — you review the finished draft PR instead).
   `/intake`/`/task` ask which you want; neither is the norm.
4. **Implement** — the plan gets carried out. If reality doesn't match the
   plan, small surprises are just noted and it keeps going; a real one
   stops and asks instead of improvising past it.
5. **Verify** — a deterministic floor (tests, duplication checks, etc.)
   runs and must pass. Fresh adversarial agents also try to break what was
   built — one tries to prove the tests are fake, another looks for
   security holes — but their findings surface for a human to weigh, they
   don't auto-block; only the floor and an unresolved plan-deviation do.
6. **Ship** — the change is gated, committed, and pushed, and you get a
   one-page briefing of what actually happened.

**Maintenance & visibility — never a gate**

| Command | Args | What it does |
|---|---|---|
| `/spine` | *(none)* | "Where am I, what do I do next." Run this when unsure. |
| `/note-issue` | `<one-line description>` | Log a known issue found outside any active task or ticket — no classification, no ceremony. Written to `docs/known-issues.md`; `/roadmap` absorbs every open entry into a milestone or an explicit decline. |
| `/prompts` | `[product-spec \| feature-handoff \| ui-handoff]` | Prints a ready-to-paste prompt for generating handoff material in another session — a product spec, a mid-project feature handoff, or a Claude Design UI handoff bundle. |
| `/ratchet` | `<description>` | Turns a finding that's genuinely recurred twice into a deterministic check. |
| `/remap` | *(none)* | Regenerates `docs/map.md` from real repo state. |
| `/update` | `[--bump-pin]` | Syncs an installed project to this checkout after a `git pull`. |

## Typical flows

Four common starting points, each ending in the same place: the ordinary
`/task` classify → research → plan → implement → verify → ship loop, run
repeatedly until a milestone (or the backlog) is done. Run `/spine` any
time you're unsure which step comes next.

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
   bundle exists.

## Working with other engineers

`/task` commits and pushes its task folder to the shared mainline before
any code is written, so everyone sees the same open-task picture. Two
mechanisms catch what git alone can't: a real write conflict with another
open task blocks plan approval, and a task whose grounding changed
underfoot gets flagged and can't advance until that's acknowledged. Class 2
changes need a second approver. Full accounting — including what still
relies on an agent following instructions rather than a hook, and where
this stops scaling — in `docs/tradeoffs.md`, under "Working with other
engineers."

## Cross-repo work

`/workspace` is for a change spanning more than one repository. It creates
one small workspace root holding the contract registry and the tasks that
touch more than one repo at once — each member repo keeps its own normal
spine install untouched. See `docs/tradeoffs.md`, under "Cross-repo work,"
for the full model and its disclosed limits.

## Experimental

A few opt-in extensions, each disclosed with its tradeoffs in
`docs/tradeoffs.md` rather than folded silently into the defaults above:

- **Ticket branches** — `/intake` now checks out (or creates and pushes) a
  branch named after the ticket key before a Class 1/2 task starts, instead
  of assuming you'd already branched by hand. Automatic, no flag; Class 0's
  traced-trivial path is untouched.
- **Worktree isolation** — start a second `/task` in another terminal while
  one's already active in the same checkout, and it offers a separate git
  worktree instead of colliding with the first task's uncommitted work. See
  `core/skills/task/SKILL.md`'s Resuming section and `docs/tradeoffs.md`.
- **`/autopilot`** `[--milestone <id>]` — loops an already-planned milestone
  backlog end to end with no human stops at all, including the ones Class 2
  normally forces (a second approver, halt-tier deviations). Every override
  gets logged and reviewed once, at the end, not per task; commits stay
  local, nothing pushes. See `core/skills/autopilot/SKILL.md` and
  `docs/tradeoffs.md`.

## If something feels like it's fighting you

It might be right to — re-read `core/ADAPTER-CONTRACT.md` and the hook you
hit before assuming it's a bug. If it's genuinely wrong for your situation,
`/ratchet` turns a real, recurring friction into a fix instead of a
workaround. `/ship --bypass <reason>` exists for the one case you can't
wait even for that.
