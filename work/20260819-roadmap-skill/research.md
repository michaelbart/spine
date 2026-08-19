# Research: `20260819-roadmap-skill`

Commit: HEAD at research time — see `git log -1`.

## How the skill system works

Every skill lives at `core/skills/<name>/SKILL.md` with frontmatter
(`name`, `description`, `disable-model-invocation: true`, optional
`argument-hint`). The install mechanism symlinks
`.claude/skills/<name>` → the skill directory, making `/name` a
user-invocable command. No registration file — the symlink IS the
registration.

**Skill shapes in the codebase:**
- **Setup skills** (bootstrap, design, workspace, adopt): run once per
  install or project event, produce no `work/<task-id>/` folder, commit
  their own output. These are the right model for `/roadmap`.
- **Task skills** (task, ship, verify): per-task, produce `work/<task-id>/`
  state. Not the right model here.
- **Read-only skills** (spine, tasks, costs, visualize, task-report): no writes.

## Milestone template format (core/templates/milestone.md)

Key structure a `/roadmap`-created file must conform to:
- `# Milestone \`<id>\`: <title>`
- `## Member tasks` — numbered list of `TBD` entries (with one-line
  descriptions) until `/task --milestone <id>` replaces each with a real
  task ID
- `## Inter-task contracts` — empty at creation; filled in by tasks
- `## Known gaps for future member tasks` — starts empty; populated by
  `/ship` §3a only. Must include `<!-- next-gap-id: 1 -->` counter.
- `## Capability targets` — table; for M0 this is required; for later
  milestones it can be empty or omitted per the template's own comment
- `## Done-definition` — prose; `/roadmap` should draft a one-sentence
  done-definition from the milestone's scope

## Input sources

1. `docs/vision.md` — the milestone sequence (new convention from this
   build). Required input. If absent, /roadmap asks the human to provide
   one or paste their build plan inline before continuing.
2. `docs/product-spec.md` — optional detailed spec; if present, read it
   for richer scoping context when drafting member-task descriptions.
3. Completed `work/M*/milestone.md` `## Known gaps` sections — must be
   absorbed into the appropriate upcoming milestone's scope or
   inter-task contracts. Format must be verified before absorbing (per
   the §3a fix in this session).
4. `docs/decisions/D-<n>-*.md` — each decision's `## Consequences`
   section may have a "Leaves open:" line naming follow-on work; these
   must be assigned to a milestone or explicitly noted as out of scope.
5. `docs/charter.md` — non-negotiables and hard constraints; these bound
   what milestones can assume.

## What already exists

- `work/M*/milestone.md` detection: `core/skills/spine/SKILL.md` already
  globs `work/M*/milestone.md` and determines completeness. Same glob
  works here.
- The `/ship` §3b trigger (added this session) already tells the human to
  run `/roadmap` — the entry point is already wired.
- The `docs/vision.md` convention is already documented in bootstrap
  (added this session) but the file has no template — intentionally,
  it's human-authored.
- No existing skill creates multiple milestone files in one pass.

## Milestone boundary rule (from our design discussion)

"One independently-demoable user-facing capability, plus the backend work
it structurally requires, sequenced so each milestone's dependencies are
satisfied by earlier ones."

Operationally: scan vision.md's milestone titles; for each one, ask
whether it structurally requires any other milestone's output. Build a
simple dependency ordering and present it to the human. The human confirms
or reorders before any file is written.

## What /roadmap does NOT do

- Does not plan member tasks (the specific code changes). That's what
  `/task --milestone <id>` does at implementation time, grounded in real
  code. Roadmap only writes TBD entries with one-line descriptions.
- Does not touch M0 or any already-complete milestone.
- Does not generate `docs/vision.md` — that's the human's document.
- Does not produce `docs/product-spec.md`.
- Does not modify the charter or any decision record.
