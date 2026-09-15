---
name: adopt
description: Install the spine into an existing repository — calibration, adapter generation, a bounded survey, and a charter draft the human must edit. Run from a session in the spine repo itself.
disable-model-invocation: true
argument-hint: --project <path-to-existing-project>
---

You are running `/adopt` against `--project <path>` from `$ARGUMENTS`. If
`<project>/.spine/capabilities.json` already exists, this is a
recalibration, not a first install — say so, and treat every step below as
"re-confirm, don't blindly re-ask" rather than a cold interview.

Unlike `/bootstrap`, there is real code here already. Depth arrives per-task
via research, not in one encyclopedic pass now — a comprehensive day-one
survey would be stale within a month and, per the wrong-beats-missing
constraint, actively harmful. Stay bounded.

Scripts and templates below are referenced as
`${CLAUDE_SKILL_DIR}/../../scripts/<name>` /
`${CLAUDE_SKILL_DIR}/../../templates/<name>` — hand that to the shell
verbatim, `../../` included. Do **not** lexically collapse it to
`.claude/`; `.claude/skills/adopt` is a symlink into the spine core
checkout, and collapsing the text yields a nonexistent `.claude/scripts/...`
path.

## 0.5. Delegate the survey

Before any calibration step below, spawn the `surveyor` agent (Agent tool,
`subagent_type: surveyor`) with a delegation message containing the
`--project` path and whether this is a first install or a recalibration
(does `<project>/.spine/capabilities.json` already exist). Its entire reply
is the survey digest — do not scan the repository yourself. Everywhere §2
through §4 below says "infer from the repo," "read the repository layout,"
or similar, that inference is now the surveyor's digest, not your own
Read/Grep/Bash calls against the project. This mirrors how `/task` delegates
research to the `researcher` agent: the raw exploration (every grep, every
file read on a codebase that may be large) happens in the surveyor's own
fresh context, and only the digested findings — small, structured — land in
this session, which still has adapter generation, `docs/map.md`/
`docs/charter.md` writing, and the install commit ahead of it.

Adapter generation itself (writing executable scripts, running
`adapter-conformance --all`) stays in this session — the surveyor is
read-only and cannot write files or run scripts; only the inference of
*what commands go into those adapters* is delegated.

## 1. Layer 1 calibration

Identical to `core/skills/bootstrap/SKILL.md` §2 — check
`~/.spine/user-config.json` first, confirm-or-override if present, full
interview only if absent.

## 2. Layer 2 calibration — inspect first, then confirm

