# PR-description patch — Phase A: inventory and assembly plan

Companion patch to the readability patch (`readability-phase-A/B-handoff.md`,
merged `4b541e2`) — `/ship` gains a step that assembles a PR description
from this task's own verified artifacts, so a reviewer gets "review this at
the plan level" plus a mechanically-computed pointer list instead of an
unaided diff scroll. Same discipline as that patch: inventory first, decide
against real evidence, write the plan, stop.

## 1. How `/ship` currently ends — confirmed against `core/skills/ship/SKILL.md`

`/ship` does **not** create PRs and has no `gh`/hosting-API call anywhere in
it. Its actual endpoint, read line by line:

- §4 writes `work/<task-id>/briefing.md` (the human-facing summary).
- §5 ("Ledger and commit") does the ledger harvest, then `git add` +
  `git commit` — **single-repo: one commit. Multi-repo (Extension B): one
  commit per repo in `## Ship order`, staged and ordered, plus one for the
  workspace root** — then runs `propagate`. Its own explicit closing line:
  *"Do not push, in either case. Committing locally is this skill's job;
  pushing or opening a PR is the human's call, made after reading the
  briefing."*
- §6 writes `state` = `done`, runs `registry-sync`, clears
  `.spine/current-task`, and tells the human where the briefing is.

**Conclusion, per the build prompt's own instruction to meet the actual
workflow:** there is no PR-creation call to populate. The new step writes
`work/<task-id>/pr-description.md` and, at the same point `/ship` already
tells the human where the briefing is (§6), also tells them where the PR
description is. It does not shell out to `gh pr create` — that would be
new tooling this patch has no mandate to add, and would silently assume
every installed project uses GitHub, which nothing in `core/` currently
assumes (no `gh` reference exists anywhere in `core/`, confirmed by grep).

**Insertion point, chosen to avoid renumbering:** a new `## 4a. Write the
PR description` section, directly after `## 4. Write the delta briefing`
and before `## 5. Ledger and commit` — same moment the build prompt asks
for (after briefing generation, before the commit), same sources. I did
**not** renumber `§5`/`§6` to `§6`/`§7`: three files outside `ship/SKILL.md`
reference its sections by number (`core/scripts/claims-check` → §0,
`core/templates/writing-mandate.md` → §4, `docs/tradeoffs.md` → §1,
`docs/example/.../deviations.md` → §0), and touching those cross-references
would be scope creep for a patch that's supposed to be surgical. There's
already a precedent for a lettered insertion in this exact codebase —
`core/skills/verify/SKILL.md`'s `## 1b. Contract conformance` sits between
`§1` and `§2` the same way. `4a` follows that convention.

## 2. Source-artifact formats — confirmed by reading the real templates/scripts, not assumed

