---
name: ship
description: Gate, commit, and brief a task that's passed verification. Invoked by /task at the ship phase, or directly as `/ship --bypass <reason>` for a genuine emergency.
disable-model-invocation: true
argument-hint: <task-id> [--bypass <reason>]
---

You are running `/ship` for task ID and optional `--bypass <reason>` from
`$ARGUMENTS`. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`,
templates at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.

## 1. The merge gate — unless `--bypass`

Two deterministic checks, both must pass:

- `work/<task-id>/verify.md` records the floor as `PASS`, not `FAIL`.
- `work/<task-id>/deviations.md` has zero `^- Status: open` lines
  (`grep -c`). An open deviation means a halt is still waiting on the
  human — shipping over it is exactly the silent-improvisation failure mode
  this system exists to prevent (build prompt §1, failure mode 7).

Either check failing: stop, tell the human specifically which check and
why, do not proceed to §2–6. This is not a touchpoint you invent — it's the
same plan-approval-adjacent judgment the human already exercised; you're
just not allowed to walk past a gate they haven't cleared.

**Multi-repo (Extension B): a third check.** If `plan.md` has a `##
Ship order` section, validate it against `workspace.json`'s contract
registry direction (open question §5.4 — declared in the plan, validated
here, never silently derived): for every contract `work/<task-id>/
artifacts/contract-touch.json` reports touched with `spec_change ==
"additive"`, if both the producer and at least one consumer appear in
`## Ship order`, the producer's position must come at or before every
such consumer's. A violation halts here — "ship order for '<name>' ships
the consumer before the producer, an additive change is never safe in
that direction" — same non-negotiable framing as the floor/deviation
checks above, not a soft warning.

**`--bypass <reason>`** skips both checks — loudly, never silently. Record
the bypass in the ledger (`ledger set <task-id> bypass "<reason>"`) and give
it its own visible section in the briefing (§4). Bypass is for a genuine
emergency (production down, the fix touches auth) that can't wait on the
harness — it is not a way to route around a check you disagree with.

## 2. Decisions

**Implement decisions this task's plan cited.** Read
`work/<task-id>/plan.md`'s `## Grounds on decisions` section (per
`core/templates/plan.md` — absent entirely if the plan cited none; skip
this part in that case). For each bullet there: resolve which store it
lives in — **multi-repo**, a bullet may be repo-qualified,
`<repo-name>:D-<seq>` (that member repo's own local `docs/decisions/`,
kept from before it joined the workspace or added since); unqualified
means the workspace root's own store (build prompt §2's "one system
charter at the workspace" extends naturally to workspace-level decisions,
but a member repo's pre-existing local decisions are never silently
absorbed into it). Read `<store-root>/docs/decisions/D-<seq>-*.md`, append
this task's *actual* diff paths from **that store's own repo**
(`git -C <store-root> diff --name-only <base>..HEAD`, not the plan's
predicted-touch list — the diff is what's real; for the workspace root's
own store this is an ordinary `git diff` at the workspace root) to its
`## Implementing paths` section, and if its `- Status:` line currently
reads `adopted`, flip it to `implemented` — a single-line edit, nothing
else in the file changes (`core/scripts/decision-hash` excludes that line
by design, specifically so this flip never invalidates a research.md that
already cites this decision). If the cited decision's status is already
`implemented` (a later task building on the same decision) or
`superseded`, still append this task's paths to `## Implementing paths`
but leave the status line alone — don't un-supersede a record or re-flip
an already-implemented one.

**Distill new decisions from this task's own deviations.** Read
`work/<task-id>/deviations.md`. For any *resolved* record whose resolution
establishes a rule future tasks should follow — not every trivial
record-and-proceed note, only ones a researcher on a later task would
actually want to find — write a new record from
`${CLAUDE_SKILL_DIR}/../../templates/decision.md`, citing this task ID. Do
the same for any halt-tier escalation resolved during this task. Skip this
part (write nothing) if nothing this task hit rises to that bar — a
decision record manufactured to have something to show is worse than none.

Allocate the next id — one more than the highest existing
`docs/decisions/D-<n>-*.md` (or `1` if none exist yet):

```
ls docs/decisions/D-*.md 2>/dev/null | sed -E 's|.*/D-([0-9]+)-.*|\1|' | sort -n | tail -1
```

Write `docs/decisions/D-<n>-<kebab-slug>.md`. Unlike a `/design`-authored
decision, a ship-distilled one is never written speculatively — the code
that prompted it already exists, as this task's own diff — so write it
directly as `- Status: implemented` (skip the `adopted` intermediate;
there is no window where a ship-distilled decision sits un-implemented),
`## Implementing paths` pre-filled from the same diff-path list used
above, and `## Alternatives rejected` reading "n/a — distilled from a
resolved deviation, see `work/<task-id>/deviations.md`" unless a real
alternative was genuinely weighed and rejected in the deviation's own
`Options considered` field.

## 3. Milestone done-definition

