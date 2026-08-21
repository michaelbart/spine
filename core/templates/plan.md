<!--
Hard cap: 200 lines, this comment block included. If it doesn't fit, the
task splits — this is a forcing function, not a formality. Before proposing
this plan for approval, count the lines.

Two layers here, per the writing mandate
(core/templates/writing-mandate.md): the prose sections (gist, risks,
latitude, steps, how we'll know) are the human review surface — plain
language, headline first, everything material present, nothing pushed
below its section's first line. The `<!-- MACHINE: ... -->`-fenced
sections are read verbatim by a script or a skill's own deterministic
instructions, not by a human skimming — don't reformat what's inside a
fence, and don't let the fenced content and the prose above it disagree
about which files or decisions are in play.

**Only `## Predicted touch` is read by a real script**
(`core/scripts/conformance`, plain `awk`): a line matching
`^## Predicted touch` opens capture, each following `^- ` line is kept
(with the leading `- ` and anything from the first ` — ` onward
stripped), capture closes at the next `^## ` heading or EOF. That heading
text and single-dash-space bullet form must appear byte-for-byte inside
the fence — see the fenced block below.

**`## Grounds on decisions`, `## Ship order`, and `## Contract change`
are read by `/task` and `/ship`'s own instructions**, not a compiled
parser — but the heading text and bullet grammar are just as load-
bearing, because those skills fail silently and partially on a
malformed section, never loudly the way a script would. Keep them exact.

**Multi-repo tasks (Extension B, one plan for the whole workspace task —
build prompt §2, "one task folder at the workspace, one plan, one human
approval"):** every `## Predicted touch` entry is repo-qualified,
`<repo-name>:<path>` (e.g. `api:src/routes/items.ts`), even for a repo
this plan only touches once. `/verify` is what splits these apart per
repo when it runs `conformance` and `contract-touch` (it writes a
scratch, single-repo copy of the predicted-touch list before invoking
either) — neither script itself parses the `<repo-name>:` prefix; both
take one `--project` per invocation and resolve repos from
`workspace.json`'s own `repos[]` list. An unqualified entry in a
multi-repo plan still cannot be scored or diffed against anything, so
the repo-qualification rule stands regardless of which script ends up
doing the split.

Two sections apply only to a multi-repo plan, both omitted entirely (not
left empty) for a single-repo one:

- `## Ship order` — required the moment `## Predicted touch` names more
  than one repo, or has any unqualified (workspace-native) entry alongside
  at least one repo-qualified one. Ordered list of repo names, one per
  line, plus the reserved name `workspace` for the workspace root's own
  commit (its `work/<task-id>/` artifacts, and any unqualified predicted-
  touch path like a contract spec) wherever it belongs in the sequence —
  omit `workspace` only if `## Predicted touch` has no unqualified entry.
  Producer before consumer for an additive contract change (open question
  §5.4: declared here, validated by `/ship` against `workspace.json`'s
  registry direction — the plan is the human review surface, not something
  `/ship` derives silently). Validation failing here halts the ship, it
  does not silently reorder.
- `## Contract change` — required only when this task's diff touches a
  declared contract (`core/scripts/contract-touch` would report it
  touched). One of `expand`, `migrate`, `contract`, or `additive`
  (`core/rules/contracts.md`). A `breaking` classification `contract-touch`
  computes from the real diff, on a plan that doesn't declare `expand` or
  `contract` here, fails `/verify` outright — this line records intent,
  the mechanical check (based on the diff, not this line) is what actually
  gates.
-->

# Plan: `<task-id>` — <title in plain words>

<!-- MACHINE: header -->
task: <task-id>   class: <0|1|2>   owner: <git identity>   milestone: <id|none>
grounding: research `<research sha>` (`work/<task-id>/research.md`)<if this plan grounds on any docs/decisions/ record, add>, decisions <D-id, D-id, ...>
<!-- /MACHINE -->

## The gist

<!-- One sentence: what this task makes true that isn't true now. Then two
     to five sentences, plain words, no jargon beyond real code names: what
     changes, where, and the shape of the approach. If there were real
     alternatives, name the one rejected and why in one line — this is not
     a design-doc comparison. A reader who stops here knows what they're
     approving. -->

## What could go wrong

<!-- The two or three real risks, in human terms — the thing the research
     flagged, the invariant being edged, the blast radius. Not boilerplate:
     "edge cases may exist" is not a risk, it's a non-statement (see the
     writing mandate). If the honest answer is "little", say that in one
     line and stop. -->

## What I'll decide alone vs. stop and ask

<!-- build prompt §2.4. Every kind of decision this task might hit, sorted
     into exactly one of the three tiers below. Each list's parenthetical
     keyword is the literal `- Tier:` value `deviations.md` records against
     it (core/templates/deviations.md) — keep the keyword even though the
     heading text around it is free prose, so a real deviation can always
     be traced back to the tier that predicted it. Halt is not negotiable
     regardless of what's written here: schema, public contracts, new
     dependencies, auth logic, and anything matching a protected-path glob
     always halt — the hooks enforce the file-level cases independent of
     this list. -->

**I'll just do** (`decide-alone`):
- Naming, private structure, test organization

**I'll do and note** (`record-and-proceed`):
- Unanticipated but inside declared boundaries — logged to `deviations.md`,
  then I keep going

**I'll stop and ask before** (`halt`):
- Schema, public contracts, new dependencies, auth logic, protected paths

## Steps

<!-- Each step: what changes, and the acceptance check that proves it did —
     a command, a test name, an observable behavior. A step without a
     checkable acceptance criterion is not a step, it's a hope. -->

1. <what> — **acceptance:** <check>
2. <what> — **acceptance:** <check>

<!-- MACHINE: predicted-touch -->
## Predicted touch

<!-- Every file expected to change. This is what conformance.md scores
     against the real diff after implementation — a low score means this
     list was wrong, which means research or planning missed something.
     Be concrete; "various files in lib/" is not a predicted-touch entry.
     Multi-repo: every entry is repo-qualified, `<repo-name>:<path>`.
     Write the bare path, no backticks/code-fencing around it — conformance
     compares this string byte-for-byte against real `git diff` output,
     which is never backtick-wrapped; a Markdown-formatted path silently
     scores as a miss even when the file matches (caught for real during
     this patch's own demonstration re-render). -->

- <path/to/file1> — <why>
- <path/to/file2> — <why>
<!-- /MACHINE -->

## How we'll know it worked

<!-- One short paragraph: what the floor, the adversaries, and (if
     applicable) smoke will demonstrate. Then, plainly, anything
     verification cannot show this time — name the absent capability, per
     `.spine/capabilities.json`'s own reason, not something discovered
     later and quietly absorbed. -->

<!-- The two sections below are appendices, not part of the human review
     narrative above — omit each entirely (not left empty) when it doesn't
     apply, per this file's own header comment. -->

<!-- MACHINE: grounds-on-decisions -->
## Grounds on decisions

<!-- Optional — omit this whole section if this plan doesn't cite any
     docs/decisions/ record. /ship reads this to know which decisions to
     append this task's real implementing paths onto and flip
     adopted -> implemented. Multi-repo: a bullet may be repo-qualified,
     `<repo-name>:D-<seq>`, for a member repo's own local decision store
     (unqualified means the workspace root's own store) — same convention
     research.md's grounding-decisions: header already uses. -->

- D-<seq> — <why this plan grounds on it>
- <repo-name>:D-<seq> — <why this plan grounds on it>
<!-- /MACHINE -->

<!-- MACHINE: ship-order -->
## Ship order

<!-- Multi-repo only — omit entirely for a single-repo plan. Ordered list
     of repo names from `## Predicted touch`. /ship validates this against
     workspace.json's contract registry direction (producer before
     consumer for an additive change) before staging the merge. -->

1. <repo-name>
2. <repo-name>
<!-- /MACHINE -->

<!-- MACHINE: contract-change -->
## Contract change

<!-- Only when this task's diff touches a declared contract. One of:
     expand | migrate | contract | additive. See core/rules/contracts.md —
     a `breaking` diff classification on a plan that doesn't say `expand`
     or `contract` here fails `/verify` outright, regardless of this line. -->

<expand | migrate | contract | additive>
<!-- /MACHINE -->

<!-- MACHINE: resolves-known-gaps -->
## Resolves known gaps

<!-- Optional — omit this whole section if this plan doesn't close out any
     entry from work/<milestone-id>/milestone.md's own `## Known gaps for
     future member tasks` (docs/proposals/flagged-finding-carryforward.md
     §6). Only for a milestone member task (`--milestone <id>` given).
     /ship reads this to know which gap-<n> fenced entries to remove from
     milestone.md once this task's diff has actually shipped — never
     inferred from prose, only from this explicit citation, same
     "no implicit resolution" discipline `docs/proposals/
     flagged-finding-carryforward.md` §6 already commits to. Citing a gap
     here that this plan's own `## The gist` doesn't actually address is
     caught the same way an unfounded `## Grounds on decisions` citation
     would be — at human plan review, not mechanically. -->

- gap-<n> — <how this plan's approach closes it>
<!-- /MACHINE -->

<!-- MACHINE: closes-milestone-gap -->
## Closes milestone gap

<!-- Optional — omit this whole section entirely unless this task was
     *not* created with `--milestone <id>` (i.e. `work/<task-id>/milestone`
     is unset going into this plan) and this plan's own scope is required
     to make some existing milestone's `## Done-definition` true, despite
     not being one of that milestone's listed `## Member tasks` — e.g. a
     gap a prior member task's own briefing.md flagged as still open. Never
     used for a task already created with `--milestone` — that task is
     already a listed member, this section would be redundant by
     construction.

     Presence of this section is the mechanical trigger `/task` acts on at
     plan-approval time: it resolves `work/<id>/milestone.md`, appends a
     new numbered entry to `## Member tasks` with this task's own real id
     (no `TBD` — the task already exists) and a one-line description drawn
     from `## The gist`, and writes `work/<task-id>/milestone` = `<id>` —
     the same splice `--milestone <id>` would have produced at classify
     time, just arriving after the fact so `/ship`'s §3a/3b/3c milestone
     bookkeeping engages for this task instead of silently never firing.
     A milestone named here that this section doesn't actually explain
     how the plan closes is a plan-quality problem for human review to
     catch, same as an unfounded `## Grounds on decisions` citation — not
     something `/task` verifies semantically. -->

- <milestone-id> — <what gap this closes and why it isn't one of that
  milestone's listed member tasks>
<!-- /MACHINE -->
