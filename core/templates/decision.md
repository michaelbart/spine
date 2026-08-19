<!--
One file per decision under docs/decisions/, named `D-<seq>-<kebab-slug>.md`
— `<seq>` is a single counter shared across the whole store regardless of
which entry path wrote the record (next value: one more than the highest
existing `docs/decisions/D-<n>-*.md`, or 1 if none exist yet). The `- Id:`
field below is authoritative if a filename and this field ever disagree
(they must not — but nothing besides discipline enforces that on rename).

Three entry paths write into this one store — never a fourth, and never a
parallel decision-document format elsewhere:

  - **/design**, before any code exists. Authored speculatively, by
    design — status starts `proposed`, moves to `adopted` once design
    review (falsifier/security, design mode) has run against it. Nothing
    can falsify an `adopted` design-time decision yet; there is no code to
    disagree with it. This is the one entry path where "written
    speculatively" is correct, not a violation.
  - **/ship** (the original path, unchanged in spirit): distilled from a
    *resolved* deviation or halt-tier escalation whose resolution
    establishes a rule future tasks should follow — never every trivial
    record-and-proceed note, only ones a researcher on a later task would
    actually want to find. A ship-distilled decision is written directly
    as `adopted`, and since the code that prompted it already exists (this
    task's own diff), `/ship` flips it straight to `implemented` in the
    same edit, `## Implementing paths` pre-filled — there is no window
    where a ship-distilled decision sits at `adopted` un-implemented.
  - **Human-directed, disclosed** (any spine session, mid-project): a human
    explicitly asks to record a standalone decision that fits neither of
    the above — most often a process/sequencing rule discovered after
    design but never distilled from any task's own deviation (e.g. "the
    design system must be built before any screen is assembled"). Written
    directly as `adopted` (no design review to run, no diff to distill
    from — same reasoning the `/ship`-distilled path above already uses),
    `Category: other` (the only entry path allowed to use it — the
    six-category discipline above still binds `/design` alone, never
    invent a seventh named category there), `Source: human decision
    (<date>)`. Same rule `/design` already states applies here too: never
    write one because a plausible answer occurred to you — only because
    the human explicitly asked for this to be recorded.

Consumers, and what each needs the `- Id:` field to resolve exactly:
  - the research skill greps this directory for decisions touching its
    target area before writing new research, so a decision made once
    doesn't get silently re-litigated task after task.
  - `core/scripts/check-stale`'s `grounding-decisions:` header branch
    (core/templates/research.md) — quarantines research whose cited
    decision's content hash (`core/scripts/decision-hash`, which excludes
    the `- Status:` line by design — see that script's own comment) has
    drifted, or whose status has moved to `superseded`.
  - `core/scripts/verdict-filter`'s `decision:<id>` evidence kind — an
    adversary verdict citing a decision must give an id that resolves to a
    real file here and a quote that appears verbatim in it, or the verdict
    is dropped before any model reads it.
  - `core/scripts/design-gate`'s stopping-rule checks #3 (foundational
    category coverage — reads `- Category:`) and #4 (adopted-decision
    count vs. the cap — reads `- Status:`).

Lifecycle (`- Status:` line, exact single lowercase word after the colon —
same discipline `core/templates/deviations.md`'s `- Status:` line already
requires, for the same mechanical-greppability reason):

  proposed -> adopted -> implemented -> superseded

`implemented`: once `/ship` has recorded implementing paths onto this
record from a task whose plan cited it, code is authoritative from that
commit forward — `check-stale` treats further drift in those implementing
paths as it already treats any grounding file's drift.
`superseded`: append-only. A new record supersedes by reference
(`- Supersedes: D-<old-id>`); the *old* record's only permitted edit,
ever, is its own `- Status:` line flipping to `superseded` (plus
`- Superseded-by:` naming the new id) — never rewrite its Context/
Decision/Consequences to match the new reality. A typo-level fix to an
already-adopted record is not a reason to edit it in place either:
supersede with a note, or live with the typo. Editing a decision's actual
content in place destroys the reason a content hash exists — a hash
change must mean the decision changed, never that someone tidied it.
-->

# <decision title>

- Id: D-<seq>
- Status: proposed | adopted | implemented | superseded
- Category: state-management | persistence | module-boundaries | error-handling | auth-model | repo-topology | other
- Date: <yyyy-mm-dd>
- Source: design session (`docs/charter.md`) | task `<task-id>` (`work/<task-id>/`) | human decision (<yyyy-mm-dd>)
- Scope: <glob>[, <glob>...]  <!-- paths/modules this decision governs — greppable, consumed by class escalation and future rules -->
- Supersedes: D-<id> | none
- Superseded-by: D-<id> | none

## Context

<!-- What forced this decision — the deviation/escalation it came from, or
     (design-time) the design question it answers — in enough detail that
     someone who never saw the task or the design session understands why
     this came up. -->

## Decision

<!-- What was decided, stated as a rule future work can follow, not a
     narrative of the discussion. -->

## Alternatives rejected

<!-- Real alternatives and why each was rejected. Required, with reasons,
     for a /design-authored record. For a /ship-distilled record where
     there wasn't a designed set of alternatives to weigh — just a
     deviation that got resolved — write "n/a — distilled from a resolved
     deviation, see work/<task-id>/deviations.md" rather than inventing
     alternatives that were never actually considered. For a human-directed
     record, write the real alternatives the human actually named when
     asked, or "n/a — ad hoc process decision" if none were weighed —
     never invent one either. -->

## Consequences

<!-- What this rules out, what it commits to, what it leaves open. -->

## Implementing paths

<!-- Populated by /ship, append-only, the moment a shipped task's plan
     cites this decision (core/templates/plan.md's `## Grounds on
     decisions`). The real files that made this decision concrete — never
     hand-edited to "correct" a stale entry, only added to as further
     tasks implement more of it. "None yet" until that first happens. -->

## Contracts implied

<!-- Optional. "None" unless this decision implies a producer/consumer
     boundary between repos (Extension B, ws/contracts/<name>/) — most
     single-repo decisions will say "none." -->
