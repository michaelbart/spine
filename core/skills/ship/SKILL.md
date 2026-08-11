---
name: ship
description: Gate, commit, and brief a task that's passed verification. Invoked by /task at the ship phase, or directly as `/ship --bypass <reason>` for a genuine emergency.
disable-model-invocation: true
argument-hint: <task-id> [--bypass <reason>]
---

You are running `/ship` for task ID and optional `--bypass <reason>` from
`$ARGUMENTS`. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`,
templates at `${CLAUDE_SKILL_DIR}/../../templates/<name>`.

## 0. Ship-time re-grounding (Extension C §2.4)

The window between plan approval and ship is unguarded otherwise — a
neighbor's task can merge and invalidate this task's grounding after
`check-stale` already passed at plan time. Two re-runs, both recorded in
the ledger so their cost is measured, not guessed (build prompt's own
instruction):

```
${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md
```

If STALE: this is a real deviation, not a soft warning — append a
`work/<task-id>/deviations.md` record, tier `halt` (grounding drifted
since this was last verified, the same halt-tier build prompt §2.4
already assigns to schema/contract/auth surprises), and **it counts
toward the circuit breaker** (`core/skills/task/SKILL.md`'s existing
three-deviation rule) — decided and defended here, not left as an open
question: neighbor-caused drift isn't this plan's own fault, but the
circuit breaker's actual trigger condition is "the research this plan
stands on is no longer trustworthy," which is exactly as true when a
neighbor invalidated it as when the original research was simply wrong.
Treating it differently would need a second, parallel invalidation
channel this system doesn't have and shouldn't grow one just for this.
`ledger set <task-id> ship_time_regrounding "check-stale: stale"` (or
`"ok"`) before proceeding — proceeding means going back to
`core/skills/task/SKILL.md` step 2 (research), not continuing here.

If ok: `git pull --rebase` onto the current mainline (single-repo: this
project; multi-repo: the workspace root, then each repo in `## Ship
order`), then re-run the floor:

```
${CLAUDE_SKILL_DIR}/../../scripts/floor <class> --task <task-id> \
  --out work/<task-id>/artifacts/floor-result-postrebase.json
```

The second merger always re-verifies against the first's reality — this
is what makes that literally true instead of aspirational. If this
post-rebase floor fails, that's a real merge-gate failure (§1 below), not
a deviation — the diff itself now conflicts with what actually landed.
`ledger set <task-id> ship_time_regrounding "floor: pass"` (or `"fail:
<capability>"`).

**Third re-grounding check: the real diff against other open tasks'
claims** (Phase D self-red-team finding — narrow-claims verification).
The plan-time `claims-check` (`task/SKILL.md` §3) only ever sees this
task's own *declared* `predicted_touch` — a task that under-declared its
claims to dodge a real conflict was invisible to it by construction. This
closes that specific hole against the diff that actually exists now:

```
${CLAUDE_SKILL_DIR}/../../scripts/claims-check <task-id> --project <project root> --diff <base-ref>
```

(multi-repo: once per repo in `## Ship order`, `--project <repo-path>`,
`<base-ref>` that repo's own pre-task sha). Any `[UNDECLARED]`-tagged
result is the defeat itself, caught, not a hypothetical — append a
`deviations.md` record, tier `halt`, same as a stale `check-stale` result
above, and it counts toward the circuit breaker the same way. A
`[DECLARED]`-tagged hit means the plan-time check should have caught this
already and didn't — most likely the colliding task's own claims changed
after this one's plan was approved without `propagate` reaching this
task; record it the same way, but note the distinction in the deviation
record rather than treating both as the identical failure mode.
`ledger set <task-id> ship_time_regrounding_claims "clear"` (or
`"undeclared: <n>"` / `"declared: <n>"`) — a third, distinct field from
the two above, so `/costs` can eventually tell which of the three
re-grounding checks is actually catching things.

**Honest limit, stated plainly rather than implied by silence**: this is
discovered late (ship time, code already written) and is not as strong as
plan-time prevention — reverting a real diff costs more than declining to
approve a plan. It is real, mechanical, and specific (names the exact
undeclared path and the exact colliding task), which is the distinction
that matters — before this check existed, the identical defeat surfaced
only as a generic `conformance` precision drop with no link back to which
other task it endangered, indistinguishable from an ordinary, harmless
scope change.

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

**Class 2 — a third check, second approver (Extension C §2.6):**

```
${CLAUDE_SKILL_DIR}/../../scripts/second-approver-check <task-id> --project <project root>
```

Exit 1 halts here, verbatim message. Exit 0: read its stdout — an
`override` result is not a quiet pass, `ledger set <task-id>
second_approver "override: <reason>"` and it gets its own unconditional
line in the briefing (§4), same visibility standard as `--bypass`. A real
second-approver result: `ledger set <task-id> second_approver "<approver
identity>"` and proceed normally — expected, non-remarkable, still
recorded for `/costs`' per-engineer view but not called out as loudly in
the briefing.

**`--bypass <reason>`** skips every check above (floor/deviations, ship
order, second-approver) — loudly, never silently. Record the bypass in the
ledger (`ledger set <task-id> bypass "<reason>"`) and give it its own
visible section in the briefing (§4). Bypass is for a genuine emergency
(production down, the fix touches auth) that can't wait on the harness —
it is not a way to route around a check you disagree with. `--bypass` does
not skip §0's ship-time re-grounding — that runs first, unconditionally;
what it skips is acting on a stale result as a hard halt.

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
`${CLAUDE_SKILL_DIR}/../../templates/briefing.md`, its prose per the
writing mandate at
`${CLAUDE_SKILL_DIR}/../../templates/writing-mandate.md`. ≤1 page, hard.
Section by section, each sourced only from what's already been produced —
this file quotes, it doesn't re-derive:

- **What & why** / **What surprised us**: one or two sentences on what's
  now true and why; deviations straight from `deviations.md`, one line
  each, resolution included ("Nothing — the plan held." if none).
- **Verification, honestly** bullets — `Floor` from `verify.md`'s floor
  result; `Re-grounding` quoted from what §0 recorded ("check-stale: ok,
  floor re-run: pass" is the unremarkable case, still shown); `Adversaries`
  as count + max severity + one-line gist each, pointer to `verify.md`,
  never compressed further (this file's own header note); `Capability
  gaps` straight from `verify.md`; `Tooling gaps` straight from
  `verify.md`'s own "Tooling gaps" section (never re-derive or
  re-summarize — quote); `Plan accuracy` from `conformance.json`'s score,
  in words. `Approver` (Class 2 only, omit for Class 0/1): the
  second-approver identity from `second-approver-check`'s real-approver
  result — omit this bullet (not the fact) when an override put it in
  "Overrides & bypasses" instead.
- **Contracts** (multi-repo only, omit entirely if `contract-touch.json`
  reported nothing touched): per touched contract, its
  `spec_change`/`registry_stale` and each gated consumer's `contract-check`
  result straight from `verify.md`'s own "Contract conformance" section
  (never re-derive), plus any undeclared-coupling finding the falsifier's
  cross-repo mandate kept (build prompt §2: "registry coverage made
  visible, so neglect is loud" — a touched contract with zero findings and
  zero gaps is still worth its one line, a clean bill is not the same as
  an omitted section).
- **Overrides & bypasses** (omit entirely if none occurred): `--bypass`'s
  own line is not optional when used; a plan-time claims-check override
  (`work/<task-id>/deviations.md`'s own record of it, per
  `core/skills/task/SKILL.md` §3) gets its own line here too; a
  second-approver self-approval override (`.override == true` in
  `approval.json`) gets its own line with the override reason, same
  visibility standard as `--bypass` — never folded into a single
  "approvals" line that could bury it. A ship-time `claims-check --diff`
  `[UNDECLARED]` collision is a halt-tier deviation, not an override —
  it belongs in "What surprised us," not here.
- **Milestone** (omit entirely if this task isn't part of one): §3's
  result — which milestone, and (only on the completing ship) whether its
  Done-definition is actually met by real state, said plainly either way.
- **In six months you'll want to know** is the one line most worth
  spending real thought on — don't let it default to a restatement of
  "What & why."
- **Record**: `work/<task-id>/` — the pointer into the full artifacts this
  briefing summarized, always present, last line.

## 4a. Write the PR description

Same moment as §4 (same sources, before the commit), because this is what
a reviewer reads instead of scrolling the raw diff — the merge gate has
already passed, so the record this reads from is final for this ship.
`work/<task-id>/pr-description.md` from
`${CLAUDE_SKILL_DIR}/../../templates/pr-description.md`. Its own header
comment is the single source of the four-unions "Where to look" rule and
the completeness-line rule — don't re-derive or restate that logic here;
read the template, follow it exactly, including its ratchet-trigger note
on the deviations.md extraction heuristic. Every other section reads
straight from the same artifacts §4 just finished reading (plan.md,
deviations.md, verify.md, the ledger, approval.json) plus three §4 doesn't
need: `work/<task-id>/artifacts/conformance.json` (the real predicted/
actual file lists, not verify.md's count-only summary line),
`work/<task-id>/artifacts/<agent>-verdict.json` for each adversary that ran
(the post-`verdict-filter` kept verdicts, for their structured
`evidence.file`/`evidence.line`), and `.spine/protected-paths.conf` against
the real diff (`git diff --name-only <base>..HEAD` — the same base §0's
ship-time floor re-run used).

**This file is never a summary of the diff.** If you catch yourself about
to read the changed source files to describe what they do, stop — that's
the post-hoc-summary failure mode this step exists to prevent. Every
sentence traces to plan.md, deviations.md, verify.md, the ledger, or one
of the three artifacts above; nothing here is generated by re-reading code.

**Multi-repo (Extension B):** one shared `pr-description.md`, written once
at the workspace root — not one per repo (§2 of this patch's own
`work/.build/pr-patch-phase-A-handoff.md` records why: the record it reads
from is task-scoped, not repo-scoped). Its `**Contracts**` section carries
`## Ship order` plus each contract's `contract-touch.json` result. Omit the
whole section on a single-repo task, or a multi-repo task whose
`contract-touch` run found nothing touched.

`ledger set <task-id> pr_description "generated"` — one field, so a future
`/costs` view can notice a ship that skipped this step (the ledger key is
freeform; this adds no new schema to `core/scripts/ledger`).

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

**Propagate** (Extension C §2.5) — this task's own commit(s) just changed
files (and possibly decisions/contracts) other open tasks may ground on:

```
git diff --name-only <base>..HEAD -- . | \
  ${CLAUDE_SKILL_DIR}/../../scripts/propagate <task-id> --project <project root> \
    --decisions <comma-list from plan.md's ## Grounds on decisions, if any> \
    --contracts <comma-list from verify.md's Contract conformance section, if any>
```

Multi-repo: run once per repo actually committed in `## Ship order`
(`--project <repo-path>`, changed paths repo-qualified to match
`claims.json`'s own convention), plus once at the workspace root for its
own commit. This never blocks the ship — it's informational at ship time,
the same way `contract-touch` is; what it writes (flags in *other* tasks'
folders) is what later blocks *their* phase advance, via
`core/skills/task/SKILL.md`'s flag-blocked-advance check, not this one.

Do not push, in either case. Committing locally is this skill's job;
pushing or opening a PR is the human's call, made after reading the
briefing.

## 6. Close out

Write `work/<task-id>/state` = `done` (this is the transition out of
`shipping (n of n)` for a multi-repo task — every repo's commit from §5
must have actually landed before this write, never write `done` while a
repo in `## Ship order` is still pending). `registry-sync <task-id>` —
this task's own final registry write; a `done` task no longer participates
in `claims-check`/`propagate`/`/tasks`' open-task scan (all three skip
by `state`), so this is what actually removes it from the shared
registry's live view, not just from this machine's local one. Remove
`.spine/current-task`
(the task is no longer active — a subsequent trivial edit should default
back to Class 0, not stay phase-gated against a finished task; for a
multi-repo task this file lives at the workspace root only — member repos
never had one). Tell the human where the briefing is, and where
`pr-description.md` is (§4a) — pushing and opening the PR is their call,
made after reading both. That read is the third recurring touchpoint
(build prompt §2.7), and it happens now, once, not as a gate this skill
enforced on itself.
