---
name: workspace
description: Initialize or extend a spine workspace root that coordinates multiple repositories through declared contracts. Run once per workspace (greenfield, right after /design decides a multi-repo repo-topology) or to register an existing set of repos (brownfield). Not run per task.
disable-model-invocation: true
argument-hint: --root <path> [--repo <name>=<path> ...] [--from-design <project-path>]
---

You are running `/workspace` (build prompt §2, Extension B). Scripts at
`${CLAUDE_SKILL_DIR}/../../scripts/<name>`, templates at
`${CLAUDE_SKILL_DIR}/../../templates/<name>`.

**This is setup, not a task** — it produces zero `work/<task-id>/` folders,
runs no floor, needs no class. It runs once per workspace, the same
non-recurring-event shape `/bootstrap` and `/design` already are. A project
that never runs this skill is completely unaffected by anything below —
its own `.claude/settings.json` has no `workspace.json` beside it, and
every hook this skill's siblings extend falls back to exactly its
pre-Extension-B behavior the moment `workspace.json` is absent. Verify that
fallback for real before calling this skill done (§6 below).

## 0. Preflight — which mode

**First, before either entry path below: does `<root>/workspace.json`
already exist?** If so, this is an **extend**, not a create — the
template's own header comment states this outright ("never hand-edited
except through `/workspace`'s own re-run") and this section is what makes
that re-run actually safe rather than merely asserted. Read the existing
`workspace.json` in full before doing anything else. Confirm the new
repo(s) named this invocation aren't already listed — compare by absolute
`path`, not just `name` (catches a rename or a re-registration attempt) —
and tell the human plainly if one already is, rather than silently
duplicating or silently no-oping. Extend mode changes §1, §2, and §4 below
exactly as each says; §3, §5, and §6 are unchanged by it (§3's contract
registration and §6's fallback check care about repos and contracts, not
about whether this is the workspace's first or fifth repo).

If `workspace.json` does not exist yet, this is a **create** — proceed to
the two entry paths below exactly as written.

Two entry paths, mutually exclusive per invocation (apply to both create
and extend — an extend can equally arrive via `--from-design`, if a later
design/repo-topology decision names a repo an earlier workspace session
didn't know about yet):

- **`--from-design <project-path>`** (greenfield): that project just ran
  `/design` and its `repo-topology` decision
  (`docs/decisions/D-<n>-*.md`, `Category: repo-topology`) says "more than
  one repository." Read that record's `## Decision` and `## Context`
  sections and present them to the human — decision.md's body is prose, by
  design (§1 of the build prompt's own decision-record shape has no
  structured repo-list field, and inventing one now would be a second,
  parallel schema for something one conversation turn resolves better) —
  `core/templates/decision.md` itself is unchanged by this skill, so
  this step is a **facilitated confirmation**, not a parse: ask the human
  to name each repo the decision implies (name, absolute path — create the
  directory if it doesn't exist yet, role) one at a time — build prompt §2
  is explicit that decision.md's own `## Decision` prose is what's
  authoritative here, this step just confirms it against the human rather
  than re-deriving it — and confirm the original project itself becomes
  one of the member repos (usually the first-named one). Each named repo
  still needs its own `/bootstrap` run
  before it has real capabilities — say so plainly, this skill does not
  install spine into member repos, only coordinates already-installed (or
  about-to-be-installed) ones.
- **`--root <path> --repo <name>=<path> [--repo <name>=<path> ...]`**
  (brownfield): the human names existing repos directly, no design session
  involved. Confirm each path exists and is a spine-installed project
  (`.spine/capabilities.json` present) before proceeding — if one isn't
  installed yet, tell the human to `/bootstrap` or `/adopt` it first; this
  skill registers repos, it does not install spine into them (build prompt
  §2: "each repo keeps its own adapters and capability manifest, correctly,
  because the stacks differ").

Either path ends with the same set of facts: a workspace root path, and a
list of `(name, absolute path, role)` triples. Everything from §1 onward is
identical regardless of which path got you there.

## 1. Create the workspace root — or extend it

**Create mode** (`workspace.json` did not exist at §0):

`mkdir -p <root>/{contracts,work,docs/decisions,.spine}`. `cd <root> && git
init` — the workspace root is its own small git repo (workspace.json,
contract specs, work/, docs/), never a superset of any member repo's own
history. This is required, not optional: `contract-touch` (§4 of the build
prompt) detects a touched contract by diffing `contracts/<name>/spec*`
against history, and there is no history to diff without a real repo here.

Write `<root>/workspace.json` from
`${CLAUDE_SKILL_DIR}/../../templates/workspace.json`: one `repos[]` entry
per triple from §0 (`producer_paths_match_count` and `contracts[]` both
start empty — no contract exists yet at init time, `/task` or a dedicated
follow-up declares the first one). Write `<root>/.spine/protected-paths.conf`:

```
workspace.json#migration
contracts/**
```

(Both tagged implicitly Class-2-only by `path-escalate`'s existing
protected-path enforcement — no `#migration` tag on `contracts/**` itself,
since a contract spec changing is not a schema migration, but it is exactly
the kind of cross-repo, high-blast-radius edit Class 2 exists for. Tagging
`workspace.json` itself `#migration` — flat deny outside Class 2, no soft
escalation path — because a change to the repo/contract topology mid-task
is never something to improvise past.)

Write `<root>/docs/charter.md` — the **one system charter** (build prompt
§2: "one system charter at the workspace; per-repo maps as today"). If this
is the greenfield path, this is a short pointer document: what the system
is, and a one-line reference to each member repo's own charter/map for
stack-specific detail — not a duplicate of any member repo's charter. Draft
it, human confirms, same `DRAFT`→`CONFIRMED` protocol `/bootstrap`'s own
charter uses.

Write `<root>/CLAUDE.md` — thin, same line-budget discipline as
`core/templates/CLAUDE.md` (hard cap 60 lines): what this workspace
coordinates, that `/task` here spans repos via `permissions.
additionalDirectories`, and a pointer to `workspace.json` and
`contracts/` rather than restating their contents.

**Extend mode** (`workspace.json` already existed at §0):

`cd <root> && git init` — safe to re-run unconditionally; git no-ops on an
already-initialized repo rather than erroring or resetting history. Skip
`mkdir` (the directories already exist).

**Append, never regenerate, `<root>/workspace.json`.** Read the file,
add one new `repos[]` entry per newly-named triple from §0
(`producer_paths_match_count` on any *existing* `contracts[]` entry stays
exactly as it was — a new consumer repo does not by itself change an
existing contract's producer-path count), and write the file back whole.
**Every existing `contracts[]` entry — spec_path, spec_hash,
producer_paths, consumers, registered date — must come out byte-for-byte
identical to how it went in.** This is the one genuinely destructive
mistake this mode exists to prevent: writing `workspace.json` fresh from
the template here, the same way create mode does, would silently discard
every contract this workspace has already declared. If a new repo is
itself a *consumer* of an already-declared contract, that is a second,
explicit edit — append its name to that contract's `consumers` array —
never inferred silently just because the repo joined the workspace.

Do not touch `.spine/protected-paths.conf`, `docs/charter.md`, or
`CLAUDE.md` — they already exist and describe what this workspace *is*,
which a new member repo joining does not change.

## 2. Wire the install — same mechanism as `/bootstrap` §5, plus one line

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --project <root>
```

(Extension C §2.1 — this is the same per-machine step `/bootstrap`/`/adopt`
now delegate to; the workspace root is a spine-consuming session like any
other, primitive §0.4 (skills/agents resolve from the *session's own*
startup directory, not from added directories) is exactly why it needs its
own copy of this wiring rather than inheriting a member repo's. `setup`
also initializes this workspace root's own `.spine/core-pin.json` and adds
`.claude/skills/`, `.claude/agents/`, `.claude/rules/`, `.claude/hooks` to
`<root>/.gitignore` — they're generated locally now, never committed. Safe
to re-run in extend mode too — idempotent against an already-wired root.)

**Create mode**: write `<root>/.claude/settings.json`: the same three-hook
`PreToolUse` wiring `/bootstrap` writes, **plus**:

```json
{
  "permissions": {
    "additionalDirectories": ["<member-repo-path-1>", "<member-repo-path-2>", "..."]
  }
}
```

**Extend mode**: read the existing `<root>/.claude/settings.json` and
append the new repo's path(s) to the existing
`permissions.additionalDirectories` array — write the file back whole,
same append-never-regenerate discipline as §1's `workspace.json` edit.
Leave every other key (the three hook entries, anything else already
there) untouched.

This is primitive §0.1(c)'s confirmed mechanism — added directories become
readable/editable under the session's permission mode, and the *workspace
root's own* hooks (not any member repo's) are what fire on edits targeting
them. Do not additionally symlink `.claude/` into any member repo — per
primitive §0.1(b), a hook defined only in an added directory's own settings
never fires from the workspace session, so there would be nothing to
symlink for *this* purpose; each member repo's own `.claude/` (if it's
independently spine-installed) continues to matter only for sessions
started directly inside that repo.

## 3. Register repos and (optionally) the first contract

`workspace.json`'s `repos[]` is already written (§1). If the human already
knows the first contract this workspace exists to coordinate, declare it
now (same shape a later `/task` would use): create
`<root>/contracts/<name>/spec.md` (or whatever spec format fits the
producer's stack — stack-specific content, per build prompt §2, is fine
here, the registry entry around it is what's stack-blind), compute its hash
(`${CLAUDE_SKILL_DIR}/../../scripts/decision-hash <spec-path>` — the same
whole-file hash algorithm, no status-line exclusion needed since a spec has
no status line), and append a `contracts[]` entry to `workspace.json` with
real `producer_paths` and the current match count:

```
git -C <producer-repo-path> ls-files -- <producer_paths glob> | wc -l
```

If no contract is known yet, leave `contracts: []` — a workspace with zero
declared contracts is a legitimate, if unusual, starting state; the first
task that adds a real cross-repo dependency is what should declare one, not
this skill speculatively.

## 4. Commit

**Create mode:**

```
git add -A -- workspace.json .spine/ contracts/ docs/ work/ CLAUDE.md .claude/
git commit -m "spine: workspace init — <n> repos, <m> contracts"
```

**Extend mode** — its own distinct message shape, so the event feed
(`render-dashboard`'s event kind, `/costs`) can tell "this workspace was
created" from "this workspace grew" apart rather than collapsing both into
one indistinguishable "workspace init" line:

```
git add -A -- workspace.json .claude/settings.json
git commit -m "spine: workspace extend — added <repo-name>[, <repo-name>...]"
```

Both untrailered, same precedent as `/bootstrap`/`/design`'s own install
commits — there is no task ID yet.

## 5. Hand off

Tell the human: the repos registered (or, in extend mode, the repo(s) just
added and the workspace's now-complete member list), any contract
declared, and that a fresh session must start **in the workspace root**
(not any member repo) before running `/task` for cross-repo work — hook/
skill wiring does not hot-reload mid-session, identical caveat to
`/bootstrap`'s own §6, and it applies exactly as much to an extend as to a
first-time create: a session already running against the old
`additionalDirectories` list will not see the newly-added repo until a
fresh session starts. A member repo remains independently usable for its
own local, single-repo tasks
exactly as before — joining a workspace does not revoke that; a session
started directly inside the member repo never sees `workspace.json` at all
and every hook there runs its plain single-repo path.

**Also tell the human, explicitly, if this is the workspace root's first
session, or a newly-added member repo's very first time appearing in
`additionalDirectories`**: Claude Code's own project-trust layer silently
drops `permissions.additionalDirectories` (and `permissions.allow`) for a
directory that has never been opened in an interactive session before —
confirmed for real in Phase D, where this produced a read-permission
denial on every member-repo path, before any hook ever ran, with no
mention of hooks or `workspace.json` in the error at all. This applies per
*directory*, not once per workspace — extend mode adding a repo that has
never itself been opened interactively hits the identical silent-drop,
even though the workspace root itself was trusted long ago. Run `claude`
(interactively, no `-p`) directly in that repo's own path once and accept
the trust dialog — or set `hasTrustDialogAccepted: true` for its path in
`~/.claude.json` — before the first real `/task` session touching it, or
before fire-testing anything here. This is a Claude Code primitive, not
something `/workspace` itself can do on the human's behalf.

## 6. Verify the fallback, once, before calling this done

From the workspace root, confirm (read `docs/tradeoffs.md`'s Auto Mode
classifier wall section first — this needs a real nested `claude` session,
which this session's own Bash tool cannot invoke; hand the commands to the
engineer via `!` passthrough, per `ext-phase-A-handoff.md` §5): a session
started **inside a member repo directly** (not the workspace root) still
sees that repo's own protected-paths/hooks behave exactly as they did
before this workspace existed — no `workspace.json` in scope, no
`additionalDirectories`, nothing new. This is the regression check for the
mechanism this skill itself introduces, distinct from Phase D's own
project-wide single-repo regression demonstration.
