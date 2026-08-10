<!-- spine:research
sha: <full commit sha HEAD was at when this research was written>
files:
  - <path/to/file1>
  - <repo-name>:<path/to/file2>
repos:
  - <repo-name>@<full commit sha>
grounding-decisions:
  - D-<seq>@<hash>
  - <repo-name>:D-<seq>@<hash>
-->

<!--
Written by the researcher subagent: read-only tools, fresh context, no
access to this session's conversation. The header above is load-bearing,
not decoration — core/scripts/check-stale parses it verbatim:
  - `sha:` must be a full commit SHA, one space after the colon. For a
    multi-repo task this is the *workspace root's own* HEAD sha (it governs
    workspace-native grounding files only, e.g. a cited contracts/<name>/
    spec — see `repos:` below for member-repo files).
  - `files:` must be followed by one `  - <path>` line (two leading
    spaces, dash, space) per grounding file, nothing else indented under it.
    **Multi-repo (Extension B)**: an entry may be repo-qualified,
    `<repo-name>:<path>` — check-stale resolves `<repo-name>` via
    `workspace.json` and diffs that file against *that repo's own* sha
    from the `repos:` block below, not the top-level `sha:`. An
    unqualified entry is always treated as workspace-native (checked
    against the top-level `sha:` and the workspace root's own git tree) —
    this is a single-repo research.md's entire files: block, unchanged;
    the qualified form is additive syntax a single-repo project never
    produces or needs.
  - `repos:` — **optional, multi-repo only.** One `  - <repo-name>@<sha>`
    line per member repo this research cites a file from (same
    two-space-dash-space shape as `files:`/`grounding-decisions:`): that
    repo's own HEAD sha at research time. Required if and only if `files:`
    contains at least one `<repo-name>:<path>` entry citing that repo;
    absent entirely for a single-repo research.md.
  - `grounding-decisions:` is optional — omit the whole line if this
    research doesn't ground on any docs/decisions/ record. When present,
    same two-space-dash-space list shape as `files:`, one
    `D-<seq>@<hash>` entry per cited decision. Compute `<hash>` with
    `core/scripts/decision-hash docs/decisions/D-<seq>-*.md` at the moment
    you cite it — never hand-write a hash. check-stale quarantines this
    document if a cited decision's hash has since drifted (its actual
    content changed) or its status has moved to `superseded`; a status
    flip alone (e.g. adopted -> implemented) does *not* count as drift —
    decision-hash excludes the status line by design. **Multi-repo
    (Extension B)**: a decision id may be repo-qualified,
    `<repo-name>:D-<seq>@<hash>` — a member repo that joined the workspace
    brownfield keeps its own pre-existing `docs/decisions/` store (a
    `D-<seq>` id is only unique *within* one store, exactly like a
    repo-qualified `files:` path is only unique within one repo's tree),
    resolved the same way via `workspace.json`. Unqualified means the
    workspace root's own store — the natural place for a decision made
    *about the relationship between repos* rather than about one repo's
    own internals, but a task is free to ground on either kind, or both.
Every file the claims below depend on must be listed, or check-stale can't
catch drift in it. Same for every decision this research grounds on. This
doc is the plan's only grounding — if it's wrong, the plan built on it is
wrong in the same shape.
-->

# Research: <task title>

Task: `<task-id>` · Class: `<0|1|2>` (as declared, pending plan-time
auto-escalation if the predicted-touch list turns out to hit a protected
path)

## What this task needs to change

<!-- One or two sentences: the request in plain language, not yet a plan. -->

## How it works today

<!-- The actual current behavior, traced through real code, not assumed
     from naming. Cite file:line. This is the bulk of the document. -->

## For a bug fix: root cause

<!-- Root-cause mode only. Reproduce first (what input, what actual vs.
     expected output), then trace backward to the real cause — not the
     first plausible-looking line. Omit this section entirely for
     non-bug-fix tasks rather than leaving it empty. -->

## Relevant invariants

<!-- What other code currently depends on staying true. This is what the
     falsifier's caller-map mandate and the migration discipline both need
     downstream — surface it here even if the plan doesn't touch it, so the
     plan author can see what NOT to break. -->

## docs/map.md staleness

<!-- State the map's sha: stamp and how far it is from this research's own
     sha (commit count or "current"). If materially stale, say so plainly
     rather than treating the map as ground truth. -->

## Open questions for planning

<!-- Things this research could not resolve that the plan (or a deviation,
     once implementation starts) will have to. Not a place to guess. -->
