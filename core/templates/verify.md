<!--
Assembled by /verify, not hand-written. Every section here has a mechanical
source — this file quotes scripts, it doesn't paraphrase them. A degraded
gate (a capability that's `unavailable`/`not-applicable`) must appear here
explicitly; a silently missing section is exactly the "silently skipped
gate" — the worst object this system can produce.

Multi-repo (Extension B): §"Floor results" repeats once per repo this task
*edited* — full floor, unmodified per-repo, per the edited-vs-affected
rule. §"Contract conformance" is new, and covers every
repo this task's diff put in *contract* blast radius without editing it
directly (core/scripts/contract-touch) — those repos never run their own
floor for this task, only contract-check, so a consumer's pre-existing
unrelated failures can never block a producer task forever (the "stranger's
mess" anti-pattern). A single-repo task has exactly one
"Floor results" table and an empty "Contract conformance" section — this
is the same zero-behavioral-change guarantee as everywhere else in
Extension B, expressed at the template level.

§"UI render" is unrelated to Extension B — it applies to a single-repo
task exactly as it does a multi-repo one, gated purely by whether the
task's own diff touched a declared UI path (core/scripts/ui-touch), never
by repo topology. Omit the whole section on any task (single- or
multi-repo) where nothing was touched.
-->

# Verify: `<task-id>`

Class: `<1|2>` · Floor run: `<ISO timestamp>` · Result: `<PASS | FAIL>`

## Floor results — `<repo-name, omit label for single-repo>`

<!-- One line per capability, verbatim from `floor --out`'s JSON: PASS/FAIL
     with the adapter's one-line success message, or DEGRADED with the
     capabilities.json reason. Fail-fast means capabilities after the first
     failure are marked "not reached," not silently absent. Repeat this
     whole section, once per repo, for a multi-repo task that edited more
     than one — never merge two repos' results into one table. -->

| Capability | Result | Detail |
|---|---|---|
| typecheck | | |

## Contract conformance

<!-- Multi-repo only, omitted entirely for single-repo. One line per
     contract core/scripts/contract-touch reported touched: the contract
     name, its spec_change classification (additive/breaking/unchanged),
     registry_stale flag, and — for every repo in that contract's
     consumers_in_blast_radius that this task did *not* edit directly —
     that repo's contract-check result (PASS/FAIL/DEGRADED, same
     discipline as a floor capability). "None" only if contract-touch
     reported zero touched contracts. -->

## UI render

<!-- Omitted entirely if core/scripts/ui-touch found no UI path touched
     (this project's own .spine/ui-paths.conf) in any repo — this is not a
     degraded gate, it's a gate that correctly never applied. When it did
     apply: the ui-render result — PASS/FAIL with the adapter's own
     diagnostics on fail (which route rendered blank/wrong, per
     core/ADAPTER-CONTRACT.md §3.3's pass criterion), `SKIPPED (Class 1,
     ui_render_class1_optin not set)` if this class wasn't eligible, or
     DEGRADED with capabilities.json's recorded reason if the capability
     isn't implemented. Same discipline as "Contract conformance" above —
     never folded into the floor table even though it's a floor-shaped
     pass/fail. -->

## Conformance

<!-- Verbatim from core/scripts/conformance against this task's plan.md.
     Informational only — never a merge gate (Layer 4).
     A low score here means research or planning is failing; track the
     trend across tasks via /costs, don't chase a single low score. -->

predicted=`<n>` actual=`<n>` precision=`<p>` recall=`<r>` f1=`<f>`

<!-- Multi-repo: one predicted/actual/precision/recall/f1 line per edited
     repo — conformance itself is unchanged (core/scripts/conformance still
     takes one --project), /verify just calls it once per repo with that
     repo's own predicted-touch subset (the repo-qualified prefix stripped)
     and its own git diff. -->

## Adversary verdicts

<!-- One subsection per adversary that ran or was reused (falsifier always;
     security per Layer 1 ceremony calibration / Class 2). Quote the
     filtered output of verdict-filter verbatim — never the raw pre-filter
     file. State the dropped count from verdict-filter's own stderr line;
     a verdict this section doesn't mention was either never raised or was
     dropped for a stated, mechanical reason, never silently.

     Adversary re-run caching (core/skills/verify/SKILL.md §3) has three
     outcomes, not two:

     - Skipped (REUSED): replace the Attacked/kept/dropped lines below
       with a single line, `REUSED (content and blast radius unchanged,
       prior clean-enough verdict from <ran_at>)` — every blast-radius
       file matched a prior pass's recorded content hash, and that prior
       verdict kept no medium/high finding (empty or low-only both
       qualify). Never means a narrowed or partial pass — the prior pass
       itself saw the full diff.
     - Focused re-run: keep the normal Attacked/kept/dropped lines and
       findings below, but prepend one line above Attacked: `Focused
       re-run — budget ~<used> of a normal ~<class-based figure>; <n> of
       <m> blast-radius files unchanged since <ran_at>: <short list or
       count>.` The adversary still received the full current diff and
       full authority; only its wall-clock budget was reduced.
     - Full re-run: normal Attacked/kept/dropped lines and findings,
       no extra line — this is the default, unremarked case. -->

### Falsifier

Attacked: <the `attacked` list, verbatim>

<!-- Multi-repo: falsifier's delegation additionally received the touched
     contracts + their consumer lists (core/agents/falsifier.md's
     "Cross-repo mandate") — its `attacked` list should show that hunt
     alongside (a)/(b)/(c) and the three questions; if it doesn't, that's
     itself worth a line here, not silent. -->

Verdicts kept: `<n>` · dropped: `<n>`

<!-- one entry per kept verdict: claim, severity, evidence -->

### Security

Attacked: <the `attacked` list, verbatim>

Verdicts kept: `<n>` · dropped: `<n>`

## Capability gaps

<!-- Every capability marked `unavailable`/`not-applicable` in
     .spine/capabilities.json that this class would otherwise have run,
     with its recorded reason. Empty only when every capability for this
     class is genuinely `implemented`. -->

## Tooling gaps

<!-- Distinct from capability gaps above: this is about spine's own core
     scripts (ledger, check-stale, conformance, verdict-filter, floor
     itself) being unreachable during this task, not a project capability
     being unimplemented. Merged from work/<task-id>/notes.md's
     accumulated `TOOLING GAP:` lines plus anything hit during this
     /verify run. One line per gap: script name, consequence. Write "none"
     only if genuinely empty. -->