| Artifact | Real shape (confirmed) |
|---|---|
| `plan.md` | `## The gist` (prose, first sentence = headline), `## What could go wrong` (2-3 real risks or "little"), `## What I'll decide alone vs. stop and ask` (3 tiers, tag keywords `decide-alone`/`record-and-proceed`/`halt`), `<!-- MACHINE: predicted-touch -->`-fenced `## Predicted touch` (bare paths, one script — `conformance` — actually parses this), `## Grounds on decisions` / `## Ship order` / `## Contract change` (all optional, omitted-not-empty) |
| `deviations.md` | `## Deviation N` records: `- Tier:`, `- Status: open\|resolved`, `Plan assumed`, `Actually true` (prose, told to cite file:line or command+output but **not a structured field** — real records backtick-wrap the paths they cite, e.g. `` `src/lib/slug.test.ts` ``), `Options considered`, `Recommendation`, `Resolution` |
| `verify.md` | Floor results table, `## Conformance` (one summary line: `predicted=n actual=n precision=p recall=r f1=f` — **counts only, not the file lists**), `## Adversary verdicts` (`### Falsifier` / `### Security`, `Attacked:` list, `Verdicts kept: n · dropped: n`, one entry per kept verdict: claim/severity/evidence), `## Capability gaps`, `## Tooling gaps`, `## Contract conformance` (multi-repo only) |
| `work/<task-id>/artifacts/conformance.json` | `{predicted: [...], actual: [...], precision, recall, f1}` — the **real file lists** verify.md's prose line summarizes. This is the mechanical source for drift, not verify.md itself. |
| `work/<task-id>/artifacts/<agent>-verdict.json` (post-`verdict-filter`) | Each kept verdict: `{claim, severity, evidence: {kind: "file_line", file, line} \| {kind: "command", command, output}}`. **`file_line` evidence is a real structured field** — confirmed against `core/agents/falsifier.md`'s own output contract and against real raw verdict JSON in `~/horizon/work/20260808-.../artifacts/{falsifier,security}-verdict-raw.json` (5 of 7 real verdicts there carry `file_line` evidence with exact file+line). |
| `work/<task-id>/artifacts/contract-touch.json` | Per touched contract: name, `spec_change` (additive/breaking/unchanged), `registry_stale`, `consumers_in_blast_radius[]` — multi-repo only |
| `work/<task-id>/ledger.json` | Flat key-value, freeform (`ledger set` does no schema validation). Fields already in use: `bypass`, `second_approver`, `ship_time_regrounding`, `ship_time_regrounding_claims`, `claims_conflicts`, `class_declared`, `class_escalated`, `deviation_count`, `conformance_score`, `capability_gaps`, `tooling_gaps`, `hand_tracked` |
| `work/<task-id>/approval.json` | `{approver, at, override, override_reason}` — Class 2 self-approval override lives here |
| `.spine/protected-paths.conf` | One glob per line, `#` comments, `#migration`/`#manifest` tags — confirmed against `~/bgr`'s real file |
| `claims-check --diff` output | Tags each collision `[DECLARED]` or `[UNDECLARED — not in this task's own claims.json]` (confirmed by grep against `core/scripts/claims-check`); ship-time `[UNDECLARED]` hits are already turned into a halt-tier `deviations.md` record by `ship/SKILL.md` §0 before this step ever runs — so they surface a second time, automatically, through the deviations union below, not through a separate claims-check read. |

`briefing.md` is confirmed to be the nearest sibling exactly as the build
prompt expects: same sources, same moment (`/ship` §4, immediately before
the new §4a), and its own header comment already states no script reads it
back in — meaning the new PR-description step can read the same underlying
artifacts `briefing.md` just finished quoting, not `briefing.md` itself
(reading briefing.md as a source would make the PR description a summary
of a summary — one extra hop of exactly the compression the build prompt's
"nothing omitted, only ordered" rule and this patch's own no-re-derivation
rule both forbid). `pr-description.md` and `briefing.md` are siblings that
share sources, not one derived from the other.

## 3. Multi-repo reality — decision

**One shared PR description per task, plus one line per repo's own commit
position** — the build prompt's own recommendation, and the only option
consistent with how the record is actually scoped: `plan.md`, `verify.md`,
`deviations.md`, the ledger are all **one task-scoped set of files at the
workspace root**, never duplicated per repo (confirmed: `## Ship order`
lives in the one `plan.md`; `work/<task-id>/artifacts/contract-touch.json`
is one file at the workspace root; per-repo data that exists —
`conformance-<repo>.json`, per-repo floor tables in `verify.md` — is keyed
by repo *within* the one file set, not split into separate task folders).
Writing N divergent descriptions from one record would either (a) silently
re-derive per-repo content that isn't actually per-repo-scoped in the
source artifacts, or (b) copy-paste the same content N times with no real
difference — both worse than one shared body.

