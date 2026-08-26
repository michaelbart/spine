---
name: ship
description: Gate, commit, and brief a task that's passed verification — re-grounds against what changed underfoot since research, runs the merge gate, distills/updates decisions, does milestone bookkeeping (flagged-finding triage, done-definition check, known-gap resolution), writes the delta briefing and PR description, and (multi-repo) drives a resumable staged commit across repos. Invoked by /task at the ship phase, or directly as `/ship --bypass <reason>` for a genuine emergency.
disable-model-invocation: true
argument-hint: <task-id> [--bypass <reason>]
---

You are running `/ship` for task ID and optional `--bypass <reason>` from
`$ARGUMENTS`. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`,
templates at `${CLAUDE_SKILL_DIR}/../../templates/<name>`. Hand
`${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim, `../../` included —
do **not** lexically collapse it to `.claude/`; `.claude/skills/ship` is a
symlink into the spine core checkout, and collapsing the text yields a
nonexistent `.claude/scripts/...` path.

**This skill does six distinct jobs, in order — not one "gate and commit"
step.** A single-repo, non-milestone, guided task still passes through all
six; the ones that only apply conditionally say so inline. Use this as a
map, not a summary — each job's own section is where the real detail
lives:

1. **§0 Ship-time re-grounding** — did anything this task's plan grounded
   on change underfoot since research/plan was written.
2. **§1 The merge gate** — the deterministic floor, zero open
   `deviations.md` entries, unless `--bypass`.
3. **§2 Decisions** — distill or update `docs/decisions/` records from
   what this task actually taught.
4. **§3 Milestone bookkeeping** (only if this task belongs to one) —
   flagged-finding triage (§3a), the done-definition check on the final
   member task (§3b), known-gap resolution (§3c).
5. **§4/§4a Briefing and PR description** — the one-page human-facing
   summary, and (if `open-pr` applies) the PR body.
6. **§5/§6 Ledger, commit, and close-out** — including, multi-repo, a
   resumable staged commit sequence across repos.

## 0. Ship-time re-grounding (Extension C §2.4)

The window between plan approval and ship is unguarded otherwise — a
neighbor's task can merge and invalidate this task's grounding after
`check-stale` already passed at plan time. Two re-runs, both recorded in
the ledger so their cost is measured, not guessed:

```
${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md
```

**Before treating STALE as real drift, cross-check each drifted file
against this task's own record** (disclosed fix — the naive "any STALE
verdict is drift" rule produces a guaranteed false positive on every task
that touches a file it also read as grounding, which is most tasks). A
drifted file is
**expected, not drift**, if any of the following is true: (a) it appears
in `work/<task-id>/plan.md`'s own `## Predicted touch` list — this task's
own approved implementation changed it, not a neighbor; (b) it's
explained by a `work/<task-id>/deviations.md` record with `Status:
resolved` (e.g. a manifest file changed because an approved
new-dependency deviation added one); or (c) it's named in
`work/<task-id>/verify.md`'s own "Adversary verdicts" section against a
kept finding marked `FIXED` there — an adversary-found fix applied and
recorded during `/verify` is exactly as "this task's own approved work,
not a neighbor's" as (a)/(b), and forcing every such fix through
`deviations.md` too would make the circuit breaker fire on legitimate,
already-adversarially-verified work with nothing left to stash (disclosed
fix — see docs/tradeoffs.md); or (d) it is `work/<id>/milestone.md` for the
milestone this task itself belongs to (`work/<task-id>/milestone` names
`<id>`), and every changed line `git diff <sha> -- work/<id>/milestone.md`
shows traces to this task's own mandated bookkeeping in that shared file —
the classify-time replacement of the milestone's first `TBD` member-task
line with this task's own id (`core/skills/task/SKILL.md`'s `--milestone`
header note, made before research even ran) and/or a `## Known gaps for
future member tasks` entry whose `source` field cites this task's own
`work/<task-id>/verify.md` (the edit §3a below makes at ship time,
possibly made early during `/verify` under the same triage convention). A
milestone-tagged task's `research.md` citing `work/<id>/milestone.md` as
grounding — the expected case, since `core/skills/task/SKILL.md` loads it
as planning context for research to read — makes this guaranteed on the
classify-time edit alone, on every such task, not an edge case (disclosed
fix). A changed
line that isn't one of those two things — another member task's own `TBD`
slot resolved, a different task's Known-gaps entry, an edit to `##
Inter-task contracts` or `## Capability targets` — is a neighbor's real
change and stays fully driftable; (d) accounts for only this task's own
hand in a shared file, never the file wholesale. Only a file covered by
**none of the four** is genuine unexplained drift. If every drifted file
is expected by this rule: `ledger set <task-id> ship_time_regrounding_check_stale
"stale (expected — matches
predicted-touch/deviations/verify-fixed/own-milestone-edit)"` and proceed
normally — not a halt, not a deviation, does not touch the circuit
breaker.

