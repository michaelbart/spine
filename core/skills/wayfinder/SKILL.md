---
name: wayfinder
description: Chart a large, genuinely foggy effort as a map of decision tickets and resolve them one session at a time, until the map clears into real docs/vision.md milestone entries (and, where warranted, real docs/decisions/ records). For when the destination is known but the path isn't — too big for one /design session's fixed six categories, not yet a known list /roadmap could sequence.
disable-model-invocation: true
argument-hint: [--map <map-id>]
---

You are running `/wayfinder` against `$ARGUMENTS` (an optional `--map
<map-id>` to target a specific map instead of the active one). Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/<name>`, templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`. Hand
`${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim, `../../` included —
do **not** lexically collapse it to `.claude/`; `.claude/skills/wayfinder`
is a symlink into the spine core checkout.

**What this is for, and what it isn't.** `/design` walks a fixed six
categories in one session. `/roadmap` sequences a milestone list that's
already known. Neither covers an effort whose destination is known but
whose path is still foggy — too large and undecided for either. This
skill charts that as a map of decision tickets, resolved one at a time,
one per session by default, until the map clears — then it writes real
`docs/vision.md` milestone entries (and, where a ticket genuinely
resolved an architecture question, a real `docs/decisions/` record) and
hands off to `/roadmap`, the same way `/design` already hands off to it.

**This is sequential, file-coordinated, one map active at a time — never
parallel.** `.spine/current-wayfinder` names at most one active map id.
There is no "claim" mechanism and no notion of two sessions working the
same map concurrently; `docs/tradeoffs.md`'s Deferred table still means
what it says about team-of-agents orchestration. This skill's only claim
is the narrower one: a large effort's shape can be worked out across many
*sequential* human sessions, the same way a milestone's member tasks
already are, just one level earlier.

## 0. Preflight

`docs/charter.md` must already exist (same precondition `/design` uses) —
if it's missing, stop and say so; `/wayfinder` charts an effort within an
established project, it doesn't found one.

**If `.spine/current-wayfinder` exists** (and `--map` wasn't given, or
names the same id), this is a resume — skip to §2. **If `--map <id>` was
given** and it doesn't match the active pointer, resume that specific map
instead (`work/wayfinder/<id>/map.md` must exist) without changing
`.spine/current-wayfinder` — this is a deliberate one-off look at a
different map, not a switch. If it doesn't exist, say so and stop.

**If neither applies, this is a new map** — go to §1.

## 1. Chart a new map

Before creating anything, a proportionate sanity check, same spirit as
`/prototype`'s own §0: if the human already knows the milestone list, say
so and point at `/roadmap` instead — it's faster and doesn't need a map.
If this is really one architecture question, not an effort with many open
questions, point at `/design` (or, if genuinely just one visual/behavioral
unknown, `/prototype`). Proceed here only when the honest answer is "the
path itself is unclear, and it's bigger than one sitting."

Ask the human for the destination — what this effort is trying to reach,
in a paragraph. Then propose an initial set of decision tickets grounded
in `docs/charter.md` (and `docs/map.md`/`docs/design-summary.md` if they
exist) — **propose, don't decide**, the same posture `/design` §1 already
uses. For each proposed ticket, its type follows from what would actually
resolve it, not from guessing — pick whichever is true:

- **grilling** — resolvable by a facilitated conversation, right now or
  in a future session; the default when nothing more concrete is needed.
- **prototype** — resolvable only by building something concrete and
  seeing/using it — a real visual or behavioral unknown.
- **research** — resolvable by investigating how the existing code
  actually behaves today — a factual question, not a judgment call.

Let the human confirm, add, remove, split, or retype any proposed ticket
before anything is written — nothing is written until this converges,
same as `/roadmap` §2's own confirmation step.

Generate the map id: one more than the highest existing `work/wayfinder/
W<n>/`, or `W1` if none exist. `mkdir -p work/wayfinder/<map-id>/tickets`.
Write `work/wayfinder/<map-id>/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/wayfinder-map.md` with the
destination paragraph and the confirmed tickets' table rows (`T1`, `T2`,
... in the order presented, all `Status: open`, `Blocked-by` only where a
ticket genuinely can't start before another resolves — most won't have
one). Write each ticket's own file from
`${CLAUDE_SKILL_DIR}/../../templates/wayfinder-ticket.md`. Write
`.spine/current-wayfinder` = `<map-id>`.

Commit (untrailered — setup-shaped, no task exists):

```
git add -- work/wayfinder/<map-id>/ .spine/current-wayfinder
git commit -m "spine: wayfinder <map-id> opened — <n> tickets"
```

Continue straight to §3 (work the first session's ticket) rather than
stopping here — charting the map and starting to resolve it are the same
sitting by default, though the human is always free to stop after this
commit instead.

## 2. Resume

Read `work/wayfinder/<map-id>/map.md`'s `## Destination` for orientation,
then run:

