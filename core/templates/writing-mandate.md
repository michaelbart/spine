<!--
Referenced by /task's plan-writing step (core/skills/task/SKILL.md §3) and
/ship's briefing-writing step (core/skills/ship/SKILL.md §4) — edit here
when this needs to change, not in either skill; one place, referenced
twice, never duplicated prose.
-->

# Writing mandate — plan.md and briefing.md prose

1. **Bottom line first.** A reader who stops after a section's first
   sentence already has its real content, not a lead-in to it.
2. **Plain language.** Real code identifiers, file names, and command
   names are fine; jargon standing in for an explanation is not.
3. **Write for the least-context reader on the team** — a briefing is read
   by an engineer who wasn't in the task.
4. **Never bury a surprise below its section's first line.** If it's worth
   knowing and it's three sentences deep, move it to the first sentence.
5. **No stock phrases.** "Edge cases may exist," "further testing
   recommended," and their relatives say nothing. Name the real risk, or
   say "little" and stop.
6. **Prose and machine-fenced sections must agree.** Don't let "the gist"
   describe a different set of files or decisions than `## Predicted
   touch` or `## Grounds on decisions` actually lists.
7. **Two floors are hard rules, not style**, in the briefing: adversary
   findings never compress below count + max severity + one-line gist +
   pointer; overrides and bypasses are never omitted or folded together
   when real.