If any drifted file is **not** covered by any of these checks: this is a real
deviation, not a soft warning — append a
`work/<task-id>/deviations.md` record, tier `halt` (grounding drifted
since this was last verified, the same halt tier already assigned to
schema/contract/auth surprises), and **it counts
toward the circuit breaker** (`core/skills/task/SKILL.md`'s existing
three-deviation rule) — decided and defended here, not left as an open
question: neighbor-caused drift isn't this plan's own fault, but the
circuit breaker's actual trigger condition is "the research this plan
stands on is no longer trustworthy," which is exactly as true when a
neighbor invalidated it as when the original research was simply wrong.
Treating it differently would need a second, parallel invalidation
channel this system doesn't have and shouldn't grow one just for this.
`ledger set <task-id> ship_time_regrounding_check_stale "stale"` (or
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
`ledger set <task-id> ship_time_regrounding_floor "pass"` (or `"fail:
<capability>"`) — a fourth, distinct field alongside `_check_stale`,
`_claims`, and `_index` below, so a ship that hits more than one
re-grounding check keeps every result instead of the last write winning.

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
the two above, so `/task-report` (`render-task`'s "Conformance, approval &
re-grounding" section) shows which of the three re-grounding checks fired
on this task, not just whether ship-time re-grounding happened at all.

**Honest limit, stated plainly rather than implied by silence**: this is
discovered late (ship time, code already written) and is not as strong as
plan-time prevention — reverting a real diff costs more than declining to
approve a plan. It is real, mechanical, and specific (names the exact
undeclared path and the exact colliding task), which is the distinction
that matters — before this check existed, the identical defeat surfaced
only as a generic `conformance` precision drop with no link back to which
other task it endangered, indistinguishable from an ordinary, harmless
scope change.

**Fourth re-grounding check: the decision index, against whatever
`docs/decisions/` looks like right now.** §2 below regenerates
`docs/decisions/INDEX.md` for *this* task's own decision edits, but a
neighbor's task (or a hand-edit, or a `--bypass`) can have changed the
store since without regenerating it — the same staleness shape
`check-stale` exists to catch for `research.md`, one layer over:

```
${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <project root> --check
```

(multi-repo: once per repo in `## Ship order` that has its own
`docs/decisions/`, `--project <repo-path>`.) A `STALE` result here is
never this task's own fault by construction — §2 hasn't run yet at this
point in the skill — so it is not a deviation and does not touch the
circuit breaker; it's a stale, inherited artifact this task is about to
fix anyway once §2's regeneration runs. Note it in the ledger
(`ledger set <task-id> ship_time_regrounding_index "stale (regenerating in
§2)"` or `"ok"`) so a human reading `/task-report` for this task can see
whether the index was stale at ship time, and move on — §2's own
regeneration (which runs
unconditionally whenever this task touched `docs/decisions/`, and should
also run here if it's stale for a reason unrelated to this task, e.g. a
neighbor's un-regenerated ship) is what actually resolves it before this
task's own commit.

## 1. The merge gate — unless `--bypass`

Two deterministic checks, both must pass:

- `work/<task-id>/verify.md` records the floor as `PASS`, not `FAIL`.
- `work/<task-id>/deviations.md` has zero `^- Status: open` lines
  (`grep -c`). An open deviation means a halt is still waiting on the
  human — shipping over it is exactly the silent-improvisation failure mode
  this system exists to prevent (failure mode 7).

Either check failing: stop, tell the human specifically which check and
why, do not proceed to §2–6. This is not a touchpoint you invent — it's the
same plan-approval-adjacent judgment the human already exercised; you're
just not allowed to walk past a gate they haven't cleared.

**Multi-repo (Extension B): a third check.** If `plan.md` has a `##
Ship order` section, validate it against `workspace.json`'s contract
registry direction (declared in the plan, validated here, never silently
derived): for every contract `work/<task-id>/
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
`override` result is not a quiet pass, and gets its own unconditional line
in the briefing (§4), same visibility standard as `--bypass`. Either way
(override or a real approver), record it with `ledger record-second-approver
<task-id> --project <project root>` — this derives the value from
`approval.json` itself (same fields `second-approver-check` reads) and writes
it inside the script, never in this command's own arguments. Use this instead
of a raw `ledger set <task-id> second_approver "..."` call: a self-approval
override's reason is real and human-authorized, but a `ledger set` command
whose literal text contains "override"/"self-approved"/"authorized" reads
identically to an agent narrating its own bypass in progress to any
safety classifier scanning Bash commands — solo-maintainer projects hit
Class 2's self-approval path on every ship, so this is a recurring false
positive, not a one-off, and `record-second-approver` exists specifically
to route around it without changing what gets recorded. An override result
is still expected to be loud in the briefing; a real second-approver result
is expected, non-remarkable, still recorded for `/costs`' per-engineer view
but not called out as loudly in the briefing.

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
means the workspace root's own store ("one system charter at the
workspace" extends naturally to workspace-level decisions, but a member
repo's pre-existing local decisions are never silently absorbed into it).
Read `<store-root>/docs/decisions/D-<seq>-*.md`, append
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

**Regenerate the decision index.** If this task touched `docs/decisions/`
at all above — a status flip, an `## Implementing paths` append, or a
newly-distilled record — regenerate its store's index before this task's
commit(s) in step 5:

```
${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <store-root>
```

For a repo-qualified decision, `<store-root>` is that member repo's own
path, not the workspace root — same split `check-stale`/`decision-hash`
already use for repo-qualified citations. Skip entirely if this task
cited no decision and distilled none — the index doesn't need
regenerating when the store it summarizes hasn't changed. Mechanical, no
review needed.

## 3. Milestone bookkeeping

Read `work/<task-id>/milestone` (absent = this task isn't part of a
milestone — skip §3a/§3b/§3c entirely, no note needed in the briefing). If
present, resolve `work/<id>/milestone.md` using the same probe order the
task skill used at creation time: **if `workspace.json` exists at the
project root**, check (1) `work/<id>/milestone.md` at the project root,
then (2) `<member-repo-path>/work/<id>/milestone.md` for each repo in
`workspace.json`'s `repos` array in order; use the first path found. **If
`workspace.json` is absent**, use `work/<id>/milestone.md` at the project
root. All three steps below run against the resolved path — §3a and §3c on
*every* member-task ship, §3b only on the milestone's completing ship. §3a
and §3c can run in either order — they touch the same section but never
the same entries (§3a only ever allocates new ids off the monotonic
`next-gap-id` counter, §3c only ever removes ids this task's own plan
cited), so there's no ordering hazard between them.

### 3a. Flagged-finding triage

The routing gap this step closes: an adversary-confirmed, cross-task-relevant finding that gets
deliberately flagged rather than fixed in this task's own `/verify` pass
has, until now, had no path into the one place a future member task's
planning actually looks (`## Inter-task contracts`/`## Known gaps` in
`milestone.md`, loaded by `core/skills/task/SKILL.md`'s `--milestone`
handling) — it stayed fully documented in this task's own `verify.md`/
`notes.md` and fully invisible to whoever plans the task that needs it.

**Gather.** Read every `work/<task-id>/artifacts/<agent>-verdict.json` this
task's `/verify` produced (falsifier always, security per §2's adversary
count). Collect every kept verdict whose `disposition`
(`core/ADAPTER-CONTRACT.md §5`) is `"not_fixed"` or absent — absent is
never treated as resolved, same discipline as everywhere else in this
system a gate could otherwise silently read as passing.

**Dedup against what's already tracked.** Before asking anything, read
`work/<id>/milestone.md`'s `## Known gaps for future member tasks` section
(absent or empty on this milestone's first ship — nothing to dedup
against yet). **Before treating any existing entry as well-formed, verify
its shape:** each entry must be a fenced block with an `id: gap-<n>` line
matching `core/templates/milestone.md`'s own format, and the file must
have a `<!-- next-gap-id: N -->` counter. If any entry is plain prose
(no fence, no `id:` line) or the counter is missing, **flag it to the
human before continuing** — report each malformed entry by quoting its
text, explain that it can't be machine-cited by future tasks, and ask
whether to reformat it into the proper `gap-<n>` shape now (content
unchanged, prose → fenced block) or leave it as-is and note it in
`notes.md` as non-machine-citable. Never silently absorb malformed entries
as if they were well-formed — the carry-forward mechanism degrades
invisibly if you do. For each well-formed candidate, check whether its
`claim`/`evidence.file` substantially matches an existing `gap-<n>`
entry's `source` field (a cheap containment check against the
machine-fenced block, not semantic matching — same fidelity `check-stale`'s
own file-drift comparison already uses). A match: don't include it in the
question below; instead record in `notes.md`, "already tracked as
`gap-<n>`, not re-asked" — a suppressed question is still a decision, and
must read differently from a finding nobody ever looked at. This is what
keeps two sibling member tasks that independently trip the same underlying
gap from re-triaging it twice.

**Zero candidates remain** (nothing was flagged, or everything flagged is
already tracked): nothing further to do, no section in the briefing (§4)
— this is a gate that correctly never applied, not a degraded one.

**One or more candidates remain — branch on `work/<task-id>/autonomy`**
(absent = `guided`, same convention §5's PR-opening step already uses):

- **`guided`** — ask now, interactively, before proceeding to §3b/§4: for
  each candidate, show severity + claim + evidence pointer (file:line or
  command), and ask which should carry into `milestone.md`'s Known gaps
  for future member tasks to see — "none" is a complete, valid answer, not
  a thing to talk the human out of. This blocks the same way plan approval
  and the Class 2 second-approver stop already block; it is not a merge
  gate (§1's two checks are unchanged, adversary findings still never fail
  `/verify` by that skill's own report step), just a question that has to
  be asked before this task's ship completes.
- **`checkpointed` / `auto`** — no scheduled stop exists here, so don't
  manufacture one. Draft the candidate entries (same shape the "apply the
  human's picks" step below produces for `guided`) into a new "Proposed
  milestone gap entries — undecided" section of the briefing (§4) and the
  PR description (§4a),
  explicitly not yet applied to `milestone.md`. The human's post-hoc PR
  review — the same relocated touchpoint these autonomies already use for
  plan review — is where these get triaged, by hand-editing `milestone.md`
  or leaving them. Never write to the always-loaded `milestone.md` without
  a human having actually looked, whether that look happens now (`guided`)
  or at PR review.

**For `guided`, apply the human's picks now.** For each carried finding:
allocate the next id from `milestone.md`'s own `next-gap-id` counter
(`core/templates/milestone.md`'s comment — a monotonic counter, never
"highest id currently present," so a gap-<n> §3c already removed this
milestone's history is never reused for something unrelated), then
increment that counter in the file. Append a new fenced entry per
`core/templates/milestone.md`'s own comment — `source` = this task's
`verify.md` path plus the agent/severity, prose drafted from the verdict's
own `claim`/`evidence` plus `milestone.md`'s `## Member tasks` list (never
copied verbatim from `verify.md`'s adversary-voice prose, which is written
for an attacker's audience, not a future planner's). For each declined
finding: record the decision and its stated reason in `notes.md` — a
finding the human looked at and declined must read differently,
permanently, from one nobody ever asked about.

### 3b. Milestone done-definition

Read `work/<id>/milestone.md`'s `## Member tasks` list. For this task's
own id, treat it as done — the merge gate (§1) already passed and §6 is
about to set `state` = `done`. For every *other* listed member task, check
its real `work/<other-task-id>/state`. If any entry is still `TBD`, or any
other member task's state isn't actually `done`, this isn't the
milestone's final ship — say nothing further, just note in passing (one
line, not a section) that member tasks remain. **If every member task is a
real id and every one is done** (by the rule above), this is the
milestone's completing ship: check the milestone's `## Done-definition`
against real state, **live, right now** — never trust
`.spine/capabilities.json`'s `implemented` flag by itself, since that flag
can go stale between whenever some earlier task set it and this exact
ship (state reads `done`, the flag reads `implemented`, and the real
capability is still broken from a cold start — a real gap a downstream
project's own M1 completion surfaced: the only reason it was caught at all
was a human asking "what's next" and reading a capability's adapter by
hand). If `## Capability targets` lists any capability, re-run conformance
for real, this exact moment, not a cached record of some earlier run:

```
${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all \
  --project <project root> \
  > work/<task-id>/artifacts/done-definition-conformance.txt 2>&1
```

Use *this* live result, not the cached `capabilities.json` flag, to decide
whether each listed capability is genuinely met right now —
`adapter-conformance --all` already exercises each capability's own
pass/fail/cold self-test scenarios, and `cold` specifically proves
smoke-seed/smoke-run/smoke-golden self-heal a torn-down stack rather than
assuming an earlier capability in the sequence left it running
(`core/ADAPTER-CONTRACT.md §2.2/§3.8`) — this is what actually catches the
"provably true from scratch," not "reported true once," distinction the
gap above turned on. If it could not run at all, this is the
could-not-run case in the tooling-gap discipline (`core/skills/task/
SKILL.md`'s header note): note the gap and say plainly in the briefing
that the done-definition is **unverified**, not met — never let a
could-not-run check silently read as passing. Report the result (met, not
met, or unverified) as its own section in the briefing (§4) — a milestone
that ships its final task without its done-definition actually being true
is exactly the "skeleton-skip" anti-pattern this build exists to make
impossible, so **do not let this pass silently**: if the done-definition
isn't met (or couldn't be checked), say so plainly in the briefing rather
than treating milestone completion as automatic just because every member
task individually shipped.

After reporting the done-definition result, check whether
`work/M<n+1>/milestone.md` exists (where `<n>` is this milestone's
number). If it does, say nothing — the next milestone is already planned.
If it does not, add one line to the briefing's **Milestone** section:
`"M<n> complete. No M<n+1> defined yet — run /roadmap to plan the next
slice."` This is purely informational, never a gate.

### 3c. Known-gap resolution

Read `work/<task-id>/plan.md`'s `## Resolves known gaps` section
(`core/templates/plan.md`, absent entirely if this plan cited none — skip
this step in that case, same as `## Grounds on decisions` in §2). For each
`gap-<n>` bullet there, remove that exact fenced entry from
`work/<id>/milestone.md`'s `## Known gaps for future member tasks` — find
by id, delete only that entry, leave `next-gap-id` and every other entry
untouched (never renumber remaining entries; a gap's id is permanent once
allocated, same reasoning `core/templates/milestone.md`'s own comment
gives for never reusing one). If a cited `gap-<n>` doesn't actually exist
in `milestone.md` (a stale citation, or a typo in the plan), don't fail
the ship over it — note it in `notes.md` ("plan cited gap-<n>, not found
in milestone.md — nothing removed") and move on; a plan-time citation
error is a plan-quality issue for a future human reader to notice, not a
merge-gate concern (§1's two checks are unchanged). This is deliberately
the mirror of §2's decision-status-flip mechanic: find by id, edit exactly
that one thing, nothing else in the file changes.

## 4. Write the delta briefing

`work/<task-id>/briefing.md` from
`${CLAUDE_SKILL_DIR}/../../templates/briefing.md`, its prose per the
writing mandate at
`${CLAUDE_SKILL_DIR}/../../templates/writing-mandate.md`. ≤1 page, hard —
operationalized as ≤60 lines (same convention `CLAUDE.md`'s own cap uses).
**This template is one unwrapped paragraph per bold label, so a line count
alone is a weak signal of page length here** (unlike `plan.md`, which wraps
at ~72 chars and so has a line count that actually tracks length) — a
briefing can run several times a fair page's worth of prose and still show
comfortably under 60 lines. Treat **~600 words** as the real "over cap"
signal for this file; record both. `wc -l < briefing.md` and `wc -w <
briefing.md` it once written (redirect stdin, not `wc -l briefing.md` —
the latter's filename suffix gets stored verbatim by `ledger set` and
breaks the numeric cap check downstream) and record both counts (`ledger
set <task-id> briefing_line_count <n>` and `ledger set <task-id>
briefing_word_count <n>`) — this cap has no other backstop, so the
recorded counts are what make an oversized briefing visible in
`/task-report` rather than only ever self-checked in the moment. Over cap
(either signal) means trim before shipping, not ship anyway. Section by
section, each sourced only from what's already been produced — this file
quotes, it doesn't re-derive:

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
  re-summarize — quote); `Setup events` straight from `verify.md`'s own
  "Setup events" section, same quote-don't-re-derive rule — distinct from
  `deviations.md`-sourced "What surprised us" above, per
  `core/skills/task/SKILL.md` §4's boundary test; `Plan accuracy` from
  `conformance.json`'s score, in words. `Approver` (Class 2 only, omit for
  Class 0/1): the
  second-approver identity from `second-approver-check`'s real-approver
  result — omit this bullet (not the fact) when an override put it in
  "Overrides & bypasses" instead.
- **Contracts** (multi-repo only, omit entirely if `contract-touch.json`
  reported nothing touched): per touched contract, its
  `spec_change`/`registry_stale` and each gated consumer's `contract-check`
  result straight from `verify.md`'s own "Contract conformance" section
  (never re-derive), plus any undeclared-coupling finding the falsifier's
  cross-repo mandate kept — registry coverage made visible, so neglect is
  loud: a touched contract with zero findings and zero gaps is still
  worth its one line, a clean bill is not the same as an omitted section.
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
- **Milestone** (omit entirely if this task isn't part of one): §3b's
  result — which milestone, and (only on the completing ship) whether its
  Done-definition is actually met by real state, said plainly either way.
  §3a's result folds in here too: which findings (if any) were carried
  into `milestone.md`'s Known gaps, with their new `gap-<n>` ids
  (`guided`), or the drafted "Proposed milestone gap entries — undecided"
  list awaiting the human's PR-time triage (`checkpointed`/`auto`) — never
  omitted just because §3a found nothing to carry; "zero flagged findings"
  and "N findings, none carried" are different facts and this line says
  which one happened.
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
at the workspace root — not one per repo (the record it reads
from is task-scoped, not repo-scoped). Its `**Contracts**` section carries
`## Ship order` plus each contract's `contract-touch.json` result. Omit the
whole section on a single-repo task, or a multi-repo task whose
`contract-touch` run found nothing touched.

`ledger set <task-id> pr_description "generated"` — one field, folded into
`ledger aggregate`'s `pr_description_count` and reported by `/costs`, so a
ship that skipped this step is visible against `task_count` rather than
silently absorbed.

**Same over-cap tracking as §4's briefing, same reason**: this template
shares briefing.md's one-unwrapped-paragraph-per-label format (its own
header comment says so), and in practice runs *longer*, not shorter — its
reader has less context than briefing's, and its extra sections
(`Review this at the plan level`, `Where to look`) add real length. `wc -l
< pr-description.md` and `wc -w < pr-description.md` (redirect stdin, same
reason as §4), record both (`ledger set <task-id>
pr_description_line_count <n>` and `ledger set <task-id>
pr_description_word_count <n>`), same ~600-word real "over cap" signal.
Over cap means trim before shipping, not ship anyway.

## 5. Ledger and commit

`ledger mark <task-id> ship` — per the rule in `core/skills/task/SKILL.md`
§6, this harvests the `verify` phase's window (its stored mark timestamp to
the `ship` timestamp you just wrote) into phase key `verify`. Then, since no
further mark follows `ship` in this task, harvest `ship` itself right now:
`ledger harvest <task-id> ship --transcript <path> --from <the ship mark
timestamp> --to <now>`.

**Derive the ticket, add its trailer.** **Prefer `work/<task-id>/ticket`** if
present (recorded by `/intake`); otherwise run
`${CLAUDE_SKILL_DIR}/../../scripts/ledger ticket-from-branch --project <project
root>`. If either yields a key, add `Spine-Ticket: <key>` as an additional trailer
line on this task's commit(s) — every repo, in the multi-repo case — alongside
`Spine-Task:`, per `core/ADAPTER-CONTRACT.md` §6's composing-trailer rule. If it
prints nothing (off-ticket), omit that line.

Then, **single-repo**:

```
git add -A -- <the task's actual changed paths, work/<task-id>/, docs/decisions/>
git commit -m "$(cat <<'EOF'
<subject line — follow this project's own commit-message convention if it
has one (e.g. Conventional Commits); the trailer below composes with any of
them, it never replaces the subject-line grammar>

<body, if useful>

Spine-Task: <task-id>
Spine-Ticket: <ticket-key>
EOF
)"
```

**Multi-repo (Extension B) — staged, ordered, never partial-silent**:
commit each repo named in `## Ship order`, in that exact order, one at a
time — never a single combined commit spanning repos (they
are separate git histories). Before the first commit, write
`work/<task-id>/state` = `shipping (1 of <n>)` at the **workspace root**
(one shared state file, one task). For each repo in order:

1. `git -C <repo-path> add -A -- <that repo's own changed paths>`.
2. `git -C <repo-path> commit -m "..."` — same subject/body/trailer shape
   as single-repo above, but the trailer is identical across every repo:
   `Spine-Task: <task-id>` — spine's existing linkage primitive does the
   cross-repo join; this is the whole mechanism, nothing else ties the
   commits together.
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

**Pushing and opening the PR is autonomy-aware, gated by the team profile**
(read `work/<task-id>/autonomy`, `core/skills/task/SKILL.md`, absent = `guided`;
and `.spine/profile.json`'s `pr_open`, `core/ADAPTER-CONTRACT.md` §7, absent =
`auto-checkpointed`). `pr_open` decides which autonomies get a draft PR opened
here: `auto-checkpointed` (default) → `auto` and `checkpointed`; `all` → those
plus `guided`; `auto-only` → only `auto`; `never` → none (every PR is the
human's to open). For an autonomy `pr_open` does *not* cover, use the `guided`
behavior below regardless of the task's own autonomy:

- **`guided`** — do not push. Committing locally is this skill's job; pushing or
  opening a PR is the human's call, made after reading the briefing. This is the
  unchanged pre-Phase-4 behavior.
- **`auto` / `checkpointed`** — open a **draft** PR now via the `open-pr`
  capability (`core/ADAPTER-CONTRACT.md` §3.5), body =
  `work/<task-id>/pr-description.md` (§4a), head = the current branch, so the
  human's one remaining touchpoint is reviewing/merging it:

  ```
  SPINE_PR_TITLE="<commit subject>" \
    SPINE_PR_BODY_FILE=work/<task-id>/pr-description.md \
    <project root>/.spine/adapters/open-pr
  ```

  **Always a draft — this skill never merges** (proposal §6.2; the human marks
  ready and merges). Record the returned PR URL in the briefing (§4). If `open-pr`
  is `not-applicable`/absent or exits non-zero, degrade to the `guided` behavior:
  the commit is already made, so say plainly "couldn't open the PR (<reason>) —
  push and open it by hand" and note the gap; never silently drop it. (Profile-
  gated auto-open for `guided`, or disabling it for a team that prefers
  hand-opened PRs, is Phase 5.)

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
never had one).

**Worktree cleanup (Extension D — experimental):**
if this task's cwd path contains `/.claude/worktrees/` (the same cheap signal
`core/skills/task/SKILL.md`'s Extension D branch uses), this task ran in a
spine-created worktree. Ask once: "This task ran in worktree `<path>` — remove
it now, or keep it (e.g. still watching the PR)?" If the human doesn't answer
either way, default to keeping it — removal is the harder-to-reverse choice,
and a clean ship should have nothing uncommitted left to lose anyway so
there's no cost to leaving the decision open. `ExitWorktree({action: "keep"})`
or, only on explicit confirmation, `ExitWorktree({action: "remove"})` (adding
`discard_changes: true` only if the tool itself reports uncommitted changes
and the human confirms discarding them — never set it preemptively). Skip
this whole paragraph silently for a task that didn't run in a worktree.

Tell the human where the briefing is. For a `guided` task,
also point at `pr-description.md` (§4a) — pushing and opening the PR is their
call, made after reading both. For an `auto`/`checkpointed` task the draft PR
is already open (§5a) — give them its URL, so the one remaining touchpoint is
reviewing and merging it. That read is the third recurring touchpoint,
and it happens now, once, not as a gate this skill enforced on itself.
