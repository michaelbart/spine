---
name: ship
description: Gate, commit, and brief a task that's passed verification — re-grounds against what changed underfoot since research, runs the merge gate, distills/updates decisions, does milestone bookkeeping (flagged-finding triage, done-definition check, known-gap resolution), writes the delta briefing and PR description, and commits. Invoked by /task at the ship phase, or directly as `/ship --bypass <reason>` for a genuine emergency.
disable-model-invocation: true
argument-hint: <task-id> [--bypass <reason>]
---

**Output rule:** the `<!-- touchpoint:... -->` lines in this skill are lint markers; never print them. Show the human only the `>` lines between them.

You are running `/ship` for task ID and optional `--bypass <reason>` from
`$ARGUMENTS`. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`,
templates at `${CLAUDE_SKILL_DIR}/../../templates/<name>`. Hand
`${CLAUDE_SKILL_DIR}/../../...` to the shell verbatim, `../../` included —
do **not** lexically collapse it to `.claude/`; `.claude/skills/ship` is a
symlink into the spine core checkout, and collapsing the text yields a
nonexistent `.claude/scripts/...` path.

**This skill does six distinct jobs, in order — not one "gate and commit"
step.** A non-milestone, guided task still passes through all
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
6. **§5/§6 Commit and close-out.**


**Voice.** Every message this skill leaves for the human follows
`core/templates/human-touchpoint.md`. Before sending one, run its "Before you
send" list: gloss or drop internal names, IDs and commit hashes, keep one
decision per message, and put surprises first.

## 0. Ship-time re-grounding

The window between plan approval and ship is unguarded otherwise — the
code can change underfoot and invalidate this task's grounding after
`check-stale` already passed at plan time. Re-run it:

```
${CLAUDE_SKILL_DIR}/../../scripts/check-stale work/<task-id>/research.md
```

**When `check-stale` printed STALE for at least one file:** read `${CLAUDE_SKILL_DIR}/reference/regrounding-drift.md` and follow it exactly before continuing.

If ok: `git pull --rebase` onto the current mainline, then re-run the floor:

```
${CLAUDE_SKILL_DIR}/../../scripts/floor <class> --task <task-id> \
  --out work/<task-id>/artifacts/floor-result-postrebase.json
```

This re-verifies the diff against whatever actually landed. If this
post-rebase floor fails, that's a real merge-gate failure (§1 below), not
a deviation — the diff itself now conflicts with what actually landed.

**Third re-grounding check: the decision index, against whatever
`docs/decisions/` looks like right now.** §2 below regenerates
`docs/decisions/INDEX.md` for *this* task's own decision edits, but a
hand-edit or a `--bypass` can have changed the
store since without regenerating it — the same staleness shape
`check-stale` exists to catch for `research.md`, one layer over:

```
${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <project root> --check
```

A `STALE` result here is
never this task's own fault by construction — §2 hasn't run yet at this
point in the skill — so it is not a deviation and does not touch the
circuit breaker; it's a stale, inherited artifact this task is about to
fix anyway once §2's regeneration runs. Note it in `notes.md` and move on — §2's own
regeneration (which runs
unconditionally whenever this task touched `docs/decisions/`, and should
also run here if it's stale for a reason unrelated to this task) is what actually resolves it before this
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
just not allowed to walk past a gate they haven't cleared. Say it in this
form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start short -->
> **What happened:** shipping stopped because <the checks haven't passed | a question is still open>: <the specific one, in a sentence>.
> **What it means for you:** nothing has been merged or committed; this is a stop, not a failure.
> **To continue:** <fix it and re-run `/verify <task-id>` | answer the open question and I'll mark it resolved>.
<!-- touchpoint:end -->

**`--bypass <reason>`** skips every check above (floor/deviations) — loudly, never silently. Record the bypass in `notes.md` and give it its own visible section in the briefing (§4). Bypass is for a genuine emergency
(production down, the fix touches auth) that can't wait on the harness —
it is not a way to route around a check you disagree with. `--bypass` does
not skip §0's ship-time re-grounding — that runs first, unconditionally;
what it skips is acting on a stale result as a hard halt. Also record it in
the local event log: `${CLAUDE_SKILL_DIR}/../../scripts/spine-event bypass
reason="<the reason given>"`.

## 2. Decisions

**When `work/<task-id>/plan.md` has a `## Grounds on decisions` section or `work/<task-id>/deviations.md` has a `- Status: resolved` record:** read `${CLAUDE_SKILL_DIR}/reference/decision-distillation.md` and follow it exactly before continuing.

**Regenerate the decision index.** If this task touched `docs/decisions/`
at all above — a status flip, an `## Implementing paths` append, or a
newly-distilled record — regenerate its store's index before this task's
commit in step 5:

```
${CLAUDE_SKILL_DIR}/../../scripts/decision-index --project <project root>
```

Skip entirely if this task
cited no decision and distilled none — the index doesn't need
regenerating when the store it summarizes hasn't changed. Mechanical, no
review needed.

## 3. Milestone bookkeeping

Read `work/<task-id>/milestone` (absent = this task isn't part of a
milestone — skip §3a/§3b/§3c entirely, no note needed in the briefing). If
present, use `work/<id>/milestone.md` at the project
root. All three steps below run against it — §3a and §3c on
*every* member-task ship, §3b only on the milestone's completing ship. §3a
and §3c can run in either order — they touch the same section but never
the same entries (§3a only ever allocates new ids off the monotonic
`next-gap-id` counter, §3c only ever removes ids this task's own plan
cited), so there's no ordering hazard between them.

### 3a. Flagged-finding triage

**When `work/<task-id>/milestone` exists:** read `${CLAUDE_SKILL_DIR}/reference/milestone-flagged-triage.md` and follow it exactly before continuing.

### 3b. Milestone done-definition

**When `work/<task-id>/milestone` exists:** read `${CLAUDE_SKILL_DIR}/reference/milestone-done-and-gaps.md` and follow it exactly before continuing.

### 3c. Known-gap resolution

**When `work/<task-id>/plan.md` has a `## Resolves known gaps` section:** read `${CLAUDE_SKILL_DIR}/reference/milestone-done-and-gaps.md` and follow it exactly before continuing.

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
the latter appends the filename) — over cap (either signal) means trim
before shipping, not ship anyway. Section by
section, each sourced only from what's already been produced — this file
quotes, it doesn't re-derive:

