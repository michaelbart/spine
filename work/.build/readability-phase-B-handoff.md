# Readability patch — Phase B: templates, demonstration, regression

Builds on `readability-phase-A-handoff.md` (approved, four dispositions
folded in below). Delivers: `core/templates/plan.md`, `core/templates/
briefing.md`, `core/templates/writing-mandate.md`, updated `core/skills/
task/SKILL.md` and `core/skills/ship/SKILL.md`, a doc-drift fix in
`core/rules/contracts.md`, a README feedback-rule line, and — executed,
not narrated — a real re-render of a real task's plan+briefing, a real
live task run through the loop on the new templates, and every Phase A
consumer re-run against real output.

## What shipped

| File | Change |
|---|---|
| `core/templates/plan.md` | Rewritten: machine header, gist, risks, latitude (3 tagged lists), steps, fenced predicted-touch, "how we'll know it worked," then appendix-fenced grounds-on-decisions/ship-order/contract-change |
| `core/templates/briefing.md` | Rewritten: bold-label sections (What & why / What surprised us / Verification, honestly / Contracts / Overrides & bypasses / Milestone / In six months / Record) |
| `core/templates/writing-mandate.md` | New. ~15-line shared writing rule, referenced (not duplicated) by both skills |
| `core/skills/task/SKILL.md` | §3 references the mandate and the new fence; §4's tier-matching prose rewritten to cite the plan's new latitude labels + their `deviations.md`-matching keywords |
| `core/skills/ship/SKILL.md` | §4 rewritten to the new briefing structure, references the mandate |
| `core/rules/contracts.md` | Doc-drift fix: `## Contract change: expand` (never-real inline-colon syntax) → accurate description |
| `README.md` | New line in the day-one section: a confusing plan/briefing is a template defect, fix the template |

## Disposition 1 — consumer-wins fix + doc-drift, applied

`## Predicted touch` keeps its literal heading *inside* the `<!-- MACHINE:
predicted-touch -->` fence (not just prose around it) — verified against
the real `conformance` awk both on the bare template and on real,
regenerated content (commands below). The `contract-touch` colon-splitting
claim is corrected in `core/rules/contracts.md` and in the new `plan.md`'s
own header comment, which now states the true mechanism: `/verify` strips
the `<repo>:` prefix into a scratch file before calling `conformance`/
`contract-touch` per repo; neither script parses the prefix itself.

## Disposition 2 — soft-consumer discipline, and real (not static) proof

Every LLM-prose-read section (`## Grounds on decisions`, `## Ship order`,
`## Contract change`, the latitude table) kept its exact heading text and
bullet grammar, now inside its own `MACHINE`-labeled fence for visual
consistency with the one section a script actually parses — no parser
demands the fence, but a stable, distinctive marker is the only defense
an LLM reader (which fails silently and partially, never loudly) gets.

The latitude table's three tiers are now tagged with the literal keyword
`deviations.md`'s own `- Tier:` field uses (`decide-alone` /
`record-and-proceed` / `halt`) — this is tighter than before: pre-patch,
the plan's tier vocabulary (`Decide-alone`, capitalized, English) and
`deviations.md`'s (`decide-alone`, lowercase-hyphenated) were linked only
by an LLM's judgment call, explicitly flagged in Phase A as "loosely
coupled." Now they're the same string.

**Real proof, not a static template review** — a full task,
`20260811-extract-slugify-helper`, run by hand through every phase of
`/task`/`/ship`'s actual algorithm in `~/bgr` (spine's second real
install, clean floor, no pre-existing debt):

- Wrote a real `plan.md` on the new template (extracting `slugify()` out
  of `uniqueSlug` in `src/lib/slug.ts` — a genuine, small, real change).
- Implemented it for real, and **hit a genuine `record-and-proceed`
  moment**, not manufactured: whether an all-punctuation input falls back
  to `"user"` wasn't obvious from reading the regex chain, so I ran it for
  real (`node -e`) before writing the test. Logged to
  `~/bgr/work/20260811-extract-slugify-helper/deviations.md` with
  `- Tier: record-and-proceed` — the literal keyword the new plan's
  **I'll do and note** (`record-and-proceed`) list carries. This is the
  live link disposition 2 asked for: a real deviation, traced back to a
  real plan section, by a literal shared string, not a paraphrase.
- Ran the real floor: `PASS` on all 7 capabilities (`typecheck`, `lint`,
  `test` — 9/9 new unit tests — `secret-scan`, `dep-diff`, `clone-scan`,
  `callers`).
