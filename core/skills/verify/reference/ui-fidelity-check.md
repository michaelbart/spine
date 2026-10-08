# Verify — UI fidelity check (loaded on demand)

Loaded from SKILL.md when: step 1c found `ui_touched` is `true`.

The one check that compares a real render against the handoff's screenshots.
`ui-conformance` (1d) proves declared components exist; this proves the
screen *looks like its reference*, state by state, and that no copy or
number appeared that the spec and screenshots don't define
(`core/ADAPTER-CONTRACT.md` §3.10). Reuses step 1c's `ui-touch` result,
never a second invocation; `ui_touched` `false`:
skip this step entirely, no section (same omit-whole-section rule).

Class eligibility, its own key:

```
jq -r '.ui_fidelity_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: only if `true` (default `false`). Not
eligible: record `SKIPPED (Class 1, ui_fidelity_class1_optin not set)` in
the "UI fidelity" section — a deliberate calibration choice, worded as one.

Eligible: check `.spine/capabilities.json` for `ui-capture`. Not
`implemented` (`unavailable`/`not-applicable`): record it degraded with its
recorded reason in the section and under Capability gaps — never silently
skipped. `implemented`:

1. Run `.spine/adapters/ui-capture` directly (**not through `floor`**; CWD at
   the repo root) with `SPINE_UI_CAPTURE_DIR=<abs>/work/<task-id>/artifacts/ui-fidelity`.
   Non-zero exit is a real failure of this step (route won't render, browser
   error): record the adapter's diagnostics in the section and stop here —
   never turn a failed capture into "no findings." (A state recorded in
   `coverage.json` as `driver_failed` is not an adapter failure; the reviewer
   reports it.)
2. Dispatch one fresh `ui-fidelity` subagent (Agent tool) **per screen** in
   the capture dir, in parallel, each given only the capture dir path, the
   screen id and the task id — no plan, no diff, no implementation narrative.
   Budget per agent roughly 1 minute per captured state, at least 5 and at
   most 20 minutes; on overrun cancel it and record the screen as not
   reviewed. Before dispatching, use `core/scripts/adversary-cache-tier
   <task-id> ui-fidelity` with the screen's changed view/content files **plus
   its spec JSON and every screenshot it maps** as the blast radius, so a
   replaced mockup invalidates a prior clean pass.
3. Merge the per-screen replies into one JSON object
   (`agent: "ui-fidelity"`, concatenated `attacked` and `verdicts`), write it
   to `work/<task-id>/artifacts/ui-fidelity-raw.json`, and filter it:
   `${CLAUDE_SKILL_DIR}/../../scripts/verdict-filter <raw> --out
   work/<task-id>/artifacts/ui-fidelity-verdict.json --project <project root>`.
   Findings whose `render` evidence doesn't resolve are dropped there.

Findings **never change `/verify`'s PASS/FAIL** (a model's judgment is not a
floor gate); a failed capture does fail this step's line. The teeth are at
`/ship` §3a, which requires a human disposition for every kept, unfixed
finding.
