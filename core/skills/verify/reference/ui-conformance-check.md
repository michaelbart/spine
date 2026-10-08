# Verify — UI conformance check (loaded on demand)

Loaded from SKILL.md when: step 1c found `ui_touched` is `true`.

Reuses the exact `ui-touch` result step 1c already produced this pass —
never a second invocation. If `ui_touched` was `false`:
skip this step entirely, same omit-whole-section rule step 1c
itself uses.

If `ui_touched` was `true`: check class eligibility, same
opt-in shape step 1c uses, its own separate key:

```
jq -r '.ui_conformance_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: eligible only if the above reads
`true` (default `false`). Not eligible: record `SKIPPED (Class 1,
ui_conformance_class1_optin not set)` in verify.md's own "UI
conformance" section — a deliberate calibration choice, not an
unexplained absence.

Eligible: check `.spine/capabilities.json` for `ui-conformance`'s
status. If `implemented`, run it directly — **not through
`floor`** — `.spine/adapters/ui-conformance` (CWD at the
project root, per `core/ADAPTER-CONTRACT.md §3.9`). Record pass/fail in its own
"UI conformance" section (§5) — never folded into "UI render," even
though both gate on the same `ui-touch` result; they check different
things (a real render vs. declared-token/component fidelity) and each
gets its own line so a reader can tell which one failed. If not
`implemented` (`unavailable`/`not-applicable` — the common case for a
project with no `docs/ui/` bundle at all), record it degraded with
its recorded reason.