**Concretely:** `pr-description.md` is written once, at the workspace root,
same as `briefing.md` already is. A new `**Contracts**` section (already
in the build prompt's template) carries ship order and this PR's position
in it (*"2 of 3 — API ships after schema, before frontend"*), sourced from
`## Ship order` (plan.md) + `contract-touch.json`. Per repo, at commit time
in §5's existing per-repo loop, the human is told (same place §6 already
points at the briefing) that this repo's commit carries
`Spine-Task: <task-id>` and its PR should link/reference
`work/<task-id>/pr-description.md` at the workspace root — no per-repo copy
of the file is written. This has zero real-data proof available in this
environment (Phase A confirmed, same as the readability patch's own
disposition 4: neither `~/horizon` nor `~/bgr` has a `workspace.json`) —
disclosed, not faked, in Phase B, same as last time, and added to the same
first-real-multi-repo-task watch-item.

## 4. The "Where to look" section — the four unions, made mechanical

This is the patch's center of gravity per the build prompt, and the one
place "computed, not composed" has to survive contact with real,
messy artifacts. Each union, and the concrete rule after checking real
data:

**(1) Every file cited in a deviation.** `deviations.md` has no structured
file field — confirmed above. Rule: for each record, extract every
backtick-wrapped token that contains `/` or a recognizable extension
(`\.[a-zA-Z0-9]{1,8}` — a real, if approximate, dotfile/extension test).
Checked against the real slugify deviation record: it correctly pulls
`` `src/lib/slug.test.ts` `` and `` `work/20260811-.../plan.md` `` while
correctly *skipping* backtick-wrapped non-paths in the same record
(`` `"!!!"` ``, `` `""` ``, `` `"   "` `` — quoted test inputs, no `/` or
extension). This is a fixed extraction rule, applied identically to every
record — "script-like reading," not judgment about which file seems more
important. Each hit pairs with that deviation's own one-line "Actually
true" opening clause as the why.

**(2) Every file:line in an adversary finding.** Real structured data
exists for this, confirmed above — `evidence.kind == "file_line"` on a
**kept** verdict (never a dropped one) in
`work/<task-id>/artifacts/<agent>-verdict.json`. Rule: union
`evidence.file`/`evidence.line` across every kept verdict of every
adversary that ran, paired with that verdict's `claim` (first clause, one
line). **`evidence.kind == "command"` verdicts contribute nothing to this
list** — there's no file to point at, only a command to rerun — even when
severity is high. Checked against the real horizon security verdict 1 (the
`.claude/settings.json` permission-escalation finding, `severity: high`,
`evidence.kind: command`): it does **not** appear in Where to look under
this rule. It still appears in full in "How it was verified: Adversaries"
(count + max severity + gist + pointer, the floor-protected line) — so
it's never lost, just not double-counted as a file pointer it doesn't
actually have. Documented as a real, disclosed tradeoff below, not a gap
found later.

**(3) Every touched protected path.** Mechanical: `git diff --name-only
<base>..HEAD` (same diff conformance/contract-touch already compute) tested
against `.spine/protected-paths.conf`'s globs (same match semantics
`path-escalate` and `plan.md`'s own step-3 check already use — a real diff
against a real glob list, not a new matcher). This is the one union with
no prior-art source file (plan-time protected-path checking happens against
*predicted* touch, in `task/SKILL.md` §3; this is the same check re-run
against the *actual* diff, at ship time — same rule, different input,
already the exact relationship `/ship`'s own ship-time re-grounding in §0
establishes for `check-stale`/`claims-check`).

