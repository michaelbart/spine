---
name: note-issue
description: Log a known issue found outside any active task or ticket — no classification, no research, just "record this now." Wraps core/scripts/issue-ledger against docs/known-issues.md.
disable-model-invocation: true
argument-hint: <one-line description>
---

You are running `/note-issue`. `$ARGUMENTS` is a one-line description of
something worth tracking — found by manual testing, a hunch while reading
code, anything with no active `/verify` run to attach a finding to and no
ticket-tracker key to fetch (`/intake` needs a real ticket; a local-only
project, or a stray observation mid-session, has neither). This is the
low-ceremony entry point on purpose: no task folder, no classification, no
plan, no phases — the entire job is "write it down before it's forgotten."

Project root: the workspace root if `workspace.json` exists here,
otherwise this project. Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/<name>`. `${CLAUDE_SKILL_DIR}` is a
placeholder you expand to this skill's own directory; hand the resulting
path — including the `../../` — to the shell verbatim. Do **not**
lexically collapse `skills/note-issue/../..` to `.claude/`:
`.claude/skills/note-issue` is a symlink into the spine core checkout, so
the shell must resolve `../../` against the symlink's real target
(`<spine>/core/...`). Collapsing it as text yields a nonexistent
`.claude/scripts/...` path and a "no such file" error.

**This never touches `work/<milestone-id>/milestone.md`'s "## Known gaps
for future member tasks" section.** That section is provenance-locked to
real adversary verdicts from `/ship`'s flagged-finding triage — an entry
there always traces to a `disposition: "not_fixed"` verdict from some
task's own `verify.md`. What `/note-issue` records has no such
provenance; it lives in a structurally separate file,
`docs/known-issues.md`, precisely so the two can never be confused or
accidentally merged.

## 1. No arguments — ask

If `$ARGUMENTS` is empty, ask for the one-line description before doing
anything else. Don't guess at what the engineer wants recorded.

## 2. Optionally sharpen severity and evidence — briefly

Don't turn this into an interrogation. If the description or surrounding
conversation already makes severity and evidence obvious, just proceed —
don't ask when you already know. Ask only when genuinely ambiguous, and
keep it to at most one quick question:

- **Severity** (`low` / `med` / `high`): infer from the description if it's
  clear (a crash or data-loss risk reads `high`; a cosmetic nit reads
  `low`); default `med` if you can't tell and it's not worth asking.
- **Evidence**: a `file:line`, a URL, or a short free-text note pinning
  down where this was observed — carry forward anything already named in
  the conversation (a file path just discussed, a command's output).
  `"none"` is fine if there's genuinely nothing more specific than the
  description itself.

## 3. Record it

```
${CLAUDE_SKILL_DIR}/../../scripts/issue-ledger add "<one-line description>" \
  --severity <low|med|high> --evidence "<evidence>" --project <project root>
```

This creates `docs/known-issues.md` from `core/templates/known-issues.md`
on first use if it doesn't exist yet — nothing to set up beforehand.

If `issue-ledger` could not run at all (not a "ran and reported an
error" case — genuinely could not execute), say so plainly and, as a
fallback, hand-append the same fenced shape `core/templates/
known-issues.md` documents directly to `docs/known-issues.md` yourself —
the point of this skill is that the observation never gets lost to a
tooling hiccup, not that the script must be the one to write it.

## 4. Confirm

Report back plainly: the issue id it was logged as (`issue-<n>`), and
that `/roadmap` will pick it up for placement into a milestone (or an
explicit decline) the next time it runs. Nothing else — no next steps to
take now, no phase to advance. The whole point of this command is that
recording an issue costs one line and stops there.
