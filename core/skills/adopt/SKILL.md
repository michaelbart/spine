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
constraint (build prompt §1), actively harmful. Stay bounded.

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
answers **whether this project serves a browser UI a person looks at**; if
yes, confirm its proposed view/component glob(s) for `.spine/ui-paths.conf`
and dev-server start command + port. If no: mark `ui-render`
`not-applicable`, reason "no browser UI in this project's runtime shape."
Generate `.spine/adapters/<name>` for each of the 18 capabilities per
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
surveyor's `## Charter draft material` section — marked `DRAFT`. **State
plainly to the engineer that this draft must be edited before it's
trusted** — an inferred charter is a starting point, not a substitute for
the engineer's own non-negotiables.

The survey itself is already bounded at the source (§0.5, and the
surveyor's own instructions) — this step is just writing up what came back,
not a second pass of reading the repository.

## 5. Wire the install

Identical to `core/skills/bootstrap/SKILL.md` §5 — the same symlink set
(`.claude/skills/`, `.claude/agents/`, `.claude/rules/`, `.claude/hooks`),
the same `.claude/settings.json` hook wiring, the same `CLAUDE.md` ≤60-line
generation and line-count audit, `docs/decisions/.gitkeep`, `work/.gitkeep`.

## 6. Commit and hand off

Same as `core/skills/bootstrap/SKILL.md` §6 — one setup commit for
everything this skill wrote, then tell the engineer to start a fresh
session in `<project>` before running anything else (hook/skill wiring does
not hot-reload mid-session). Note explicitly in your final message that any
existing uncommitted work in `<project>` was left untouched by this install
— confirm that's still true before finishing, don't just assert it.
