# Human touchpoints

Design rationale moved out of `docs/tradeoffs.md`. The standard itself is `core/templates/human-touchpoint.md`; `core/scripts/touchpoint-lint` checks it.


Design rationale: questions and end-of-phase reports were reaching the
human full of spine's own vocabulary (`check-stale`, `D-24`, `gap-10`,
`Class 2`) and option labels that described mechanism, not consequence.
Most of the worst ones were not written anywhere in spine: at a stop where
a skill only said "tell the human what you need resolved," the model
improvised the question, jargon included. The fix therefore puts fixed
wording (a marked block: decision, why it's yours, what to know, a
recommendation, options by consequence) at every stop site, and a
matching `report` block for the messages that end a phase.

What is checked mechanically, and what is not:

- **Checked** (`core/scripts/touchpoint-lint`, run by `core-selftest`):
  every marked block in `core/skills/*/SKILL.md` has its required labels,
  every option states `next:` and `undo:`, and every glossary term and
  `D-<n>` / `gap-<n>` / `M<n>` id is glossed inline at first use. The
  messages hooks and scripts show a human are run on fixtures and linted
  the same way; a skill that cites the standard but contains no block
  fails.
- **Not checked**, deliberately: whether an *unmarked* stop exists (the
  words "stop", "ask" and "wait" mean too many things in these skills for
  a grep to be reliable, and a noisy lint gets ignored), whether the
  wording is actually clear, and what the model says at run time. The
  block gives the model fixed text to fill in instead of composing its
  own, which is the mitigation; a stop added without a block is caught in
  review, or by `/ratchet` if it recurs.
- **Placeholders are unchecked:** the lint reads the fixed wording, not what
  gets filled into `<...>` at run time, so "say the milestone's title, not
  `M2`" is a rule for the writer.
- **Plain-word limit:** the glossary is a closed list, so a new internal
  term is invisible to the lint until someone adds it.

Quick yes/no prompts (cheap to undo, confident recommendation) use a compact
`confirm` block — a question, two answers, one marked recommended, six lines
at most — after the full block proved too heavy for them; anything with
real consequences keeps the full block.

Two behavior changes rode along, both approved: `/task` no longer asks the
human whether to redo research when `check-stale` flags only the task's own
placeholder-to-task-id edit in `milestone.md` (it keeps the research and
notes the false positive); and `core-selftest` now exits non-zero when a
case fails (it previously ended with whatever its last command returned).
