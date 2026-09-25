<!--
Referenced by every skill that stops to ask the human something
(core/skills/*/SKILL.md) and by the hooks/scripts whose messages tell the
human to decide (core/hooks/*, core/scripts/*). Checked mechanically by
core/scripts/touchpoint-lint (run from core-selftest). Edit the standard and
the glossary here — never restate them in a skill, and never paste a gloss
that disagrees with the glossary below.

Sibling of core/templates/writing-mandate.md, which covers plan.md and
briefing.md prose; this covers the moments spine *stops and asks*.
-->

# Human touchpoint standard

The reader is a busy human switching in from other work. They have not read
`research.md`, `plan.md`, or any decision record, and they do not remember
spine's vocabulary. A question they cannot answer without opening a file is a
defective question.

## The rules

1. **Open with the decision.** One plain sentence: what is being decided, and
   why it is the human's to decide rather than spine's.
2. **Minimum context, no file-opening.** What happened, what was found, what
   is at stake — enough to answer from the screen alone.
3. **Options by consequence.** For each: what happens next, what it costs in
   time or risk, whether it can be undone. Never by mechanism.
4. **Recommendation first**, with a one-line reason. If spine has no real
   recommendation, say "no recommendation" and why — never invent one.
5. **No internal IDs or spine vocabulary** (`D-24`, `gap-10`, `Class 2`,
   `M2`, `check-stale`, `halt-tier`) unless glossed inline at first use, in a
   few words, from the glossary below. Prefer the plain words.
6. **One decision per question.** Say what is safe to ignore.
7. **Short.** Background that needs a paragraph goes in `Need to know`, never
   in the option labels.
8. **Explain it like they were on holiday.** Everyday words, short sentences,
   one idea per line. Say what changed *for a user or for them*, not which
   files, hashes, counts or IDs changed; those belong in the file you point
   to, never in the message. A number earns its place only if it changes what
   the reader does.

This standard changes how spine asks, never what it asks about or when.

## The block (skills)

Wrap each touchpoint's wording in the markers below so `touchpoint-lint` can
check it. Angle-bracket placeholders (`<...>`) are filled with the real facts
at run time; everything else is fixed wording.

```
<!-- touchpoint:start -->
> **Deciding:** <what, and why it's yours>
> **Need to know:** <what happened, what was found, what's at stake>
> **Recommend:** <option> — <one-line reason>
> 1. **<label in plain words>** — next: <what happens>; cost: <time/risk>; undo: <yes/no/how>
> 2. **<label>** — next: ...; cost: ...; undo: ...
> **Safe to ignore:** <what they need not care about, or "nothing">
<!-- touchpoint:end -->
```

## The short block (a plain "do this" or a blocked action)

For a stop that isn't a choice — a script/hook halting, or spine asking the
human to run a command themselves:

```
<!-- touchpoint:start short -->
> **What happened:** <one sentence>
> **What it means for you:** <one sentence: what is and isn't affected>
> **To continue:** <the exact command or answer that unblocks it>
<!-- touchpoint:end -->
```

Hooks and scripts write the same three labelled parts in their messages
(they cannot use markers; `core-selftest` runs them on fixtures and feeds the
output to `touchpoint-lint --stdin`). Their machine-read tokens (`HALT`,
`PROCEED`, `STALE`, `BLOCKED`, exit codes) stay exactly as they are.

## The report block (end-of-phase messages)

The same reader also gets *reports* — "implementation done", "verified",
"shipped". They follow the same rules, in this order. The bottom line must
make sense to someone who read nothing else:

```
<!-- touchpoint:start report -->
> **Bottom line:** <one plain sentence: where things stand, and whether anything waits on you>
> **What I did:** <2-4 short lines, in terms of what now works or behaves differently>
> **What you need to do:** <the next step and the exact command — or "nothing">
> **Worth knowing:** <anything surprising, left open on purpose, or not proven — in plain words — or "nothing">
<!-- touchpoint:end -->
```

Test counts, commit hashes, file lists and finding IDs stay out unless the
reader needs them to act; point to the file (`briefing.md`, `verify.md`)
instead. "Left open on purpose" items say what could go wrong and when it
would matter, in one line each.

## Requests for information

Asking for something that isn't a choice (the effort's destination, a repo
path, a name) needs no block: say what you need, why you need it, and give
one example of a good answer, in plain words.

## Glossary (the closed set `touchpoint-lint` enforces)

First use of a term inside a touchpoint must be followed directly by
` (<gloss>)` — the gloss may be shortened but must say the same thing. IDs
matching a pattern below (`D-<n>`, `gap-<n>`, `M<n>`) need a gloss of what
that specific one *is*, not what the family is.

| Term | Plain gloss |
|---|---|
| Class 0 | the lightest kind of change: no ceremony |
| Class 1 | a routine change: plan, checks, one approval |
| Class 2 | the highest-risk kind of change: needs a second person's approval |
| halt-tier | a change I promised to ask you about first |
| protected path | a file or folder you marked as needing extra care |
| check-stale | the check that notices my notes about the code are out of date |
| claims-check | the check that notices two tasks are about to edit the same things |
| content-sources-check | the check that every piece of on-screen text has a known source |
| registry-sync | saving the task's records to the shared history |
| second-approver-check | the check that a second person approved a Class 2 plan |
| path-escalate | the guard that blocks edits to protected paths |
| dep-gate | the guard that asks before any package is added or changed |
| phase-gate | the guard that blocks code edits before the plan is approved |
| design-gate | the check that the design stage is finished |
| floor | the automatic checks (build, tests, lint) that must pass |
| adversary | a reviewer whose only job is to find what's wrong |
| deviation | a place where the work departed from the approved plan |
| Known gaps | the list of problems knowingly left for a later task |
| guided | I stop after every phase and wait for you |
| checkpointed | I stop for plan approval, then once more at the finish |
| auto | I run without scheduled stops; you review the finished PR |
| handoff | the summary passed from one stage to the next |
| walking skeleton | the thinnest end-to-end version of the app |
