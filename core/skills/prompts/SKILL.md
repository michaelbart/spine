---
name: prompts
description: Print a ready-to-paste prompt for generating spine handoff material in another session — a product spec, a mid-project feature handoff, or a Claude Design UI handoff bundle. Read-only, no gate, no state written.
disable-model-invocation: true
argument-hint: [product-spec | feature-handoff | ui-handoff]
---

You are running `/prompts` against `$ARGUMENTS`. Source files live at
`core/prompt-templates/*.md` in the spine checkout (`${CLAUDE_SKILL_DIR}/
../../prompt-templates/<name>.md` — hand that path to the shell verbatim,
`../../` included; do **not** lexically collapse it to `.claude/`, which is
a symlink into the spine core checkout). This command only reads files —
it writes nothing, gates nothing, and needs no class.

## No argument — list what's available

Read `${CLAUDE_SKILL_DIR}/../../prompt-templates/README.md`'s table and
present it as a short menu: each prompt's name, what file it produces, and
what consumes that file downstream (the table already has all three).
Close with: "Run `/prompts <name>` to print one directly." Don't print any
prompt's actual content in this no-argument case — just the menu.

## With an argument — print that prompt

Match `$ARGUMENTS` against `product-spec`, `feature-handoff`, or
`ui-handoff` — accept an obvious loose match too (`spec`, `product`,
`feature`, `ui`, `design` should all resolve unambiguously to one of the
three — both `ui` and `design` map to `ui-handoff`; if it's genuinely
ambiguous, show the menu instead of guessing).

Read the matched file. It has an HTML-comment usage header at the top and
one or more fenced prompt blocks below (`feature-handoff.md`: one;
`product-spec.md`: a conditional Stage 0 plus the main prompt — Stage 0
only applies if the idea currently exists solely in a Claude Design
session, say so plainly rather than printing it as always-required;
`ui-handoff.md`: three staged ones plus a closing note). Do not dump the
raw file, comment markup included — a reader in a terminal shouldn't have
to see `<!-- -->` fencing:

1. **Restate the usage note in plain prose**, one short paragraph: when to
   use this prompt (including any conditional stage, and when it does or
   doesn't apply), where to paste it, and where the output should be
   saved (the header always states the target path — e.g.
   `docs/product-spec.md`, `docs/ui/`).
2. **Then print every fenced prompt block verbatim, each in its own code
   fence**, in the file's own order, with the file's own stage/section
   headings (`ui-handoff.md`'s three stages, `product-spec.md`'s Stage 0)
   as plain text labels above each one — so the human can copy exactly
   one block cleanly, without reformatting anything themselves.
3. **Close with what to do with the result**: which real file path to
   save the output as, and — from the README table — which spine command
   picks it up next (`/design`, `/wayfinder`, `/roadmap`, or, for
   `ui-handoff`, `core/rules/ui-design-system.md` and the
   `ui-conformance` capability).

If the human wants to edit the prompt before using it (a common case —
these are starting points, not fixed scripts), that's expected; say so if
it seems relevant, don't present the block as something that must be
pasted unmodified.