Read `work/<task-id>/milestone` (absent = this task isn't part of a
milestone — skip this step entirely, no note needed in the briefing). If
present, read `work/<id>/milestone.md`'s `## Member tasks` list. For this
task's own id, treat it as done — the merge gate (§1) already passed and
§6 is about to set `state` = `done`. For every *other* listed member task,
check its real `work/<other-task-id>/state`. If any entry is still `TBD`,
or any other member task's state isn't actually `done`, this isn't the
milestone's final ship — say nothing further, just note in passing (one
line, not a section) that member tasks remain. **If every member task is a
real id and every one is done** (by the rule above), this is the
milestone's completing ship: check the milestone's `## Done-definition`
against real state — for M0 specifically, that means every capability named in `##
Capability targets` is `implemented` in `.spine/capabilities.json` and
passes `adapter-conformance --all`. Report the result (met or not) as its
own section in the briefing (§4) — a milestone that ships its final task
without its done-definition actually being true is exactly the
"skeleton-skip" anti-pattern this build exists to make impossible, so
**do not let this pass silently**: if the done-definition isn't met, say
so plainly in the briefing rather than treating milestone completion as
automatic just because every member task individually shipped.

## 4. Write the delta briefing

`work/<task-id>/briefing.md` from
`${CLAUDE_SKILL_DIR}/../../templates/briefing.md`. ≤1 page. Pull capability
gaps straight from `verify.md`, tooling gaps straight from `verify.md`'s own
"Tooling gaps" section (never re-derive or re-summarize it — quote), deviations
straight from `deviations.md`, decisions from what §2 just produced (or
"none"), milestone done-definition result from §3 if this was a completing
ship (omit the section entirely otherwise — not every task belongs to a
milestone). **Multi-repo**: "Contracts touched" (omit entirely if
`contract-touch.json` reported nothing touched) — quote, per touched
contract, its `spec_change`/`registry_stale` and each gated consumer's
`contract-check` result straight from `verify.md`'s own "Contract
conformance" section (never re-derive), plus any undeclared-coupling
finding the falsifier's cross-repo mandate kept (build prompt §2:
"registry coverage made visible, so neglect is loud" — a touched contract
with zero findings and zero gaps is still worth its one line here, a clean
bill is not the same as an omitted section). If this ship used `--bypass`,
its own section here is not optional. The "what you'd want to know in six
months" line is the one line most worth spending real thought on — don't
let it default to a restatement of "what changed."

## 5. Ledger and commit

`ledger mark <task-id> ship` — per the rule in `core/skills/task/SKILL.md`
§6, this harvests the `verify` phase's window (its stored mark timestamp to
the `ship` timestamp you just wrote) into phase key `verify`. Then, since no
further mark follows `ship` in this task, harvest `ship` itself right now:
`ledger harvest <task-id> ship --transcript <path> --from <the ship mark
timestamp> --to <now>`. Then, **single-repo**:

```
git add -A -- <the task's actual changed paths, work/<task-id>/, docs/decisions/>
git commit -m "$(cat <<'EOF'
<subject line — follow this project's own commit-message convention if it
has one (e.g. Conventional Commits); the trailer below composes with any of
them, it never replaces the subject-line grammar>

<body, if useful>

Spine-Task: <task-id>
EOF
)"
```

**Multi-repo (Extension B) — staged, ordered, never partial-silent** (build
prompt §2): commit each repo named in `## Ship order`, in that exact
order, one at a time — never a single combined commit spanning repos (they
are separate git histories). Before the first commit, write
`work/<task-id>/state` = `shipping (1 of <n>)` at the **workspace root**
(one shared state file, one task). For each repo in order:

1. `git -C <repo-path> add -A -- <that repo's own changed paths>`.
2. `git -C <repo-path> commit -m "..."` — same subject/body/trailer shape
   as single-repo above, but the trailer is identical across every repo:
   `Spine-Task: <task-id>` (build prompt §2: "spine's existing linkage
   primitive does the cross-repo join" — this is the whole mechanism,
   nothing else ties the commits together).
3. Update `work/<task-id>/state` = `shipping (<k+1> of <n>)` at the
   workspace root immediately after each commit — this is what makes the
   inconsistency window **visible and bounded**, not eliminated (the
   commits are still separate events; nothing here makes them atomic). If
   this skill's own session is interrupted mid-sequence, a resumed session
   reads this state and knows exactly which repos already have their
   commit and which don't — resume from the next repo in `## Ship order`,
   never re-commit one already done, never skip one still pending.

Also commit the workspace root's own changes (`work/<task-id>/`, any
`docs/decisions/` or `contracts/` edits) as one more commit in the
sequence, at the position `## Ship order`'s reserved `workspace` entry
names (`core/templates/plan.md` — typically last, since its own artifacts
— briefing, verify.md — describe the completed change) — same
`Spine-Task:` trailer.

For `--bypass`, add `Spine-Bypass: <reason>` as its own trailer line
alongside (or instead of, if this was never a real task-folder task)
`Spine-Task:` — on every repo's commit in the multi-repo case, not just
one. Never omit both — that's the untracked-commit ratio
(`ledger scan-untracked-ratio`) existing specifically to catch, and it runs
per repo (`core/ADAPTER-CONTRACT.md §6`), so a repo whose commit is missing
the trailer is caught independently of its siblings having it.

Do not push, in either case. Committing locally is this skill's job;
pushing or opening a PR is the human's call, made after reading the
briefing.

## 6. Close out

Write `work/<task-id>/state` = `done` (this is the transition out of
`shipping (n of n)` for a multi-repo task — every repo's commit from §5
must have actually landed before this write, never write `done` while a
repo in `## Ship order` is still pending). Remove `.spine/current-task`
(the task is no longer active — a subsequent trivial edit should default
back to Class 0, not stay phase-gated against a finished task; for a
multi-repo task this file lives at the workspace root only — member repos
never had one). Tell the human where the briefing is — that read is the
third recurring touchpoint (build prompt §2.7), and it happens now, once,
not as a gate this skill enforced on itself.
