<!--
Assembled by /verify, not hand-written. Every section here has a mechanical
source — this file quotes scripts, it doesn't paraphrase them. A degraded
gate (a capability that's `unavailable`/`not-applicable`) must appear here
explicitly; a silently missing section is exactly the "silently skipped
gate" the build prompt calls the worst object this system can produce.
-->

# Verify: `<task-id>`

Class: `<1|2>` · Floor run: `<ISO timestamp>` · Result: `<PASS | FAIL>`

## Floor results

<!-- One line per capability, verbatim from `floor --out`'s JSON: PASS/FAIL
     with the adapter's one-line success message, or DEGRADED with the
     capabilities.json reason. Fail-fast means capabilities after the first
     failure are marked "not reached," not silently absent. -->

| Capability | Result | Detail |
|---|---|---|
| typecheck | | |

## Conformance

<!-- Verbatim from core/scripts/conformance against this task's plan.md.
     Informational only — never a merge gate, per build prompt §2.5 Layer 4.
     A low score here means research or planning is failing; track the
     trend across tasks via /costs, don't chase a single low score. -->

predicted=`<n>` actual=`<n>` precision=`<p>` recall=`<r>` f1=`<f>`

## Adversary verdicts

<!-- One subsection per adversary that ran (falsifier always; security per
     Layer 1 ceremony calibration / Class 2). Quote the filtered output of
     verdict-filter verbatim — never the raw pre-filter file. State the
     dropped count from verdict-filter's own stderr line; a verdict this
     section doesn't mention was either never raised or was dropped for a
     stated, mechanical reason, never silently. -->

### Falsifier

Attacked: <the `attacked` list, verbatim>

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
