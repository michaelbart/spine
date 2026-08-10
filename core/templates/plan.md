<!--
Hard cap: 200 lines, this comment block included. If it doesn't fit, the
task splits — this is a forcing function, not a formality. Before proposing
this plan for approval, count the lines.

`## Predicted touch` below is machine-parsed verbatim by
core/scripts/conformance: the heading text must be exactly that, each entry
a `- path/to/file` bullet (single leading dash-space), and an optional
` — trailing description` is allowed and stripped before scoring. Don't
reformat that section.

`## Grounds on decisions`, if present, is machine-parsed the same way by
`/ship`'s decision-lifecycle step: exact heading, one `- D-<seq>` bullet
per cited decision, same optional trailing-description convention. Omit
the whole section (not an empty one) if this plan doesn't ground on any
docs/decisions/ record.

**Multi-repo tasks (Extension B, one plan for the whole workspace task —
build prompt §2, "one task folder at the workspace, one plan, one human
approval"):** every `## Predicted touch` entry is repo-qualified,
`<repo-name>:<path>` (e.g. `api:src/routes/items.ts`), even for a repo
this plan only touches once — `core/scripts/conformance` and
`core/scripts/contract-touch` both split on the first `:` to resolve which
repo's own git tree a bare path belongs to; an unqualified entry in a
multi-repo plan cannot be scored or diffed against anything. Two more
sections apply only to a multi-repo plan, both omitted entirely (not left
empty) for a single-repo one:

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

# Plan: <task title>

Task: `<task-id>` · Class: `<0|1|2>` · Research: `<research sha>`
(`work/<task-id>/research.md`)

## Approach

<!-- The shape of the change in a few sentences. Not step detail — that's
     below. If there were real alternatives, name the one rejected and why
     in one line; don't write a design-doc comparison here. -->

## Steps

<!-- Each step: what changes, and the acceptance check that proves it did —
     a command, a test name, an observable behavior. A step without a
     checkable acceptance criterion is not a step, it's a hope. -->

1. <what> — **acceptance:** <check>
2. <what> — **acceptance:** <check>

## Latitude table

<!-- build prompt §2.4. Every kind of decision this task might hit, sorted
     into exactly one tier. Halt-tier is not negotiable regardless of what's
     written here: schema, public contracts, new dependencies, auth logic,
     and anything matching a protected-path glob always halt — the hooks
     enforce the file-level cases independent of this table. -->

| Tier | Covers |
|---|---|
| Decide-alone | Naming, private structure, test organization |
| Record-and-proceed | Unanticipated but inside declared boundaries — log to `deviations.md`, keep going |
| Halt | Schema, public contracts, new dependencies, auth logic, protected paths |

## Predicted touch

<!-- Every file expected to change. This is what conformance.md scores
     against the real diff after implementation — a low score means this
     list was wrong, which means research or planning missed something.
     Be concrete; "various files in lib/" is not a predicted-touch entry.
     Multi-repo: every entry is repo-qualified, `<repo-name>:<path>`. -->

- <path/to/file1> — <why>
- <path/to/file2> — <why>

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

## Ship order

<!-- Multi-repo only — omit entirely for a single-repo plan. Ordered list
     of repo names from `## Predicted touch`. /ship validates this against
     workspace.json's contract registry direction (producer before
     consumer for an additive change) before staging the merge. -->

1. <repo-name>
2. <repo-name>

## Contract change

<!-- Only when this task's diff touches a declared contract. One of:
     expand | migrate | contract | additive. See core/rules/contracts.md —
     a `breaking` diff classification on a plan that doesn't say `expand`
     or `contract` here fails /verify outright, regardless of this line. -->

<expand | migrate | contract | additive>
