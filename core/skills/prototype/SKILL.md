---
name: prototype
description: Build a concrete, disposable artifact to settle a visual or behavioral question discussion alone can't resolve. No class, no plan, no floor, no adversary review, no ship — spine's declared escape hatch for a genuine spike, not a lightweight /task. Use when the real question is "what does this feel like," not "is this correct."
disable-model-invocation: true
argument-hint: <question to resolve>
---

You are running `/prototype` against `$ARGUMENTS` — the question this
session exists to answer. Templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`. Hand
`${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim, `../../` included —
do **not** lexically collapse it to `.claude/`; `.claude/skills/prototype`
is a symlink into the spine core checkout.

**This is spine's declared exception, not a smaller `/task`.** `docs/
tradeoffs.md` names exploratory spikes as the wrong shape for research →
plan → implement — this skill exists so that exception has a real,
bounded home instead of staying undocumented friction. Nothing it produces
is shippable by construction: no `work/<task-id>/`, no class, no floor, no
`Spine-Task:` trailer. If what actually gets decided here needs to become
real product code, that happens afterward, through a real `/task` — this
session's own artifact is never silently promoted into one.

## 0. Is this really a prototype question?

A prototype earns its cost when the honest answer to "could we settle
this by talking it through, or by reading the code" is no — the question
is genuinely "what does this feel like" or "does this actually work," not
"what's correct." If `$ARGUMENTS` reads more like an implementation
request than an open question (a clear behavior, just not built yet), say
so plainly and ask: continue as a prototype anyway, or stop here and
suggest `/task` instead? This is a one-line sanity check, not a gate —
proceed on the human's word either way. Don't run this check at all if
the question is obviously genuine (a real visual/behavioral unknown) —
this step exists to catch the mismatch, not to interrogate every request.

## 1. Set up

Generate a prototype id: `<YYYYMMDD>-<kebab-slug>` — same shape as a task
id, but a different parent directory (`work/prototypes/<id>/`, never
`work/<id>/`) so nothing that globs `work/*/state` or `work/M*/
milestone.md` ever mistakes this for a task or a milestone folder.

`mkdir -p work/prototypes/<id>`. Write `work/prototypes/<id>/findings.md`
now, from `${CLAUDE_SKILL_DIR}/../../templates/prototype-findings.md`,
with just the header fields filled in (`Question`, `Started`,
`Disposition` left blank) and every prose section still empty — this
reserves the file and the id before any building starts, the same
"visible before it's finished" posture task folders already get at
classify time, scaled down to what a prototype actually needs (no
`.spine/current-task`-style pointer — a prototype session is short enough,
and disposable enough, that mid-session resumability isn't worth a second
piece of state to keep in sync).

## 2. Build

Build whatever concrete thing answers the question fastest and cheapest —
a real UI mockup, a tiny throwaway script, a mocked interaction, whatever
form the question actually takes. Everything lives under
`work/prototypes/<id>/`. Two real constraints, both already mechanically
enforced rather than asked for:

- **No active task, so `path-escalate` defaults to Class 0** — a
  prototype that reaches into a protected path (`core/skills/task/
  SKILL.md`'s own Class 0 description) still halts, exactly as it should:
  even throwaway work doesn't get a free pass into auth, migrations, or
  whatever else this project marked high-blast-radius. If that happens,
  say so plainly and stop — this is the hook telling you the question
  needs a real task, not a workaround to route past.
- **No floor, no adapters, no adversary review, on purpose.** Don't run
  `.spine/adapters/*`, don't invoke the falsifier or security agents,
  don't write a `ledger.json`. A prototype's whole value is being cheaper
  than that ceremony — running it anyway defeats the point of this skill
  existing.

Iterate with the human as needed — this is exploratory by definition, so
expect the shape of "what to build" to shift as the answer starts to show
itself. That's normal, not a deviation; there's no plan here for reality
to diverge from.

## 3. Record what was learned

Once the question has a real answer, fill in `findings.md`'s `## What was
built` and `## What was learned` sections — the conclusion stated
plainly, not a narrative of the process. Ask the human: **kept or
discarded?**

- **Discarded** (the common case) — delete everything under
  `work/prototypes/<id>/` except `findings.md` itself. The artifact did
  its job; the finding is what's worth keeping.
- **Kept** — leave the artifact in place alongside `findings.md`. Do this
  when the artifact itself is worth having as a reference later (a real
  mockup, a spike worth revisiting), not by default.

Fill in `Disposition:` accordingly.

## 4. Where this feeds back

Ask, and fill in `## Feeds into` honestly — not every prototype resolves
something formal, and "nothing yet — recorded for later reference" is a
legitimate answer, same spirit as `/design`'s own "don't manufacture a
decision because content arrived":

- **Resolves a `/design` foundational category** — tell the human to
  resume (or start) that `/design` session; it should cite this file in
  the relevant decision's `## Context`, not restate the finding.
- **Resolves a `/wayfinder` ticket** — tell the human to resume
  `/wayfinder`; it writes this file's path into that ticket's own `##
  Resolution` (`core/skills/wayfinder/SKILL.md`).
- **Resolves an ambiguity blocking a `/task`'s plan** — tell the human to
  resume that task; the plan (or, if already mid-implementation, a
  deviation record) cites this file the same way it would cite any other
  grounding.
- **Nothing formal yet** — say so, and stop. The file stays as a real,
  citable record either way.

## 5. Commit

One commit, untrailered — same "setup-shaped, not a task" precedent
`/design`'s own commit already uses (no `Spine-Task:` id; there is none):

```
git add -- work/prototypes/<id>/
git commit -m "spine: prototype <id> — <one-line question>, <kept|discarded>"
```

Tell the human what was learned, the disposition, and the one real next
command from step 4 if there is one.
