<!--
Hard cap: 200 lines, this comment block included. If it doesn't fit, the
task splits — this is a forcing function, not a formality. Before proposing
this plan for approval, count the lines.

`## Predicted touch` below is machine-parsed verbatim by
core/scripts/conformance: the heading text must be exactly that, each entry
a `- path/to/file` bullet (single leading dash-space), and an optional
` — trailing description` is allowed and stripped before scoring. Don't
reformat that section.
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
     Be concrete; "various files in lib/" is not a predicted-touch entry. -->

- <path/to/file1> — <why>
- <path/to/file2> — <why>