**(4) Every out-of-plan touch from conformance drift.** Mechanical source:
`work/<task-id>/artifacts/conformance.json`'s `actual` array minus
`predicted` array (real set difference, not verify.md's summary line, which
only carries counts). **Confirmed problem in real data:** the real slugify
`conformance.json` has `predicted=[2 files]`, `actual=[14 files]` — 12 of
those 14 are the task's own bookkeeping (`work/<task-id>/**`,
`.spine/current-task`, `work/<task-id>/owner`, etc.), not code drift. A
naive set difference would flood Where to look with the task's own record-
keeping. **Rule, disclosed explicitly rather than silently applied:**
exclude any `actual`-only entry under `work/<task-id>/` or `.spine/` before
computing this union — those paths are already the record the closing
"Record:" line points at, never reviewable code. This exclusion is itself
a fixed prefix rule (identical every time, not "does this file look
important"), and it's named in the template's own comment (per the
self-red-team's own "the four-unions rule travels with the artifact" ask)
so a future editor sees why it's there. `[UNDECLARED]` hits are covered by
union (1) already, per §2's table note above — restated here so the
build prompt's explicit mention of the tag has a direct answer: it is not
a fifth source, it's a real record that lands in deviations.md and gets
picked up by rule (1) the same way any other deviation does.

**Empty-case wiring, confirmed against the template text the build prompt
already specifies:** when all four unions are empty, the section renders
the fixed "landed exactly where the approved plan predicted... spot-check
at will" line — never an empty bullet list, never a fabricated pointer.

## 5. Demonstration sources, confirmed available for Phase B

- **`~/bgr/work/20260811-extract-slugify-helper/`** — real plan.md (new
  template), real resolved `record-and-proceed` deviation, real verify.md
  (floor PASS, adversaries deliberately not run — disclosed in verify.md
  itself), real `conformance.json` with the bookkeeping-noise problem union
  (4) needs to handle, real briefing.md. Single-repo, Class 1, `state`:
  `done`. This is the "richest single-repo record" the build prompt asks
  for on the new template.
- **`~/horizon/work/20260808-fix-building-group-delete-orphans-units/`** —
  real, rich adversary output: 4 falsifier + 3 security kept verdicts
  (unfiltered — `verdict-filter` was sandbox-blocked for this task, all 7
  reported), including a real high-severity `command`-evidence finding
  (exercises union (2)'s disclosed limit directly) and multiple real
  `file_line`-evidence findings (exercises union (2)'s main path). Floor
  **FAILed** (pre-existing lint debt) — this task never actually reached
  `/ship`; its original `briefing.md` was hand-written for that reason, and
  the readability patch already re-rendered a plan/briefing pair from this
  same frozen record on the new template with that exact caveat disclosed
  up front. Phase B follows the same precedent: assemble a PR description
  from this record too, with the same disclosure (this is what the
  assembly would have produced had `/ship` reached this step, not a claim
  that it did).
- **Empty-case proof**: neither real task above has zero drift/deviations/
  findings all at once (slugify has one resolved deviation; horizon has
  adversary findings and no conformance run at all). Phase B needs a third,
  synthetic-but-labeled minimal record (or a deliberately trimmed replay of
  slugify with its one deviation section removed) to prove the empty-case
  line renders correctly — will construct and label it clearly as
  synthetic, not present it as a real task.

## 6. What Phase B will ship

- `core/templates/pr-description.md` — new template, structure exactly as
  specified in the build prompt, with the four-unions rule and the two
  disclosed exclusions (command-evidence, bookkeeping-path prefix) written
  into its own comment so they travel with the artifact.
- `core/skills/ship/SKILL.md` — new `## 4a. Write the PR description`
  section (no renumbering, per §1 above); one added line in `§6` pointing
  the human at `pr-description.md` alongside the existing briefing pointer;
  one `ledger set <task-id> pr_description "generated"` call (freeform key,
  confirmed safe against `ledger set`'s real implementation — no schema to
  extend).
- `README.md` — `/ship`'s command-reference row gets one clause noting the
  PR description output, matching the existing row's style.
- Demonstration: real assembly against slugify + horizon, a labeled
  synthetic empty-case record, a full trace audit (every sentence in one
  generated description → its source artifact, in the handoff), self-red-
  team per the build prompt's four named risks.

No hooks, scripts, agents, or capabilities change. This is a template +
skill-prose patch, same class as the readability patch — one pin bump.

---

**Stop for review.**
