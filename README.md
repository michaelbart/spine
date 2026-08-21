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
core/scripts/    the deterministic layer — floor, conformance, ledger, ...
core/hooks/      the three PreToolUse gates (phase, protected-path, dependency)
core/skills/     every slash command — see the table below
core/agents/     researcher, falsifier, security — fresh-context, read-only
core/rules/      path-scoped discipline (migrations, contracts)
core/templates/  every artifact format the skills above produce
```

## Commands

Every command is a skill under `core/skills/`, resolved live from whatever
this machine's checkout has. Most take `disable-model-invocation: true` —
you type them, the model doesn't reach for one on its own.

**Set up — run once per project, or when planning what's next**

| Command | Args | What it does |
|---|---|---|
| `/bootstrap` | `--project <path>` | Install spine into a new, near-empty project. |
| `/adopt` | `--project <path>` | Install spine into an existing codebase. Re-run anytime to recalibrate. |
| `/design` | `[--project <path>] [--handoff <path>]` | Facilitated session turning a charter into foundational decisions and milestone 0. `--handoff` feeds it a product spec (first run) or a later design delivery to reconcile against what's already decided (re-entry). |
| `/roadmap` | `[--after M<n>]` | Sequences the next milestones from `docs/vision.md`, absorbing anything flagged along the way. You confirm the order before anything's written. |
| `/workspace` | `--root <path> --repo <name>=<path> ...` | Multi-repo only — sets up a workspace root coordinating several repos through declared contracts. |

**Every task**

| Command | Args | What it does |
|---|---|---|
| `/intake` | `<ticket-key-or-url>` | Front door for ticketed work — sizes it, proposes a class, routes into a trace or a `/task`. |
| `/task` | `[description] [--milestone <id>]` | The default way work gets done: classify → research → plan (you approve it) → implement → verify → ship. Run it bare to auto-continue the next queued task in the current milestone. |
| `/verify` | `<task-id>` | Runs the floor and adversary review, writes `verify.md`. You type this yourself when `/task` asks. |
| `/ship` | `<task-id> [--bypass <reason>]` | Gates, commits, writes the briefing and PR description. You type this yourself when `/task` asks, or use `--bypass` for a genuine emergency (loud and recorded, never silent). |

**Maintenance & visibility — never a gate**

| Command | Args | What it does |
|---|---|---|
| `/spine` | *(none)* | "Where am I, what do I do next." Run this when unsure. |
| `/tasks` | *(none)* | Lists every open task — owner, class, phase, claims, flags. |
| `/costs` | `[--since <date>]` | Cost and drift instrumentation, not a leaderboard. |
| `/ratchet` | `<description>` | Turns a finding that's genuinely recurred twice into a deterministic check. |
| `/remap` | *(none)* | Regenerates `docs/map.md` from real repo state. |
| `/update` | `[--bump-pin]` | Syncs an installed project to this checkout after a `git pull`. |
| `/task-report` | `<task-id>` | Self-contained HTML view of one task's record. |
| `/visualize` | `[--project <path>]` | Project-wide HTML dashboard — timeline, decisions, milestones, drift. |

## Working with other engineers

`/task` commits and pushes its task folder to the shared mainline before
any code is written, so everyone sees the same open-task picture. Two
mechanisms catch what git alone can't: a real write conflict with another
open task blocks plan approval, and a task whose grounding changed
underfoot gets flagged and can't advance until that's acknowledged. Class 2
changes need a second approver. Full accounting — including what still
relies on an agent following instructions rather than a hook, and where
this stops scaling — in `docs/tradeoffs.md`'s Extension C section.

## Cross-repo work

`/workspace` is for a change spanning more than one repository. It creates
one small workspace root holding the contract registry and the tasks that
touch more than one repo at once — each member repo keeps its own normal
spine install untouched. See `docs/tradeoffs.md`'s Extension B section for
the full model and its disclosed limits.

## If something feels like it's fighting you

It might be right to — re-read `core/ADAPTER-CONTRACT.md` and the hook you
hit before assuming it's a bug. If it's genuinely wrong for your situation,
`/ratchet` turns a real, recurring friction into a fix instead of a
workaround. `/ship --bypass <reason>` exists for the one case you can't
wait even for that.
