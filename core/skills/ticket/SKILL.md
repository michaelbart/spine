---
name: ticket
description: Generate a copy-paste JIRA ticket description from a built-and-verified task's own artifacts — summary, acceptance criteria, QA notes, files changed — reading plan.md/verify.md/deviations.md rather than re-summarizing the diff. Output only; no tracker integration. Optional step for tasks that have no upfront ticket (e.g. /intake was not used); /ship has no dependency on this having run.
disable-model-invocation: true
argument-hint: [<task-id>]
---

**Output rule:** the `<!-- touchpoint:... -->` lines in this skill are lint markers; never print them. Show the human only the `>` lines between them.

**This command is optional.** If a ticket already existed before the task
started (created via `/intake`), run `/ship` directly — the key is already
recorded and this command adds nothing. `/ticket` is for tasks where no
ticket existed upfront and one is needed for QA or stakeholder visibility.
`/ship` has no dependency on this having run.

You are running `/ticket`. `$ARGUMENTS` is an optional task ID. If given,
use it directly. If omitted, read `.spine/current-task` (one line, the
active task ID) from the project root. If neither yields an ID, ask the
engineer which task to generate a ticket description for before proceeding.

Project root: the workspace root if `workspace.json` exists here,
otherwise this project. Templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`. `${CLAUDE_SKILL_DIR}` is a
placeholder you expand to this skill's own directory; hand the resulting
path — including the `../../` — to the shell verbatim. Do **not** lexically
collapse `skills/ticket/../..` to `.claude/`: `.claude/skills/ticket` is a
symlink into the spine core checkout, so the shell must resolve `../../`
against the symlink's real target (`<spine>/core/...`). Collapsing it as
text yields a nonexistent `.claude/scripts/...` path and a "no such file"
error.

## §1 Resolve the task and check preconditions

From the resolved task ID, look for `work/<task-id>/verify.md`.

**If `work/<task-id>/verify.md` does not exist**, the task has not yet
reached the verify phase. Stop:

<!-- touchpoint:start short -->
> **What happened:** `work/<task-id>/verify.md` doesn't exist — this task hasn't been built and verified yet.
> **What it means for you:** `/ticket` generates a description from what was actually built; there's nothing concrete to describe yet.
> **To continue:** Run `/task <task-id>` to finish the implement and verify phases, then re-run `/ticket`.
<!-- touchpoint:end -->

**If `work/<task-id>/verify.md` exists but its `Result:` line does not
read `PASS` verbatim**, the task failed verification. Stop:

<!-- touchpoint:start short -->
> **What happened:** `work/<task-id>/verify.md` is present but verification didn't pass — the `Result:` line is not `PASS`.
> **What it means for you:** The ticket description should reflect finished, passing work — address the failures first.
> **To continue:** Run `/task <task-id>` to resolve the verification failures, then re-run `/ticket`.
<!-- touchpoint:end -->

## §2 Ensure the ticket template exists

Check for `.spine/ticket-template.md` at the project root.

**If it is missing**, copy
`${CLAUDE_SKILL_DIR}/../../templates/ticket-template.md` to
`.spine/ticket-template.md` — then stop:

<!-- touchpoint:start short -->
> **What happened:** No ticket template was found, so a default has been written to `.spine/ticket-template.md`.
> **What it means for you:** The default covers the common JIRA fields; your team may use different section names or extra fields.
> **To continue:** Edit `.spine/ticket-template.md` to match your team's JIRA layout — rename sections, add `{{priority}}`, `{{component}}`, or any other blank fields you need — then re-run `/ticket <task-id>`.
<!-- touchpoint:end -->

**If it exists**, continue to §3.

## §3 Read artifacts and fill tokens

Read the task's own written artifacts — do not re-read the diff and
re-summarize. Quote from the source files; the plan, verify, and deviation
records already carry the relevant prose.

### Token sources

**`{{summary}}`** — from `work/<task-id>/plan.md` `## The gist` section.
Take the opening sentence (what this task makes true that wasn't true
before). Trim anything too implementation-specific for a ticket reader.

**`{{acceptance_criteria}}`** — from `work/<task-id>/plan.md` `## Steps`
section. Collect every line matching `**acceptance:** <check>` and reformat
as a numbered list. These are the planned checks; `verify.md`'s passing
result confirms the floor (the automatic build, test, and lint checks)
ran against them.

**`{{qa_notes}}`** — assembled from two sources:

1. `work/<task-id>/verify.md` — read the adversary findings section (the
   adversary is a reviewer whose only job is to find what is wrong). For
   each finding: severity, a one-line gist. If no findings, write
   "No adversary findings."
2. `work/<task-id>/deviations.md` (if it exists and is non-empty) — each
   deviation entry in plain words: what the plan assumed, what was
   actually true, how it was resolved. If absent or empty, write
   "No deviations."

**`{{files_changed}}`** — from `work/<task-id>/plan.md` `## Predicted touch`
section. List each `- <path>` entry, one per line. Conformance already
validated these against the real diff, so this list is the authoritative
record of what changed.

**`{{epic_link}}`** — spine knows the milestone ID but not its JIRA epic
key. If `work/<task-id>/milestone` exists (one line, the milestone ID),
set this token to `_(milestone <id> — fill in the JIRA epic key before
creating this ticket)_`. If no milestone file exists, set it to
`_(no milestone)_`.

### Unknown tokens

Any `{{...}}` placeholder in the template that is not in the list above
— `{{priority}}`, `{{component}}`, `{{labels}}`, or anything else the team
added — leave verbatim. These are manual-fill blanks the engineer or PM
completes in JIRA.

### Removed tokens

If the team edited the template to remove a section that contained a
recognized token, that token simply does not appear in the output. Do not
re-insert it.

## §4 Output the ticket description

Print the filled template inside a Markdown code fence (triple backticks,
no language tag) so the engineer can copy it cleanly without extra
formatting.

Then report (show only the `>` lines, never the `<!-- touchpoint:... -->`
marker lines):

<!-- touchpoint:start report -->
> **Bottom line:** The ticket description is ready to copy into JIRA — paste it, create the ticket, then run `/ship <task-id>`.
> **What I did:** Filled the template from `plan.md` (summary, acceptance steps, files changed), `verify.md` (review findings), and `deviations.md` (plan-versus-reality notes).
> **What you need to do:** Paste the description into JIRA and create the ticket. Then record the key so `/ship` can use it: either name the branch `<TICKET-KEY>-<slug>` (picked up automatically) or write the key to `work/<task-id>/ticket` (one line). Then run `/ship <task-id>`.
> **Worth knowing:** `{{epic_link}}` is a placeholder — link the new ticket to the milestone epic in JIRA before it goes to QA. Any unknown tokens (like `{{priority}}`) are still in the output; fill them by hand before pasting.
<!-- touchpoint:end -->