- Ran real `conformance`: `precision=1.00 recall=0.14 f1=0.25` (both
  predicted files genuinely touched; low recall is the task folder's own
  bookkeeping, not a planning miss — same shape as the horizon
  re-render's result below).
- **Hand-executed `/ship`'s new §4 against these real artifacts** —
  `~/bgr/work/20260811-extract-slugify-helper/briefing.md` is a real
  briefing on the new template, quoting only real `verify.md`/
  `deviations.md` content, correctly omitting Contracts/Overrides/
  Milestone/Approver (none apply) rather than leaving empty headers.
- **Team scenario**: created a second, synthetic open task claiming the
  same file, confirmed `claims-check` still correctly `BLOCKED` with the
  real conflict message (write/write + write/read), then deleted the
  synthetic task — proving the claims mechanism (unaffected by this
  patch, per Phase A) still composes correctly with a plan authored on
  the new template.

Commands, in order (all run for real, in this session, against
`~/bgr`):

```
core/scripts/check-stale work/20260811-extract-slugify-helper/research.md --project ~/bgr
  → ok: ... is current as of 7b9f7bc... (1 grounding files, 0 grounding decisions unchanged)
core/scripts/claims-check 20260811-extract-slugify-helper --project ~/bgr --no-pull
  → claims-check: clear — no write/write or write/read conflicts with any other open task
npx vitest run src/lib/slug.test.ts
  → 9 passed (9)
core/scripts/floor 1 --task 20260811-extract-slugify-helper --out .../floor-result.json
  → floor: PASS (7/7 capabilities)
core/scripts/conformance work/20260811-extract-slugify-helper/plan.md --project ~/bgr
  → precision=1.00 recall=0.14 f1=0.25
grep -c '^- Status: open' work/20260811-extract-slugify-helper/deviations.md
  → 0   (merge gate §1 check, real)
core/scripts/claims-check 20260811-extract-slugify-helper --project ~/bgr --no-pull   (with a synthetic conflicting task present)
  → claims-check: BLOCKED — 2 conflict(s): write/write ...; write/read ...
```

**Left deliberately uncommitted.** `~/bgr`'s working tree now has this
real change (`src/lib/slug.ts`, `src/lib/slug.test.ts`) plus the task
folder, state advanced to `ship-ready` but *not* `done` — per "only commit
when explicitly asked," I stopped short of `/ship`'s own `git commit`
step. Your call: commit it for real (it's a genuine, tested, real
improvement — extracting an untestable-before pure function), or `git
checkout`/`git clean` it back out if you'd rather this stayed purely
inside `~/spine`'s own demonstration. Nothing else in `~/bgr` was
touched.

## Disposition 3 — briefing freedom, spent with insurance

Every section in the new `briefing.md` template uses a stable
`**Bold label:**` — confirmed via Phase A that literally nothing parses
`briefing.md` today, so this costs nothing now and is aimed entirely at
`/costs`' eventual aggregation. Overrides & bypasses and the gap bullets
(`Capability gaps:`, `Tooling gaps:`) got the most attention on label
stability specifically because they're the two most likely first targets
if `/costs` grows a briefing-reading step.

## Disposition 4 — multi-repo sections: disclosed, not faked

Checked every reachable source for multi-repo/contract example data:
`docs/example/ext-c-two-engineer-demo/` (spine's own worked example) and
both installed projects (`~/horizon`, `~/bgr`). **None exist anywhere in
this environment.** The two-engineer demo's own `README.md` says so
explicitly in its own closing section: *"No multi-repo/contract scenario —
this demo project has no `workspace.json`."* Neither installed project has
one either (both are single-repo installs).

**Stated plainly: `## Ship order`, `## Contract change` (plan.md) and
`## Contracts`, `## Milestone` (briefing.md) shipped with zero real-data
demonstration in this patch.** What *is* true, and is the honest floor
under that gap: their heading text and bullet/value grammar are
byte-for-byte unchanged from the pre-patch template (verified by direct
diff against `plan-OLD.md`'s equivalent sections) — this patch introduced
no new risk to those sections even though it couldn't prove them against
real data. A maintainer adopting spine on a second workspace-mode project
is the natural point to close this gap for real; noted here rather than
manufactured now.

## Before / after — the real horizon plan and briefing

`work/.build/readability-demo/{plan,briefing}-{OLD,NEW}.md` — full files,
re-rendered from `~/horizon/work/20260808-fix-building-group-delete-orphans-units/`'s
real, already-written record (research.md, the original plan.md/
briefing.md, verify.md). Nothing invented; two honest gaps disclosed
inline in the NEW files themselves: `owner:` is unrecorded (this task
predates Extension C's owner file) and this task's real `/ship` run never
reached its briefing-writing step (floor failed at pre-existing lint debt
unrelated to the task) — the original briefing.md was itself hand-written
for the same reason, and the re-render keeps that fact in its first
visible line rather than pretending a clean ship.

Real consumer re-verification against this re-render (all commands run
this session, from `~/horizon`):

```
awk '/^## Predicted touch/{f=1;next} f && /^- /{...} f && /^## /{exit}' plan-NEW.md
  → 3 bare paths, exact conformance-awk extraction, confirmed against the live script (not reimplemented)
core/scripts/conformance plan-NEW.md --project ~/horizon
  → predicted=3 actual=21 precision=1.00 recall=0.14 f1=0.25   (real historical diff — still uncommitted in horizon's working tree)
core/scripts/check-stale work/.../research.md --project ~/horizon
  → STALE (3 items) — correctly independent of the plan.md rewrite; check-stale has never read plan.md (Phase A finding)
core/scripts/claims-check 20260808-... --project ~/horizon --no-pull
  → "claims.json not found" (exit 2) — correct: this task predates Extension C's registry, unrelated to this patch
core/scripts/second-approver-check 20260808-... --project ~/horizon
  → "owner not found" (exit 2) — same reason
```

**Line count, stated honestly**: the new plan is 117 lines vs. the old
53 (well under the 200 cap either way); the new briefing is 79 vs. the
old 92. The plan grew because content that was implicit or scattered
before (the rejected-alternative reasoning, what verification can't show)
is now explicit and headline-ordered, not because anything was padded —
every sentence in `plan-NEW.md` traces to a specific line in the original.
Line count was never the target; time-to-the-headline was — a reader who
stops after "The gist" and "What could go wrong" (9 lines) has the full
shape-and-risk picture that previously required reading past "Approach"
into "Rejected alternatives," 38 lines deep in the old file, to assemble.

## Self-red-team

**1. Boilerplate risk prose ("edge cases may exist").** Nothing
mechanically enforces this — no script can tell a real risk from a stock
phrase. Two defenses, both non-mechanical: the writing mandate's rule 5
("no stock phrases... name the real risk, or say 'little' and stop"), and
the README's new feedback rule as the human backstop when the mandate
alone doesn't catch a lazy fill-in. Not just asserted: the horizon
re-render's own "What could go wrong" section is real evidence this isn't
inert — it states two concrete, real risks (non-atomic ordering, zero
existing test coverage) rather than a generic hedge, and the bgr demo's
own risk section explicitly says "Little" *and* explains why, rather than
omitting the section.

**2. Gist drifting into marketing register.** Same limit, same two
defenses (mandate rule 2, README backstop) — nothing mechanical catches
"plain language" violations either. Both re-rendered gists in this
patch's own demonstration stay in flat, technical register (file names,
function names, real behavior) as evidence of the intended register, not
proof the rule is self-enforcing.

**3. Machine blocks drifting from prose.** Checked directly: `conformance`
does **not** catch this. It compares `## Predicted touch` against the real
diff — an axis orthogonal to whether the gist's prose names the same files
the fenced block lists. A plan whose gist says "three files" while
`## Predicted touch` lists five would conformance-score normally (against
the diff) while still being internally inconsistent. The writing
mandate's rule 6 addresses this at authoring time only. Per the build
prompt's own offered options: **recording this as a ratchet candidate**,
not building a new checker now — a prose/machine-block cross-check script
would be a new mechanism, out of scope for a "no new mechanisms" patch.
Flagged here for `/ratchet` to pick up if it recurs as real friction.

**4. One-page cap gamed by cramming.** The floors (adversary findings,
overrides) are structurally protected, not just declared: in the new
briefing template, they live as fixed-format bullets inside "Verification,
honestly" (`Adversaries:`) and their own un-omittable section ("Overrides
& bypasses") — both already terse by construction, nothing left to
compress. The only genuinely compressible content is the free prose
("What & why," "What surprised us," "In six months") — there is no other
place left to cut *to*, which is what makes cutting there the natural
failure direction rather than a hoped-for one. The template's own header
comment states this rule explicitly ("If the page is overflowing, cut
optional prose elsewhere first — never this line") so it's discoverable
by whoever's writing the briefing, not just true by section ordering.

## Pin-bump note for the maintainer

This is a template/skill-prose patch — no hooks, scripts, agents, or
capabilities changed. `adapter-conformance --all` is unaffected (nothing
here touches adapters). Before bumping `.spine/core-pin.json` in any
installed project: re-run this patch's own regeneration on that project's
own most recent real task if one exists (cheap — the same `awk`/
`conformance`/`claims-check` commands above), since Phase A's inventory
was exhaustive for *this* core checkout's current `core/` tree but a
project's own `.spine/adapters/*` are out of scope for this patch and
were not touched or re-verified.

**One real, disclosed migration edge, checked directly rather than
assumed away**: `core/skills/task/SKILL.md` §4's tier-matching prose now
names the plan's new heading verbatim (`## What I'll decide alone vs.
stop and ask`). An **in-flight task whose `plan.md` was already written on
the *old* template** (heading `## Latitude table`) does not have a section
by that literal name.

**Resolved**: in-flight tasks on the old template finish on the old
template — `conformance`'s `## Predicted touch` heading (the one section
any script actually parses) is byte-identical in both template shapes, so
nothing mechanical breaks either way; every task *opened* after the pin
bump uses the new template. No hand-migration of a live plan, no special
handling needed mid-flight.

---

**Stop. Deliver.**
