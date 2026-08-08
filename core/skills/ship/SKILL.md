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
why, do not proceed to §2–5. This is not a touchpoint you invent — it's the
same plan-approval-adjacent judgment the human already exercised; you're
just not allowed to walk past a gate they haven't cleared.

**`--bypass <reason>`** skips both checks — loudly, never silently. Record
the bypass in the ledger (`ledger set <task-id> bypass "<reason>"`) and give
it its own visible section in the briefing (§3). Bypass is for a genuine
emergency (production down, the fix touches auth) that can't wait on the
harness — it is not a way to route around a check you disagree with.

## 2. Distill decisions

Read `work/<task-id>/deviations.md`. For any *resolved* record whose
resolution establishes a rule future tasks should follow — not every
trivial record-and-proceed note, only ones a researcher on a later task
would actually want to find — write
`docs/decisions/<yyyy-mm-dd>-<kebab-slug>.md` from
`${CLAUDE_SKILL_DIR}/../../templates/decision.md`, citing this task ID. Do
the same for any halt-tier escalation resolved during this task. Skip this
step (write nothing) if nothing this task hit rises to that bar — a decision
record manufactured to have something to show is worse than none.

## 3. Write the delta briefing

`work/<task-id>/briefing.md` from
`${CLAUDE_SKILL_DIR}/../../templates/briefing.md`. ≤1 page. Pull capability
gaps straight from `verify.md`, deviations straight from `deviations.md`,
decisions from what §2 just produced (or "none"). If this ship used
`--bypass`, its own section here is not optional. The "what you'd want to
know in six months" line is the one line most worth spending real thought
on — don't let it default to a restatement of "what changed."

## 4. Ledger and commit

`ledger mark <task-id> ship` — per the rule in `core/skills/task/SKILL.md`
§6, this harvests the `verify` phase's window (its stored mark timestamp to
the `ship` timestamp you just wrote) into phase key `verify`. Then, since no
further mark follows `ship` in this task, harvest `ship` itself right now:
`ledger harvest <task-id> ship --transcript <path> --from <the ship mark
timestamp> --to <now>`. Then:

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

For `--bypass`, add `Spine-Bypass: <reason>` as its own trailer line
alongside (or instead of, if this was never a real task-folder task)
`Spine-Task:`. Never omit both — that's the untracked-commit ratio
(`ledger scan-untracked-ratio`) existing specifically to catch.

Do not push. Committing locally is this skill's job; pushing or opening a
PR is the human's call, made after reading the briefing.

## 5. Close out

Write `work/<task-id>/state` = `done`. Remove `.spine/current-task` (the
task is no longer active — a subsequent trivial edit should default back to
Class 0, not stay phase-gated against a finished task). Tell the human
where the briefing is — that read is the third recurring touchpoint (build
prompt §2.7), and it happens now, once, not as a gate this skill enforced
on itself.
