# Plan: `20260819-roadmap-skill`

## The gist

Create `core/skills/roadmap/SKILL.md` — a new setup-shaped skill that
reads `docs/vision.md` plus prior milestone Known-gaps and decision
Consequences, enforces Known-gaps format, proposes the full remaining
milestone sequence for human confirmation, and writes
`work/M<n>/milestone.md` for each. Symlink it into the install mechanism
so `/roadmap` becomes an invocable command on any spine-installed project.

## Predicted touch

- `core/skills/roadmap/SKILL.md` — **new file** (the skill itself)
- `core/scripts/setup` — add `roadmap` to the symlink list so `/bootstrap`
  and `/adopt` wire it into new and adopted projects automatically
- `docs/tradeoffs.md` — one entry in the open questions / shelf table
  noting `/roadmap` exists and what it defers

## Acceptance checks

1. `core/skills/roadmap/SKILL.md` exists and its frontmatter has
   `name: roadmap`, `description: ...`, `disable-model-invocation: true`.
2. `core/scripts/setup` symlinks `roadmap` alongside the other skills.
3. The skill's §0 preflight correctly stops when `docs/vision.md` is
   absent and tells the human what to do.
4. §1 (absorb prior Known-gaps) matches the format-enforcement rule added
   to `/ship` §3a — same shape check, same flag-and-ask behavior.
5. §2 (propose milestone sequence) presents a dependency-ordered list for
   human confirmation before writing any file.
6. §3 (write milestone files) uses `core/templates/milestone.md`'s exact
   structure: TBD member entries with one-line descriptions, empty
   Inter-task contracts, empty Known-gaps with `next-gap-id: 1`, drafted
   Done-definition, and Capability targets table (empty for non-M0
   milestones).
7. §4 (completeness check) reports any vision.md item unclaimed by any
   proposed milestone, loudly.
8. The skill never touches a completed milestone or M0's structure.

## Sections of SKILL.md to write

### §0 Preflight
- Require `docs/vision.md`. If absent: stop, tell the human to write one
  (milestone titles from their build plan, one line each) or paste the
  build sequence inline and offer to write it for them.
- Require at least one complete milestone (`work/M*/milestone.md` with
  all tasks `done`) OR `work/M0/milestone.md` existing. If neither:
  stop — there's nothing to sequence from yet; run `/task` to start M0.
- Read the highest complete milestone number to know where to start
  (`M<last+1>` onward).

### §1 Absorb prior state
- Read every complete milestone's `## Known gaps` section.
- Verify format (fenced `gap-<n>` blocks, `next-gap-id` counter) before
  absorbing — flag malformed entries to the human, same rule as `/ship` §3a.
- Read every `docs/decisions/D-<n>-*.md` for "Leaves open:" lines in
  `## Consequences`. Collect as "open follow-ons."
- Present a brief summary: N well-formed gaps to absorb, M decision
  follow-ons to place, any malformed entries flagged.

### §2 Propose milestone sequence
- Read `docs/vision.md`'s planned milestone list. Present it as the
  starting point.
- For each planned milestone, ask: does it structurally require any other
  planned milestone's output? Derive a dependency order.
- Assign each absorbed Known-gap and decision follow-on to the earliest
  milestone whose scope plausibly covers it; flag any that don't fit
  anywhere clearly and ask the human to assign them.
- Present the full proposed sequence — ordered list, each entry: milestone
  id, title, one-sentence scope, absorbed gaps/follow-ons it inherits,
  proposed Done-definition — for human confirmation.
- Human may reorder, split, merge, or rename. Iterate until they confirm.

### §3 Write milestone files
- For each confirmed milestone (starting at `M<last-complete+1>`):
  - Skip if `work/M<n>/milestone.md` already exists — do not overwrite
    existing milestone files. Report which were skipped.
  - Write from `core/templates/milestone.md` with: real title, TBD member
    tasks (numbered, with one-line descriptions from the confirmed scope),
    empty Inter-task contracts, empty Known-gaps section with
    `next-gap-id: 1`, Capability targets table (empty for non-M0), and the
    drafted Done-definition.
  - Append absorbed Known-gap entries (with their original `gap-<n>` ids
    preserved) into the appropriate milestone's `## Known gaps` section,
    updating `next-gap-id` accordingly. These entries carry forward
    verbatim — never reworded.

### §4 Completeness check
- Every item named in `docs/vision.md`'s milestone list must map to
  exactly one of the newly written (or already existing) milestone files.
  Any item with no corresponding `work/M<n>/` folder fails loudly: list
  the unclaimed items and stop until the human says to defer them or
  assigns them.

### §5 Commit
- Single commit: `docs: plan milestones M<n>–M<m>` with the list of files
  written as the body. Setup-shaped commit, no `Spine-Task:` trailer.

## What this does not do

- Does not plan member tasks (code-level). `/task --milestone <id>` does
  that, grounded in real code at implementation time.
- Does not generate `docs/vision.md` or edit it.
- Does not modify completed milestones or M0's structure.
- Does not modify any decision record or the charter.