- **What & why** / **What surprised us**: one or two sentences on what's
  now true and why; deviations straight from `deviations.md`, one line
  each, resolution included ("Nothing — the plan held." if none).
- **Verification, honestly** bullets — `Floor` from `verify.md`'s floor
  result (a `degraded:waived-bootstrap` smoke-run line — the M0 bootstrap
  waiver of the Class 2 smoke hard gate — is quoted here explicitly, never
  folded into "pass": smoke did not run); `Re-grounding` quoted from what §0 recorded ("check-stale: ok,
  floor re-run: pass" is the unremarkable case, still shown); `Adversaries`
  as count + max severity + one-line gist each, pointer to `verify.md`,
  never compressed further (this file's own header note); `Capability
  gaps` straight from `verify.md`; `Tooling gaps` straight from
  `verify.md`'s own "Tooling gaps" section (never re-derive or
  re-summarize — quote); `Setup events` straight from `verify.md`'s own
  "Setup events" section, same quote-don't-re-derive rule — distinct from
  `deviations.md`-sourced "What surprised us" above, per
  `core/skills/task/SKILL.md` §4's boundary test; `Plan accuracy` from
  `conformance.json`'s score, in words.
- **Overrides & bypasses** (omit entirely if none occurred): `--bypass`'s
  own line is not optional when used.
- **Milestone** (omit entirely if this task isn't part of one): §3b's
  result — which milestone, and (only on the completing ship) whether its
  Done-definition is actually met by real state, said plainly either way.
  §3a's result folds in here too: which findings (if any) were carried
  into `milestone.md`'s Known gaps, with their new `gap-<n>` ids
  (`guided`), or the drafted "Proposed milestone gap entries — undecided"
  list awaiting the human's PR-time triage (`auto`) — never
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
deviations.md, verify.md, approval.json) plus three §4 doesn't
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
sentence traces to plan.md, deviations.md, verify.md, approval.json, or one
of the three artifacts above; nothing here is generated by re-reading code.

**Same over-cap tracking as §4's briefing, same reason**: `wc -l
< pr-description.md` and `wc -w < pr-description.md` (redirect stdin),
same ~600-word real "over cap" signal. Over cap means trim before shipping,
not ship anyway.

## 5. Commit

**Derive the ticket, add its trailer.** **Prefer `work/<task-id>/ticket`** if
present (recorded by `/intake`); otherwise extract the ticket key from the
current branch name (`git rev-parse --abbrev-ref HEAD` and parse the
ticket-pattern from `.spine/ticket-pattern.conf` if present, or fall back to
the first `[A-Z]+-[0-9]+` match in the branch name). If either yields a key,
add `Spine-Ticket: <key>` as an additional trailer
line on this task's commit alongside
`Spine-Task:`, per `core/ADAPTER-CONTRACT.md` §6's composing-trailer rule. If it
prints nothing (off-ticket), omit that line.

**Commit a refreshed map on its own.** `/task` refreshes `docs/map.md` before
research when it was stale or empty. If `git status --porcelain docs/map.md`
shows a change, commit just that file
first, so the refresh never lands inside the task's own diff or PR:

```
git add -- docs/map.md && git commit -m "chore: refresh project map"
```

No `Spine-Task:` trailer, Skip silently if unchanged.

Then:

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

For `--bypass`, add `Spine-Bypass: <reason>` as its own trailer line
alongside (or instead of, if this was never a real task-folder task)
`Spine-Task:`. Never omit both — every commit in this system should carry
`Spine-Task:` or `Spine-Bypass:` so the task origin is traceable from
`git log`.

**Pushing and opening the PR is autonomy-aware, gated by the profile**
(read `work/<task-id>/autonomy`, `core/skills/task/SKILL.md`, absent = `guided`;
and `.spine/profile.json`'s `pr_open`, `core/ADAPTER-CONTRACT.md` §7, absent =
`auto-only`). `pr_open` decides which autonomies get a draft PR opened
here: `auto-only` (default) → `auto`; `all` → `auto` plus `guided`; `never` → none (every PR is the
human's to open); the legacy value `auto-checkpointed` is still accepted and means `auto-only`. For an autonomy `pr_open` does *not* cover, use the `guided`
behavior below regardless of the task's own autonomy:

- **`guided`** — do not push. Committing locally is this skill's job; pushing or
  opening a PR is the human's call, made after reading the briefing. This is the
  unchanged pre-Phase-4 behavior.
**When `work/<task-id>/autonomy` reads `auto`, or `.spine/profile.json` `pr_open` is `all`:** read `${CLAUDE_SKILL_DIR}/reference/pr-opening-auto.md` and follow it exactly before continuing.

## 6. Close out

Whatever else this section does, the message you leave the human with is
the report form (per `core/templates/human-touchpoint.md`), never a freeform summary.
Show only the `>` lines, never the `<!-- touchpoint:... -->` marker lines:

<!-- touchpoint:start report -->
> **Bottom line:** <shipped or not, in one plain sentence, and whether anything waits on you (a push, a PR to open)>
> **What I did:** <2-4 short lines: what now works or behaves differently, in user terms>
> **What you need to do:** <push and open the PR by hand, with the command, or "nothing">
> **Worth knowing:** <what was left open on purpose and when it would matter, one line each; anything not proven; or "nothing">. Details: `work/<task-id>/briefing.md`.
<!-- touchpoint:end -->

First log it: `${CLAUDE_SKILL_DIR}/../../scripts/spine-event shipped $(${CLAUDE_SKILL_DIR}/../../scripts/spine-stats --rollup <task-id> 2>/dev/null)`
(the rollup adds this task's token and active-time totals to the event so they
survive Claude Code pruning old transcripts; it prints nothing if it cannot run.
`spine-event` reads the task from `.spine/current-task`, so this must run before
that file is removed below).

Run `${CLAUDE_SKILL_DIR}/../../scripts/set-state <task-id> done` (the commit from §5 must have
actually landed before this write; it refuses without a passing `verify.md` and zero open
deviations, so under `--bypass` run it as `set-state --bypass <task-id> done`). Remove `.spine/current-task`
(the task is no longer active — a subsequent trivial edit should default
back to Class 0, not stay phase-gated against a finished task).

Tell the human where the briefing is. For a `guided` task,
also point at `pr-description.md` (§4a) — pushing and opening the PR is their
call, made after reading both. For an `auto` task the draft PR
is already open (§5a) — give them its URL, so the one remaining touchpoint is
reviewing and merging it. That read is the third recurring touchpoint,
and it happens now, once, not as a gate this skill enforced on itself.
