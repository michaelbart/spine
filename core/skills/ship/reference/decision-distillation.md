# Ship — decision implementation and distillation (loaded on demand)

Loaded from SKILL.md when: the plan has a `## Grounds on decisions` section or deviations.md has a resolved record.

**Implement decisions this task's plan cited.** Read
`work/<task-id>/plan.md`'s `## Grounds on decisions` section (per
`core/templates/plan.md` — absent entirely if the plan cited none; skip
this part in that case). For each bullet there: read
`docs/decisions/D-<seq>-*.md`, append
this task's *actual* diff paths (`git diff --name-only <base>..HEAD`, not
the plan's predicted-touch list — the diff is what's real) to its
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
