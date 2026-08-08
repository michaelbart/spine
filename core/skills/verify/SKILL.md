---
name: verify
description: Run the deterministic floor and the adversary agents for a task, and assemble verify.md. Invoked by /task at the verify phase — not a general code-review command.
disable-model-invocation: true
argument-hint: <task-id>
---

You are running `/verify` for task `$ARGUMENTS` (the task ID). Scripts live
at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`, the template at
`${CLAUDE_SKILL_DIR}/../../templates/verify.md`. Read `work/<task-id>/class`
first — everything below branches on it.

## 1. Floor

```
${CLAUDE_SKILL_DIR}/../../scripts/floor <class> --task <task-id> \
  --out work/<task-id>/artifacts/floor-result.json
```

This fail-fasts on the first failing capability and, for `--task`, writes
`callers`/`dep-diff` artifacts under `work/<task-id>/artifacts/` — the
adversaries read those, not the raw diff themselves. If the floor fails,
still assemble `verify.md` (§4) so the failure and everything reached before
it is on record, then report FAIL to the caller. Don't run the adversaries
against a diff the deterministic floor already rejected — that's wasted
adversary budget on something that's going to change anyway.

## 2. Adversary count

Class 2: always both, `falsifier` and `security`. Class 1: read
`~/.spine/user-config.json`'s `ceremony.class1_adversary_count` (1 or 2; if
the file or key is absent, default 2 — the calibrated default from Layer 1).
1 means falsifier only — it's the one that proves the implementation against
its own plan, which is why it's never optional.

## 3. Run the adversaries

Each adversary is a fresh subagent (Agent tool) — give it only the plan
path, the diff (`git diff <base>..HEAD` — default base `HEAD`, i.e.
whatever's currently uncommitted, since implementation work isn't committed
until `/ship`; pass a different base if the human made interim commits), and
the artifact paths from step 1. Never summarize the implementation session
into the delegation message — that defeats the independence the fresh-
context property exists for (build prompt §2.5 Layer 3).

- `subagent_type: falsifier` — delegation message points at
  `work/<task-id>/plan.md`, the diff, and
  `work/<task-id>/artifacts/callers.md`.
- `subagent_type: security` (if running) — same, plus
  `work/<task-id>/artifacts/dep-diff.md`.

Harvest each subagent's own transcript in full into the ledger under its
own phase key, same mechanism as `core/skills/task/SKILL.md` §6:
`ledger harvest <task-id> falsifier --transcript
.../subagents/agent-<id>.jsonl --from <epoch> --to <now>` (and the same for
`security` if it ran), `<id>` from the Agent tool's result. This is a
separate line item from the `verify` phase key `/task` marks for this
skill's own orchestration overhead — don't fold one into the other.

Each replies with exactly one JSON object (`core/ADAPTER-CONTRACT.md §5`).
Write each verbatim to `work/<task-id>/artifacts/<agent>-verdict-raw.json`,
then:

```
${CLAUDE_SKILL_DIR}/../../scripts/verdict-filter \
  work/<task-id>/artifacts/<agent>-verdict-raw.json \
  --out work/<task-id>/artifacts/<agent>-verdict.json
```

**Only ever read the filtered file when assembling `verify.md`.** The raw
file exists for audit, not for you to reason about — a verdict
`verdict-filter` dropped is not a verdict, regardless of how it reads.

## 4. Conformance

```
${CLAUDE_SKILL_DIR}/../../scripts/conformance work/<task-id>/plan.md \
  --out work/<task-id>/artifacts/conformance.json
```

Never blocks anything (build prompt §2.5 Layer 4) — it scores the plan, not
the change.

## 5. Assemble verify.md

Fill `${CLAUDE_SKILL_DIR}/../../templates/verify.md`'s structure into
`work/<task-id>/verify.md`, sourced only from the files written above —
this document quotes scripts, it doesn't paraphrase them:

- Floor results table from `floor-result.json`'s `results` array — one row
  per entry, pass/fail/degraded verbatim.
- Conformance line from `conformance.json`.
- One subsection per adversary that ran, from its filtered verdict file:
  the `attacked` list, then each kept verdict, then the kept/dropped counts
  (dropped count comes from `verdict-filter`'s own stderr line, capture it
  when you run step 3).
- Capability gaps: every capability in `.spine/capabilities.json` marked
  `unavailable`/`not-applicable` that this class would otherwise have run
  (Class 2 also implies `mutate` and, per the migration lane,
  `migrate-rehearse` if this task touched a migration path), with its
  recorded reason. Empty only if nothing degraded.

`ledger set <task-id> conformance_score <f1 from conformance.json>` and
`ledger set <task-id> capability_gaps <json array of the gap names>`.

## 6. Report

Reply to the caller with exactly: `PASS` or `FAIL`, plus the one-line reason
if FAIL (which capability, or "adversaries ran, see verify.md" is not a
FAIL by itself — adversary findings don't fail verify; they inform `/ship`
and the briefing). `/task` decides what happens next; this skill's job ends
at `verify.md` plus that one-line verdict.
