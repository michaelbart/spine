<!--
Written by /ship. ≤1 page — the human reads this once, at merge. This is
the mental-alignment artifact (build prompt §1, failure mode 5, named first
by the engineer): if reading this doesn't leave the engineer knowing what
their product now does and why, it failed at its one job. Don't pad it to
look thorough; cut anything without a reason to be here.
-->

# `<task-id>`: <one-line summary>

## What changed

<!-- The actual behavior change, in product terms first, code terms second. -->

## Why

<!-- The request this served — link the charter line or prior decision if
     one drove it. -->

## Deviations taken

<!-- One line per deviations.md record, resolution included. "None" if none
     occurred — don't manufacture one, don't hide one. -->

## Decisions made

<!-- Links into docs/decisions/ for anything distilled from this task's
     deviations or escalations. -->

## Contracts touched

<!-- Omit this whole section for a single-repo task, or a multi-repo task
     whose contract-touch run found nothing touched. When present: each
     touched contract, its spec_change classification, which consumer
     repos were gated via contract-check (vs. edited directly and gated by
     their own floor), and any undeclared coupling the falsifier's
     cross-repo mandate surfaced — build prompt §2's "registry coverage
     made visible, so neglect is loud." A registry_stale warning from
     contract-touch belongs here too, not just in the raw artifact. -->

## Milestone done-definition

<!-- Omit this whole section if this task isn't part of a milestone, or is
     part of one that isn't completing with this ship (core/skills/ship/
     SKILL.md §3). When present: which milestone, and whether its
     Done-definition is actually met by real state right now — say so
     plainly if it isn't; a milestone reported "done" that silently isn't
     is exactly the skeleton-skip failure mode this section exists to
     make loud instead of quiet. -->

## Capability gaps that degraded verification

<!-- Pulled from verify.md's capability-gaps section. "None" if the floor
     ran at full strength for this class. -->

## Tooling gaps

<!-- Quoted straight from verify.md's own "Tooling gaps" section — never
     re-derived. This is about spine's own scripts being unreachable
     during this task (ledger, check-stale, conformance, verdict-filter,
     floor), distinct from the capability gaps above. "None" if genuinely
     empty; a rising count of these across tasks (see /costs) means the
     engineer is running a lighter version of this system than they think
     they are. -->

## What you'd want to know in six months

<!-- The one thing a future engineer (possibly you) debugging this area
     would wish this briefing had said. Often the sharpest line in the
     document; don't skip it because the rest felt complete. -->
