# Visual fidelity and content provenance

Design rationale moved out of `docs/tradeoffs.md`. The original proposal is `docs/design/ui-fidelity-proposal.md`.


Design rationale: `docs/design/ui-fidelity-proposal.md`. Two real misses drove it: a
screen that passed `ui-conformance` while looking nothing like its mockup
(wrong layout, wrong button variant, a component rendering its own props run
together, an unused badge prop), and invented copy/numbers in fixtures that
no step could see.

- **A scripted capture plus a reviewer, not a scripted comparison.** §4's
  self-test needs a guaranteed-pass and guaranteed-fail fixture; "does this
  match the mockup" has neither without pixel-diffing, which §3.3/§3.9
  reject. `ui-capture` is deterministic and self-testable (it renders each
  state and records measurable facts); the judgment is a fresh-context
  `ui-fidelity` agent per screen.
- **Findings must cite a captured fact (`render` evidence).** `verdict-filter`
  checks the quoted span exists in the capture, the same bar `decision`
  evidence gets. Recall is deliberately capped: a visual difference no
  captured measurement expresses is dropped. This trades misses for trust;
  it is unmeasured until the calibration run in the proposal's acceptance
  section is done, and should be treated as unproven until then.
- **Not a floor gate, but never unanswered.** Findings don't change
  `/verify` PASS/FAIL (a model's judgment shouldn't fail the floor); a
  failed capture does. `/ship` §3a requires a human disposition (fix now,
  carry, decline-with-reason) for every kept unfixed finding at every
  autonomy level — a deliberate exception to `checkpointed`/`auto`'s
  no-scheduled-stop rule, since this is the only check that notices a
  wrong-looking screen.
- **Gallery routes or step files, adapter's choice.** A project that builds a
  dev-only state gallery (`/__ui/<screen>?state=<state>`, from typed test
  data) uses it and skips step files; `default` is still captured from the
  real signed-in route. A gallery proves the view can look right in a state,
  not that the app reaches it — disclosed in the contract. The capture
  adapter consumes the project's signed-in context and gallery rather than
  building its own.
- **Every declared state is compared, or reported not compared with a
  reason** (`no_screenshot`, `no_driver`, `driver_failed`). Reaching a state
  needs hand-authored `docs/ui/states/<id>.json` steps, which go stale when
  labels change (surfaced as `driver_failed`, never a pass). Chosen over a
  `?state=` hook because it needs no product code.
- **Content provenance is a plan-time gate.** `## Content sources`
  (MACHINE fence, `content-sources-check`) requires every block of copy or
  numbers to cite a real file, a human-supplied value, or `none` (a hard
  stop-and-ask, at every class and autonomy). Cited sources must be in
  `research.md`'s `files:`, which is what puts the spec and screenshots
  under `check-stale`. The fidelity reviewer also flags render text found in
  neither the spec nor the screenshot — screenshots aren't machine-readable,
  so that part is the vision reviewer's, not a script's.
- **`.spine/ui-content-paths.conf`** makes content-only diffs (fixtures,
  seed data, copy) count as UI touches in `ui-touch`, which previously fired
  only on view-file globs.
- **Rejected:** a falsifier mandate for untraceable strings (duplicates the
  plan gate plus the reviewer); a write hook keyed on `*.fixtures.*` (a
  filename convention is per-project and core is stack-blind); a CLAUDE.md
  "never invent content" rule as the only defense (advisory).
- **Measured (turnpilot, the only project with a `ui-capture` adapter).**
  Calibration on 2026-09-23 with seeded defects: 5 of 6 caught (the sixth went
  unreviewed: one agent per screen ran out of budget at 3 of 12 states), both
  decoys correctly ignored. On an unmodified control it filed 16 findings, 6 real
  and 10 noise (about 62% false positives), nearly all from gallery states
  rendering test data. In use since: 19 tasks, 254 findings with `render` evidence,
  of which 23 were fixed; 47 high-severity findings were left unfixed. The agent
  now exempts gallery states from provenance findings and spells out its reply
  shape; neither change has been re-calibrated yet. Re-run `.spine/calibration/`
  in turnpilot to confirm. The reviewer's own severities are unreliable (it rated
  a run-together headline low and unchanged test data high).

