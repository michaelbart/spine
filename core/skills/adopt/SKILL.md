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

## 1. Layer 1 calibration

Identical to `core/skills/bootstrap/SKILL.md` §2 — check
`~/.spine/user-config.json` first, confirm-or-override if present, full
interview only if absent.

## 2. Layer 2 calibration — inspect first, then confirm

Unlike greenfield, you can infer here. Read the repository layout and
propose a concrete `.spine/protected-paths.conf`: auth/authorization code,
public API surface, payment/PII handling, package manifests, anything that
looks like a migrations directory — present your inferred list to the
engineer as something to edit, not a blank interview. Confirm risk posture
(default: any protected-path touch forces Class 2) and CI integration
(default: mirror locally; note what real CI already does, if any, so the
floor can be checked against it for "mirror-or-exceed, never less").

## 2.5 Layer 2.5 — team strictness profile

Same as `core/skills/bootstrap/SKILL.md §3.5` — write and `profile-check`
`<project>/.spine/profile.json`. Unlike greenfield, **infer a starting preset**
from the repo and present it for confirmation: a codebase with real
migrations/auth/payments leans **regulated**; a scratch or spike repo leans
**prototype**; otherwise **standard**. On a recalibration, show the existing
profile and confirm-or-edit rather than re-asking cold.

## 3. Layer 3 calibration and adapter generation

Same process as `core/skills/bootstrap/SKILL.md` §4 — infer stack and
commands from the repo (lockfiles, config files, existing CI config) and
present findings for confirmation rather than asking cold; ask runtime
shape, **and specifically whether this project serves a browser UI a
person looks at** (infer first from the repo — a frontend framework
dependency, a `views`/`components`/`templates` directory, an
`index.html` served by a dev server — and present that inference for
confirmation, same inference-first posture as everything else in this
step). If yes: confirm the view/component glob(s) for
`.spine/ui-paths.conf` and the dev-server start command + port. If no:
mark `ui-render` `not-applicable`, reason "no browser UI in this
project's runtime shape." Generate `.spine/adapters/<name>` for each of
the 18 capabilities per `core/ADAPTER-CONTRACT.md`, mark `unavailable`/
`not-applicable` with real reasons where nothing viable exists, then:

```
${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance --all --project <project>
```

Same rule as bootstrap: nothing gets marked `implemented` without passing
its own known-pass/known-fail run.

Include the three workflow adapters the same way `core/skills/bootstrap/SKILL.md`
§4 describes — infer them from the repo: `ticket-fetch` (the tracker the repo's
commits/branches already reference — write `.spine/ticket-pattern.conf` from the
observed key shape), `open-pr` (the PR host the repo already uses), and
`worktree-prep` (symlink/reuse the gitignored dependency dir(s) already visible
in the repo's own lockfiles/`.gitignore`); mark any of them `not-applicable`
with a real reason if the repo shows no such tool or nothing to provision.

## 4. The bounded survey

Produce `<project>/docs/map.md` from
`${CLAUDE_SKILL_DIR}/../../templates/map.md`: real module boundaries, real
data flow for the flows that matter most, real entry points — and a
`## Known weirdness` section that says what it finds plainly (half-finished
migrations, undocumented invariants, dead code, naming that's drifted from
reality). Surface these, don't fix them and don't hide them — that's what
"known weirdness" is for. Stamp it with the current `HEAD` sha and
timestamp.

Draft `<project>/docs/charter.md` from
`${CLAUDE_SKILL_DIR}/../../templates/charter.md`, inferred from what the
survey actually found (README, existing docs, the map you just wrote) —
marked `DRAFT`. **State plainly to the engineer that this draft must be
edited before it's trusted** — an inferred charter is a starting point, not
a substitute for the engineer's own non-negotiables.

Bound the survey itself: this is not a request to read the whole
repository. Enough to populate the map's real sections and a charter draft
with genuine content — when you notice you're going deep enough that this
is starting to look like task-level research, stop; that depth belongs to
`/task`'s research phase on the task that actually needs it.

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