```
${CLAUDE_SKILL_DIR}/../../scripts/wayfinder-frontier --map <map-id> --project <project root>
```

Branch on its first token:

- **`CLEARED`** — every ticket is resolved. Go to §4 (handoff) — don't
  ask what to work on, there's nothing left to.
- **`STALLED <id> <id> ...`** — open tickets remain, but every one has an
  unresolved blocker (a real impasse, possibly a cycle). Show the human
  the stalled tickets and their `Blocked-by` chains verbatim; this needs
  a human decision (re-scope one, drop a blocking relationship that
  turns out not to be real, or split a ticket) — never resolved by
  picking one to work anyway. Stop and wait, with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start -->
  > **Deciding:** how to get unstuck. It's yours because every open question here is waiting on another one, and only you can say which dependency is real.
  > **Need to know:** <each stuck question in one plain sentence, and what it's waiting on>. Full chains are in the map file.
  > **Recommend:** <re-scope | drop a dependency | split> <which one> — <one-line reason>
  > 1. **Drop a dependency that isn't real** — next: I remove that link and the question becomes workable; cost: a minute; undo: yes
  > 2. **Re-scope or split a question** — next: we reshape it so it stops waiting; cost: a short discussion; undo: yes
  > **Safe to ignore:** the ticket ids; they're in the map file.
  <!-- touchpoint:end -->
- **`FRONTIER <id> <id> ...`** — go to §3.

If `wayfinder-frontier` could not run at all (per the tooling-gap
discipline `core/skills/task/SKILL.md`'s header describes — blocked,
denied, or errored before its own logic ran), say so plainly and fall
back to reading `map.md`'s table by hand rather than silently treating
"couldn't check" as "nothing to do."

## 3. Work a ticket

Present the frontier (question, one line each, from the ticket files —
never just the ids). Ask which to work this session; default to the
lowest id if the human has no preference. Read that ticket's full file
first — its `## Question` is the real thing the map.md row's one-liner
only summarizes.

Resolve by type:

**grilling** — have the conversation right now, in this session. Propose
a way to resolve the question, grounded in the charter/decisions/map the
same way `/design` grounds its own proposals; let the human confirm,
revise, or push back, iterating until it converges. Write the resolution
into `## Resolution` — the reasoning and the answer, not a transcript.

**prototype** — tell the human plainly: this needs a real `/prototype
<question>` session, and this ticket stays open until they come back
with it. Do not build a prototype inline here — that's `/prototype`'s own
job, with its own disposable-artifact discipline (`core/skills/prototype/
SKILL.md`); duplicating it here would be a second, undisciplined copy of
the same mechanism. Stop this session's work on this ticket (the human
may still pick a different frontier ticket to work instead, or end the
session here). When resumed after the prototype session: read its
`work/prototypes/<id>/findings.md`, write `## Resolution` citing it plus
one sentence on what it settled.

