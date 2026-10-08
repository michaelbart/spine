# Verify — UI render check (loaded on demand)

Loaded from SKILL.md when: step 1c found `ui_touched` is `true`.

If `ui_touched` is `false`: skip the rest of this
step entirely — no `ui-render` invocation, no section in `verify.md`
(this is not a degraded or skipped gate, it's a gate
that correctly never applied).

If `ui_touched` is `true`: check whether this class is
eligible to run `ui-render` at all, same gating shape `core/scripts/floor`
already uses for `mutate`:

```
jq -r '.ui_render_class1_optin // false' ~/.spine/user-config.json
```

Class 2: always eligible. Class 1: eligible only if the above reads
`true` (default `false` — same opt-in-only default `mutate_class1_optin`
uses). Not eligible: record this plainly in `verify.md`'s own "UI render"
section as `SKIPPED (Class 1, ui_render_class1_optin not set)` — this is
neither a pass nor a capability gap, it is a deliberate calibration choice,
and must read as one, not as an unexplained absence.

Eligible: check `.spine/capabilities.json` for `ui-render`'s status in
the project. If `implemented`, run it directly — **not through `floor`** —
`.spine/adapters/ui-render` (CWD at the project root, per
`core/ADAPTER-CONTRACT.md §3.3`). Record pass/fail in its own "UI render"
section (§5) — never folded into the floor table. If not `implemented` (`unavailable`/
`not-applicable`), record it degraded with its recorded reason — same
"never silently skip a gate" discipline as every other capability.
