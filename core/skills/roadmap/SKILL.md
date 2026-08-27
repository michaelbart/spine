---
name: roadmap
description: Plan the remaining milestones from docs/vision.md — absorbs prior Known-gaps, open docs/known-issues.md entries, and decision follow-ons, proposes a dependency-ordered sequence for human confirmation, then writes work/M*/milestone.md for each. Run after any milestone ships when the next one isn't planned yet.
disable-model-invocation: true
argument-hint: [--after M<n>]
---

You are running `/roadmap`. Templates at
`${CLAUDE_SKILL_DIR}/../../templates/`, scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/`. Hand that path to the shell verbatim,
`../../` included — do **not** lexically collapse it to `.claude/`;
`.claude/skills/roadmap` is a symlink into the spine core checkout.

**This is setup, not a task** — it produces no `work/<task-id>/` folder,
runs no floor, needs no class. It is facilitated: you propose, the human
confirms the sequence before you write any file. Nothing is written until
§2's confirmation step completes.

## 0. Preflight

**Check `docs/vision.md`.** If it doesn't exist:

> `docs/vision.md` is missing. Before `/roadmap` can plan milestones, you
> need a document listing the intended build sequence — one line per
> milestone, e.g.:
> ```
> ## Planned milestones
> - M1 — Feature X
> - M2 — Feature Y
> ```
> Paste your build plan or product spec's build-order section here, and
> I'll write `docs/vision.md` for you. Or create it yourself and re-run
> `/roadmap`.
>
> If you don't actually know the milestone sequence yet — the effort is
> large and the path is still genuinely foggy, not just unwritten —
> `/wayfinder` is the better starting point: it charts it as a map of
> decision tickets and writes `docs/vision.md`'s planned-milestones
> section once that map clears. Come back here once it has.

If the human pastes content, extract the milestone sequence, write
`docs/vision.md` (just the milestone list, no other content required —
keep it under 30 lines), and continue. If they decline, stop here.

**Determine the starting point.** If `--after M<n>` was given, use that
as the last complete milestone. Otherwise: glob `work/M*/milestone.md`
and determine completeness (every member entry is a real task id and every
`work/<task-id>/state` reads `done`). The starting point is one past the
highest-numbered complete milestone.

If no milestone is complete and `work/M0/milestone.md` doesn't exist:

> No milestones exist yet. Run `/task <description> --milestone M0` to
> start the walking skeleton first — `/roadmap` plans what comes *after*
> a milestone exists to sequence from.

Stop.

If `work/M0/milestone.md` exists but isn't complete yet, continue — the
roadmap can be planned even while M0 is in flight. Note it in the output:
"M0 is still in progress — planned milestones start at M1."

**Parse `docs/vision.md`.** Read the `## Planned milestones` section (or
the nearest equivalent heading — the file is human-authored, so be
tolerant of heading variation). Extract the ordered list of milestone
titles and any descriptions. If the file exists but has no recognizable
milestone list, tell the human what you found and ask them to add one
before continuing.

---

## 1. Absorb prior state

This step gathers everything that *must* land in some future milestone.
Never silently drop any item from this step — each one either gets placed
in a milestone or gets explicitly declined by the human.

### 1a. Known gaps from completed milestones

For each complete milestone's `work/<id>/milestone.md`, read the
`## Known gaps for future member tasks` section. Before absorbing any
entry, **verify its shape**:

- A well-formed entry is a fenced block containing an `id: gap-<n>` line
  and the `source:` field from `core/templates/milestone.md`'s format.
- The section must have a `<!-- next-gap-id: N -->` counter.

**If any entry is malformed** (plain prose bullet, no fence, no `id:`
line, or counter missing):

> Found malformed Known-gap entries in `work/<id>/milestone.md` that
> can't be machine-cited by future tasks. Before planning, I need to fix
> their format:
> [list each malformed entry, quoted]
> Can I reformat these into the proper `gap-<n>` shape now? Content stays
> identical — this is shape-only. Or I can note them as non-machine-citable
> and leave them as-is.

Wait for the human's answer. If reformatting: edit in place now. If
leaving as-is: note each in the output as "not absorbed — non-machine-
citable format" and exclude them from placement below.

