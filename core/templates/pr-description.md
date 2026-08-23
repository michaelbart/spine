<!--
Assembled by /ship §4a, immediately after briefing.md (same sources, same
moment) and before the commit. Every sentence in the rendered output must
trace to a specific line in a specific already-written artifact — plan.md,
deviations.md, verify.md, the ledger, work/<task-id>/artifacts/*.json. This
file quotes and computes; it never re-reads the diff and summarizes it. A
post-hoc summary is the implementer's own account of its own work — the
one thing this system's adversaries are structurally forbidden from
producing — and it inherits the author's blind spots. If a section's
source artifact doesn't exist, the section says so or is omitted per its
own rule below; it is never filled in by looking at the code.

**The two floors — never compressed below these, same rule as
briefing.md, restated here because this file is the one a reviewer who
never opens briefing.md still sees:**
- Adversary findings: count + max severity + one-line gist each, plus a
  pointer into verify.md. Never compressed further, regardless of length.
- Overrides/bypasses: never folded into a single line that could bury one,
  never omitted when real.

**"Where to look" is computed, not composed — this is the section this
whole template exists for.** It is the union of exactly four mechanical
sources, each read the same way every time:

1. Every file cited in a `deviations.md` record. `deviations.md` has no
   structured file field — extract every backtick-wrapped token in the
   record's body that contains `/` or a recognizable extension
   (`\.[a-zA-Z0-9]{1,8}`), paired with that deviation's own "Actually true"
   opening clause as the why. If two extracted tokens from the same record
   are suffix-related (e.g. `slug.test.ts` and `src/lib/slug.test.ts`),
   keep only the longer, more path-qualified one — a fixed tiebreaker, not
   a judgment call, and it's a real case: it fires on this patch's own
   demonstration record (see the Phase B handoff's trace audit). **v1
   heuristic, not a parser — pre-loaded
   ratchet trigger**: the first time this heuristic demonstrably misses a
   real file a deviation cites, the fix is a structured `- Files:` line
   added to `core/templates/deviations.md` itself, not a smarter regex
   here (see `docs/tradeoffs.md`'s PR-description-patch self-red-team for
   the armed trigger and the count-to-two rule). Do not paper over a
   second real miss with a bigger heuristic; that's what the ratchet is
   for.
2. Every `file_line`-evidence finding in a **kept** adversary verdict
   (`work/<task-id>/artifacts/<agent>-verdict.json`, post-`verdict-filter`
   — never a dropped verdict, never the raw pre-filter file). Union
   `evidence.file`/`evidence.line` where `evidence.kind == "file_line"`,
   paired with the verdict's `claim` (first clause). A verdict whose
   `evidence.kind == "command"` contributes nothing here — there's no file
   to point at — **even at high severity**. That finding still appears in
   full under "How it was verified → Adversaries" (the floor above); it is
   never lost, only absent from this specific list because this list is
   about files, not findings. Compute `N` = the count of kept verdicts
   excluded from this list for exactly this reason, across every adversary
   that ran, and if `N > 0` the list's own last line reads: `"plus N
   finding(s) without file anchors — see verify.md"` — literal, mechanical,
   never omitted when true. This is what stops the list from silently
   reading as complete over a finding it structurally cannot represent.
3. Every path in the real diff (`git diff --name-only <base>..HEAD`) that
   matches a glob in `.spine/protected-paths.conf` (single-repo: this
   project's own file; multi-repo: each repo's own file, diff scoped to
   that repo) — same match semantics `path-escalate` and the plan-time
   protected-path check already use, just run against the actual diff
   instead of the predicted one.
4. Every entry in `work/<task-id>/artifacts/conformance.json`'s `actual`
   list that isn't in `predicted` — a real set difference read directly
   from the JSON, never verify.md's summary line (which carries counts
   only). `conformance` itself already excludes spine's own bookkeeping
   (`work/**`, `.spine/current-task`) from `actual` before this file is
   even written, and states its exclusion rule in the JSON's own
   `excluded_rules` field — so this union needs no display-layer filtering
   of its own; it reads clean data. A `[DECLARED]`/`[UNDECLARED]` result
   from a ship-time `claims-check --diff` collision is not a fifth source:
   `/ship` §0 already turns any `[UNDECLARED]` hit into a halt-tier
   `deviations.md` record before this step runs, so it surfaces through
   union (1) like any other deviation. **Guard, found for real against an
   old-template plan during this template's own demonstration**: if any
   `predicted` entry itself contains a backtick, `conformance`'s match
   against it silently failed for every entry (the pre-existing
   Markdown-formatted-path bug the readability patch's template fix
   guards going forward, but does not retroactively repair on a plan
   written before that fix) — a raw set difference against a corrupted
   `predicted` list would falsely count every genuinely-predicted file as
   drift too. Detecting a backtick in any `predicted` entry: treat this
   union as unavailable for this task and say so plainly ("plan accuracy
   not computable — predicted-touch entries are Markdown-formatted, see
   conformance.json"), never emit the misleading set difference.
   `conformance.json` simply not existing (the script never ran) is a
   separate, ordinary case — say "not run for this task," not "none."

If all four unions are empty (after the completeness line's own N is also
zero), the section renders the fixed line in its own slot below — never an
empty bullet list, never a fabricated pointer, and never silently omitted
either (an empty union is itself information: the plan's predictions held).

Multi-repo (Extension B) and the Contracts/Milestone-adjacent sections
below are explicitly optional — omit the whole section (not an empty one)
per each section's own rule. No script reads this file back in (same as
briefing.md); the **bold-label**
convention is kept anyway, for the same reason briefing.md keeps it: cheap
now, `grep`-able the day `/costs` or some other aggregate reads PRs in
bulk instead of one at a time.
-->

## `<task-id>` — <title in plain words>

**What & why:** <One to three sentences — what is now true that wasn't,
and why it was worth doing. Source: plan.md's `## The gist` and the
briefing's own What & why. Least-context reader: the person reviewing this
PR was not in the task.>

**Review this at the plan level:** <plan.md's `## The gist` (the approach)
and `## What could go wrong` (the risk), condensed per the writing mandate
(`core/templates/writing-mandate.md`) — the decisions and architecture
being ratified here, not the diff's mechanics. Source: plan.md only. This
is the review surface: it was approved before implementation and verified
after — the next section says how, the section after that says where the
verification actually looked.>

**How it was verified:**
- Floor: <one line — pass, or what failed and how it resolved. Source:
  verify.md's floor result / Report.>
- Adversaries: <count + max severity + one-line gist each, pointer:
  "details: verify.md". FLOOR RULE — never compressed below this, see this
  file's own header note.>
- Plan accuracy: <conformance in words, from `conformance.json`: "diff
  landed exactly where the plan said" (actual ⊆ predicted, nothing extra)
  or "drifted: <where>, see Where to look below." Source:
  work/<task-id>/artifacts/conformance.json — the same file union (4)
  reads, so this bullet and that section can never silently disagree about
  what drifted.>
- Gaps: <capability gaps that degraded verification, from verify.md's own
  "Capability gaps" section, quoted not re-derived, or "none.">

**Where to look (in priority order):** <The union of the four sources in
this template's own header comment, each with a half-line of why it's
listed. Ends with the completeness line (rule 2 above) whenever `N > 0`.
If every union (and N) is empty: "The diff landed exactly where the
approved plan predicted; no deviations, findings, or drift. Spot-check at
will." — which is itself information, not an excuse to skip review.>

**What surprised us:** <Deviations in human terms, from deviations.md — one
separate markdown bullet per record, never a single paragraph with
parenthetical numbers even when terse: what the plan assumed, what was
actually true, how it was resolved. "Nothing — the plan held." if none.>

<!-- Omit "Overrides & bypasses" entirely when none occurred — never leave
     it present-but-empty. FLOOR RULE: never folded into a single
     summarizing line, never omitted when real. -->

**Overrides & bypasses:** <Any `--bypass`, claims-check `--diff`
`[UNDECLARED]` override, or second-approver self-approval override — each
its own line, the recorded reason included. Source: the ledger +
approval.json, same fields briefing.md's own "Overrides & bypasses"
section reads.>

<!-- Omit "Milestone" entirely when this task isn't part of one. Present
     for exactly the same reason "Adversaries" is a floor rule above: a
     `checkpointed`/`auto` task's human touchpoint is this PR, not
     briefing.md, so a flagged finding drafted for milestone.md but not
     yet applied (/ship §3a) must be visible here too, not only in a file
     that autonomy's reviewer may never open. -->

**Milestone:** <Which milestone, and (only on the completing ship)
whether its Done-definition is met by real state, said plainly either
way. This task's own flagged-finding triage result (/ship §3a): which
findings (if any) were carried into `milestone.md`'s Known gaps, with
their new `gap-<n>` ids (`guided`), or the drafted "Proposed milestone gap
entries — undecided" list awaiting this PR's own reviewer to triage
(`checkpointed`/`auto`) — never omitted just because §3a found nothing to
carry. Also this task's own known-gap resolution result (/ship §3c):
which `gap-<n>` entries (if any) this task's own plan cited and closed out
of `milestone.md`, "none cited" otherwise. Source: same fields
briefing.md's own "Milestone" section reads.>

<!-- Multi-repo only (Extension B) — omit entirely on a single-repo task. -->

**Contracts:** <Contracts touched, their `spec_change` classification,
blast-radius consumer repos, and this PR's own position in `## Ship
order` ("2 of 3 — API ships after schema, before frontend"). Source:
plan.md's `## Ship order` + work/<task-id>/artifacts/contract-touch.json.
One shared description per task (not one per repo) — the plan and
verification this describes are task-scoped, not repo-scoped; each
member repo's own PR links here rather than carrying a divergent copy.>

Record: `work/<task-id>/` — plan, verify.md, deviations.md, briefing,
ledger.
