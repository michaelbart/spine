# Verify — UI sections of verify.md (loaded on demand)

Loaded from SKILL.md when: step 1c found `ui_touched` is `true` and you are assembling verify.md.

- **UI render** (per `core/templates/verify.md`'s own section, omitted
  entirely if step 1c found no UI path touched): the
  `ui-render` result — pass/fail with the adapter's own diagnostics on
  fail, `SKIPPED (Class 1, ui_render_class1_optin not set)` if this class
  wasn't eligible, or degraded with its recorded reason if the capability
  isn't `implemented`.
- **UI conformance** (per `core/templates/verify.md`'s own section,
  omitted entirely under the same condition as "UI render" above — they
  share one `ui-touch` result): the `ui-conformance` result — pass/fail
  with the adapter's own diagnostics on fail (which declared component or
  token didn't show up in the real render), `SKIPPED (Class 1,
  ui_conformance_class1_optin not set)` if this class wasn't eligible,
  or degraded with its recorded reason if the capability isn't
  `implemented` (the common case for a project with no `docs/ui/`
  bundle).
- **UI fidelity** (per `core/templates/verify.md`'s own section, omitted
  entirely under the same condition as "UI render"): the capture result,
  then per screen the states compared vs not compared (with
  `coverage.json`'s reason), then each kept `ui-fidelity` verdict —
  severity, state, claim, and its `render` evidence — and kept/dropped
  counts from `verdict-filter`. Or `SKIPPED (Class 1, ui_fidelity_class1_optin
  not set)`, or degraded with `ui-capture`'s recorded reason. Write the same
  `disposition` (`fixed`/`not_fixed`) back into `ui-fidelity-verdict.json` as
  for the adversaries. `ui-fidelity` is not an adversary for §2's
  `class1_adversaries` count.