Collect all well-formed gaps. Each carries its original `gap-<n>` id,
source, and prose — never rewrite these.

### 1b. Open known issues (docs/known-issues.md)

`docs/known-issues.md` (`core/templates/known-issues.md`,
`core/skills/note-issue/SKILL.md`) holds observations logged outside any
active task or ticket — no adversary verdict behind them, structurally
separate from the Known-gaps section above and never to be merged with it.
Absorbing these mirrors §1a's discipline exactly: never silently drop one.

If `docs/known-issues.md` doesn't exist, skip this step silently —
nothing has ever been logged.

Otherwise, list the open entries mechanically rather than hand-parsing the
file yourself:

```
${CLAUDE_SKILL_DIR}/../../scripts/issue-ledger list --open --project <project root>
```

Each line is `issue-<n>  [<severity>]  open  <description>`. If the
script reports none (`issue-ledger: no open issues in ...` on stderr,
nothing on stdout), there's nothing to absorb — continue silently. If the
script could not run at all (tooling-gap discipline), read
`docs/known-issues.md` directly and, before absorbing any entry, **verify
its shape**: a well-formed entry has an `id: issue-<n>` line and a
`status:` field, and the section carries a `<!-- next-issue-id: N -->`
counter. If any entry is malformed:

> Found malformed entries in `docs/known-issues.md` that can't be
> machine-cited by future tasks. Before planning, I need to fix their
> format:
> [list each malformed entry, quoted]
> Can I reformat these into the proper `issue-<n>` shape now? Content
> stays identical — this is shape-only. Or I can note them as
> non-machine-citable and leave them as-is.

Wait for the human's answer, same as §1a's own malformed-entry flow. If
leaving as-is: note each in the output as "not absorbed —
non-machine-citable format" and exclude them from placement below.

Collect all well-formed open entries. Each carries its original
`issue-<n>` id, severity, evidence, and description — never rewrite these.

### 1c. Decision follow-ons

Read every `docs/decisions/D-<n>-*.md`. In each file's `## Consequences`
section, look for lines starting with "Leaves open:" or containing an
explicit follow-on item. Collect these as a list:
`D-<n> (<title>): <follow-on text>`.

If none exist, skip this step silently.

### 1d. Summary before proposing

Present a brief inventory before moving to §2:

> **Prior state absorbed:**
> - N Known-gap entries from M0 (gap-1 through gap-N)
> - [any malformed entries, flagged]
> - K open known-issues.md entries (issue-1, issue-3, ...) [or "none"]
> - M decision follow-ons: [list by D-id]

If the inventory is empty (nothing to absorb), say so briefly and continue.

---

## 2. Propose milestone sequence

**Read `docs/vision.md`'s milestone list.** Treat it as the authoritative
ordering unless the human has reason to revise it here.