Unlike greenfield, you can infer here — from the surveyor's digest (§0.5),
not your own reading of the repo. Present its `## Protected-path candidates`
list as a concrete proposed `.spine/protected-paths.conf` for the engineer
to edit, not a blank interview. Confirm risk posture (default: any
protected-path touch forces Class 2) and CI integration (default: mirror
locally; use the digest's `## CI shape` section — what real CI already
does, if any — so the floor can be checked against it for "mirror-or-exceed,
never less").

## 2.5 Layer 2.5 — team strictness profile

Same as `core/skills/bootstrap/SKILL.md §3.5` — write and `profile-check`
`<project>/.spine/profile.json`. Unlike greenfield, **use the surveyor's
`## Team strictness signal`** as the starting preset and present it for
confirmation rather than asking cold. On a recalibration, show the existing
profile and confirm-or-edit rather than re-asking cold.

## 3. Layer 3 calibration and adapter generation

Same process as `core/skills/bootstrap/SKILL.md` §4, except stack, commands,
and runtime shape come from the surveyor's digest (§0.5) rather than your own
reading of the repo — present its `## Stack & commands` and `## Runtime
shape` findings for confirmation rather than asking cold. The digest already
answers **whether this project renders a UI a person looks at — browser or
native mobile/desktop via simulator/emulator**; if yes, confirm its
proposed view/component glob(s) for `.spine/ui-paths.conf` and how
`ui-render` reaches a real rendered state (dev-server start command + port
for a browser app; simulator/emulator boot, bundle/package id, and launch
path for a native app). If no: mark `ui-render` `not-applicable`, reason
"no UI surface to render in this project's runtime shape."
If yes and this project has no `docs/ui/` handoff bundle yet, mention
**`/prompts ui-handoff`** the same way `core/skills/bootstrap/
SKILL.md` does — an existing codebase adopting spine mid-life is exactly
as likely to want one as a greenfield project.
Generate `.spine/adapters/<name>` for each of the 19 capabilities per
`core/ADAPTER-CONTRACT.md`, mark `unavailable`/`not-applicable` with real
reasons where nothing viable exists, then:

```
${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all --project <project>
```

Same rule as bootstrap: nothing gets marked `implemented` without passing
its own known-pass/known-fail run.

Include the three workflow adapters the same way `core/skills/bootstrap/SKILL.md`
§4 describes — from the surveyor's `## Workflow adapters` section: `ticket-fetch`
(the tracker the digest identifies — write `.spine/ticket-pattern.conf` from the
observed key shape it reports), `open-pr` (the PR host it identifies), and
`worktree-prep` (the gitignored dependency dir(s)/package manager it reports);
mark any of them `not-applicable` with a real reason if the digest found no
such tool or nothing to provision.

Also write `.spine/branch-naming.conf` from that same section's observed
branch-naming template — present it for confirm-or-edit rather than asking
cold, same as the class/protected-path proposals above. If the digest found
no consistent pattern, propose the plain default `{ticket}-{slug}` instead of
guessing. Either way, **the confirmed template must contain `{ticket}`**
(same requirement as `core/skills/bootstrap/SKILL.md` §4) — if the engineer's
edit drops it, say why that breaks the "already branched by hand" detection
and ask again rather than writing it as given.

## 4. The map and charter draft

Produce `<project>/docs/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/map.md`, populated from the surveyor's
`## Map content` section: real module boundaries, real data flow for the
flows that matter most, real entry points, and its `## Known weirdness`
subsection verbatim in spirit — say what it found plainly (half-finished
migrations, undocumented invariants, dead code, naming that's drifted from
reality). Surface these, don't fix them and don't hide them — that's what
"known weirdness" is for. Stamp it with the surveyor's captured `HEAD` sha
and the current timestamp.

Draft `<project>/docs/charter.md` from
`${CLAUDE_SKILL_DIR}/../../templates/charter.md`, inferred from the
surveyor's `## Charter draft material` section — marked `DRAFT`, its
`Source documents:` line filled in with the real doc path(s) that section
named (or left `none` if it found no real source document, only inference
from code structure). **State plainly to the engineer that this draft must
be edited before it's trusted** — an inferred charter is a starting point,
not a substitute for the engineer's own non-negotiables. If the surveyor's
digest carries a non-empty `## Uncategorized source material` section, call
it out to the engineer explicitly and separately from the rest of the
handoff — these are real statements from a source document that didn't fit
any of the charter's four buckets; the engineer decides whether each
belongs in the charter, in a `docs/decisions/` record, or nowhere. Don't
let this list quietly ride along inside the general "edit this draft"
instruction — a human skimming the draft charter has no way to notice
something that was never written into it at all.

The survey itself is already bounded at the source (§0.5, and the
surveyor's own instructions) — this step is just writing up what came back,
not a second pass of reading the repository.

## 5. Wire the install

Identical to `core/skills/bootstrap/SKILL.md` §5 — the same symlink set
(`.claude/skills/`, `.claude/agents/`, `.claude/rules/`, `.claude/hooks`),
the same `.claude/settings.json` hook wiring, `docs/decisions/.gitkeep`,
`work/.gitignore` (same selective-ignore content as bootstrap §5).

**`CLAUDE.md`, unlike greenfield, may already exist and already carry real
content** (engineering conventions, a CQRS pattern, whatever the team wrote
before spine existed) — unlike a brand-new project, `/adopt` must never
discard or relocate it. Always write the
`${CLAUDE_SKILL_DIR}/../../templates/CLAUDE.md` content, filled in for this
project, as a `<!-- spine:begin -->`/`<!-- spine:end -->` block at the very
top of the file — see the template's own header comment — **regardless of
whether `<project>/CLAUDE.md` already exists.** If it doesn't exist yet,
the block is simply the entire file for now; still write it with the
markers, never as the bare unmarked template — the moment this engineer
starts writing their own content below `spine:end`, it must never be
limited by the 60-line cap or need a later migration into markers just
because the file happened to start out empty. If it already exists: leave
every byte below `<!-- spine:end -->` untouched. The 60-line cap applies
only to what's inside the markers; content below it never counts against
it and is never edited, reformatted, or moved to another file by this
skill. A recalibration re-writes only the block between the markers (same
rule `/ratchet` already follows), never the content below it.

## 6. Commit and hand off

Same as `core/skills/bootstrap/SKILL.md` §6 — one setup commit for
everything this skill wrote, then tell the engineer to start a fresh
session in `<project>` before running anything else (hook/skill wiring does
not hot-reload mid-session). Note explicitly in your final message that any
existing uncommitted work in `<project>` was left untouched by this install
— confirm that's still true before finishing, don't just assert it.