**research** — delegate to the researcher agent right now (Agent tool,
`subagent_type: researcher`), delegation message containing: the
ticket's real question (from `## Question`, not the map row's one-liner),
`<map-id>.<ticket-id>` in place of a task id (purely for the agent's own
citation use — this is not a real task and nothing here writes to
`work/<task-id>/`), and pointers to `docs/charter.md`/`docs/map.md` if
they exist. Write the agent's full reply verbatim to
`work/wayfinder/<map-id>/tickets/<ticket-id>-research.md` — the same
"write the whole reply, don't paraphrase it" rule `/task` §2 already uses
for a real research.md. Then write this ticket's own `## Resolution`: one
paragraph summarizing the conclusion, citing that sibling file for the
full investigation.

**For any type**, once resolved: flip `Status: resolved`, fill
`Resolved:`, fill `## Spawned tickets` (`none`, or new ticket ids this
resolution revealed). For each spawned ticket: allocate the next id from
`map.md`'s `next-ticket-id` counter, add its `map.md` row (`Status: open`,
`Blocked-by` this ticket only if resolving this one was a genuine
precondition), and write its own ticket file. Update `map.md`'s row for
the ticket just resolved (`Status: resolved`).

Commit (untrailered):

```
git add -- work/wayfinder/<map-id>/
git commit -m "spine: wayfinder <map-id> — resolved <ticket-id>"
```

Re-run `wayfinder-frontier`, report the new frontier (or `CLEARED`), and
stop for the session — one ticket resolved per session is the default
cadence. If the human explicitly wants to keep going, loop back to the
top of this section rather than treating "one per session" as a hard
cap; it's a default, not a rule enforced by anything.

## 4. Handoff — the map is cleared

Walk every resolved ticket once. For each, decide which of two real
outcomes it is — never invent a third:

- **A build-order item** — the resolution implies a real chunk of future
  work (most `prototype`/`research`-resolved tickets, and most
  `grilling` ones too). Propose a `docs/vision.md` `## Planned
  milestones` line for it, same one-line-per-milestone shape
  `core/skills/roadmap/SKILL.md` §0's own preflight already uses.
- **An architecture decision** — the resolution is really a standing rule
  future work should follow (module boundaries, a persistence choice, a
  process rule), not a scheduled chunk of work. Propose a real
  `docs/decisions/D-<n>.md` record via the **human-directed, disclosed**
  entry path (`docs/decisions/decision.md`'s own header, third path) —
  `Category: other` (that path never uses the six named categories, only
  `/design` does), `Source: human decision (<yyyy-mm-dd>)`, `## Context`
  opening with "via `/wayfinder` map `<map-id>`, ticket `<ticket-id>` (see
  `work/wayfinder/<map-id>/tickets/<ticket-id>.md`)." Written directly as
  `adopted` — no design review to run here, same reasoning that entry
  path already states. This is the *same* store and the *same* three
  entry paths `docs/decisions/decision.md` names — never a fourth format.

Present every proposed `docs/vision.md` line and every proposed decision
record together, before writing anything — the human confirms the whole
set, same as `/roadmap` §2's own convergence step. A resolved ticket that
implies neither (a pure factual finding with no forward action) needs
neither — say so plainly rather than manufacturing one of the two to look
thorough, the same caution `/design` states about its own DEFERRED.md.

On confirmation:

- Append confirmed lines to `docs/vision.md`'s `## Planned milestones`
  (create the file, same minimal shape `/roadmap`'s own preflight would,
  if it doesn't exist yet) — append-only, never reorder or rewrite
  existing lines.
- Write each confirmed decision record, then
  `${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <project root>`
  — never let the index ship a step behind the store, same rule `/design`
  §7.6 already follows.

Delete `.spine/current-wayfinder`. Commit (untrailered):

```
git add -- docs/vision.md docs/decisions/ work/wayfinder/<map-id>/ .spine/current-wayfinder
git commit -m "spine: wayfinder <map-id> cleared — <n> vision.md entries, <m> decisions recorded"
```

Tell the human: how many of each, and that **`/roadmap`** is the next
real command — it picks up `docs/vision.md`'s new entries the same way it
already picks up any other planned-milestone line, no change on its end.