**Check for structural dependencies.** For each planned milestone, ask:
does it structurally require another milestone's output before it can
start? (E.g., "the sync backend milestone must come before the taste
mirror milestone because the mirror requires user data.") Derive a
dependency order. If the vision.md order already respects dependencies,
say so. If it doesn't, propose a reordering and explain why.

**Assign absorbed items.** For each Known-gap, open known-issues.md entry,
and decision follow-on from §1, assign it to the earliest milestone whose
stated scope plausibly covers it. If an item doesn't clearly fit any
milestone, surface it:

> This item doesn't fit clearly into any planned milestone:
> [item text]
> Which milestone should own it, or should it be deferred (excluded from
> all milestones with a stated reason)?

A known-issues.md entry assigned here is *placed*, not resolved — its
`docs/known-issues.md` status stays `open` until the work that actually
fixes it runs `issue-ledger resolve <issue-id>`. Placing it just means its
description (with the literal `issue-<n>` id, so it's grep-able later)
becomes part of the owning milestone's member-task description in §3.

**Present the proposed sequence.** For each milestone (starting at
`M<last-complete+1>`), show:

```
M<n>: <title>
  Scope: <one sentence>
  Done when: <one sentence done-definition>
  Absorbs: gap-X from M<prev>, issue-Y, D-Z follow-on (or "nothing absorbed")
  Structural dependency: M<k> must be complete first (or "none")
```

Then ask: **"Does this sequence look right? Any reordering, splits,
merges, or title changes before I write the files?"**

Wait for explicit confirmation. Iterate until the human says to proceed.
A partial approval ("looks right except swap M2 and M3") is fine — revise
and re-present only the changed entries, then confirm again.

---

## 3. Write milestone files

For each confirmed milestone, in order:

**Skip if already exists.** If `work/M<n>/milestone.md` already exists,
report "M<n> already has a milestone file — skipping" and move on. Never
overwrite an existing milestone file.

**Write `work/M<n>/milestone.md`** from `core/templates/milestone.md`
with:

- Heading: `# Milestone \`M<n>\`: <confirmed title>`
- `## Member tasks`: numbered list of TBD entries. Each TBD line should
  include a one-sentence task description drawn from the confirmed scope —
  not just `TBD` alone. How many TBD entries? One per logical unit of work
  implied by the scope. Err on the side of 2–4; don't invent granularity
  that isn't in the confirmed scope. Format: `` 1. `TBD` — <description> ``.
  If this milestone absorbs a known-issues.md entry (§1b), name its literal
  `issue-<n>` id inside the description of whichever member task covers
  it — e.g. `` 1. `TBD` — fix the flaky CI test (issue-2) `` — so a later
  `/roadmap` completeness check (§4) can find it mechanically.
- `## Inter-task contracts`: empty (tasks fill this in as they ship)
- `## Known gaps for future member tasks`: if this milestone absorbs gaps
  from §1, append each fenced entry verbatim, preserving its original
  `gap-<n>` id and source field. Update `<!-- next-gap-id: N -->` to one
  past the highest id present. If no gaps are absorbed, leave the section
  empty with just the counter at 1.
- `## Capability targets`: an empty table (no capabilities required to
  plan — tasks add these during their own research phase). Omit for
  non-M0 milestones unless the confirmed scope named a specific capability
  target.
- `## Done-definition`: the one-sentence done-definition from the
  confirmed scope, expanded slightly if needed — concrete enough that a
  future `/ship` §3b check can evaluate it against real state.

`mkdir -p work/M<n>` before writing.

---

## 4. Completeness check

After writing all files, verify coverage:

- Every milestone title listed in `docs/vision.md` must now have a
  corresponding `work/M<n>/` directory.
- Every Known-gap from §1 must be assigned to exactly one new milestone
  file OR explicitly declined by the human in §2.
- Every open known-issues.md entry from §1b must be assigned (its
  `issue-<n>` id appears in some new milestone's `## Member tasks`
  description) OR explicitly declined by the human in §2 — never silently
  dropped, same discipline as Known-gaps.
- Every decision follow-on from §1 must be assigned or explicitly
  deferred.

**If any vision.md item has no corresponding milestone folder:**

> **Completeness gap:** The following items from `docs/vision.md` have no
> milestone folder:
> [list]
> These were planned but not written — either the sequence stopped early,
> or they were intentionally deferred. Confirm which, and I'll either
> write the missing files or note the deferral in `docs/vision.md`.

Do not silently pass this check. An uncovered item is always surfaced.

If everything is covered, report:
> All N planned milestones from `docs/vision.md` are now covered
> (M<first> through M<last>). N gaps absorbed, K known-issues.md entries
> placed, M decision follow-ons placed.

---

## 5. Commit

Single commit with message:

```
docs: plan milestones M<n>–M<m>

<one line per milestone: Mn: title>

[if gaps absorbed]:   Absorbs N Known-gap(s) from prior milestones.
[if issues absorbed]: Absorbs K known-issues.md entr(y|ies).
[if reformatted]:     Reformatted N malformed entries in <files>.
```

No `Spine-Task:` trailer — this is a setup-shaped commit.

After committing, tell the human:

> Milestones M<n>–M<m> are planned. Start the next task with a bare
> `/task` — it auto-continues M<n>'s first queued task — or type the
> description yourself with `/task <description> --milestone M<n>`.
