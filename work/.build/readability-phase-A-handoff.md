# Readability patch — Phase A: consumer inventory

Grepped every file under `core/` for `plan.md`, `briefing`, and each
section marker the build prompt named (`Predicted touch`, `Grounds on
decisions`, `Latitude table`, `Ship order`, `Contract change`, `##
Approach`, `task-id`, `milestone`, `grounding`, `Decide-alone`,
`Record-and-proceed`), then read every hit in full. Two findings up front
that change the shape of Phase B:

1. **`briefing.md` has zero machine consumers.** Every hit for
   "briefing" outside `core/templates/briefing.md` itself and
   `core/skills/ship/SKILL.md` (which *writes* it) is prose — a comment
   explaining *why* something will appear in the briefing, never a script
   or skill instruction that reads it back in. Nothing parses this file.
   Phase B's briefing template is unconstrained by any consumer format —
   the only real cap is the human "≤1 page" rule and the two floors
   (adversary findings, overrides) the build prompt already specifies.

2. **Only one section is parsed by an actual bash/awk script**:
   `## Predicted touch`, by `core/scripts/conformance`. Every other
   "machine-parsed" section (`## Grounds on decisions`, `## Ship order`,
   `## Contract change`, the Latitude table) is read by an LLM executing
   a skill's prose instructions (`/task`, `/ship`, `/verify`), not by a
   compiled parser. That distinction matters for how much formatting
   latitude each section actually has — see the reconciliation table.

## Consumer table

| Consumer | Reads | Exact field/marker required | Parser type |
|---|---|---|---|
| `core/scripts/conformance` | `plan.md` | `awk` program: a line matching `^## Predicted touch` opens capture; each subsequent `^- ` line has `- ` stripped and everything from the first ` — ` onward stripped; capture closes at the next `^## ` line (or EOF). Heading text and single-dash-space bullet marker are load-bearing byte-for-byte. | **Real script (awk)** |
| `core/skills/task/SKILL.md` §3 (claims assembly) | `plan.md` | `predicted_touch` copied verbatim from `## Predicted touch` bullets into `claims.json`; `grounding_decisions` additionally pulls from `## Grounds on decisions`; a contract name from `## Contract change` when present. Same heading text and bullet grammar as conformance, but enforced by the LLM following SKILL.md prose, not a parser. | LLM/skill prose |
| `core/skills/task/SKILL.md` §3 (class escalation) | `plan.md` | Every `## Predicted touch` entry checked against `.spine/protected-paths.conf`; on a match, `work/<task-id>/class` is rewritten to `2`. Same bullet-list requirement as above. **The `path-escalate` hook itself never reads `plan.md`** — it only ever reads the already-written `class` file at edit time. The escalation *decision* is 100% LLM/skill-side. | LLM/skill prose |
| `core/skills/ship/SKILL.md` §1 (ship order gate, multi-repo) | `plan.md` | `## Ship order` — ordered list, one repo name per line (or the reserved name `workspace`), validated against `workspace.json`'s contract registry direction. Read by the LLM, not a script. | LLM/skill prose |
| `core/skills/ship/SKILL.md` §2 (decision lifecycle) | `plan.md` | `## Grounds on decisions` — heading text exact, one `- D-<seq>` (or `- <repo>:D-<seq>`) bullet per cited decision, optional trailing ` — description`. Section omitted entirely (not empty) when the plan cites nothing. | LLM/skill prose |
| `core/skills/ship/SKILL.md` §5 (staged multi-repo commit) | `plan.md` | Iterates `## Ship order`'s repo list in the order given. | LLM/skill prose |
| `core/skills/verify/SKILL.md` §1b (contract breaking-change gate) | `plan.md` | `## Contract change` — one line, exactly one of `expand \| migrate \| contract \| additive`. A `breaking` diff classification from `contract-touch` fails verify outright unless this line says `expand` or `contract`, regardless of the plan's prose elsewhere. | LLM/skill prose |
| `core/rules/contracts.md` | `plan.md` (described, not read by this file) | Prose description of the `## Contract change` gate above — not a separate consumer, just documentation of the same mechanism. | Prose only |
| `core/skills/task/SKILL.md` §4 (implement, latitude table) | `plan.md` | The Latitude table's three fixed tiers — **Decide-alone / Record-and-proceed / Halt** — are the vocabulary the implementing LLM matches a real decision against. No script greps the table; the three tier *names* are the load-bearing part, not the table's markdown shape. Separately, `core/templates/deviations.md`'s `- Tier:` field uses its own lowercase-hyphenated vocabulary (`decide-alone \| record-and-proceed \| halt`) — explicitly "machine-read by nothing today," and already a different casing/spelling convention from the plan's table. The two are linked only by an LLM's judgment, never a shared string. | LLM/skill prose, loosely coupled |
| `core/agents/falsifier.md` | `plan.md` | Given the plan path as one of three grounding inputs (plan, diff, caller map) — read as prose for context, never field-parsed. Its cross-repo mandate (d) references `## Predicted touch`'s repo-qualification convention (`<repo>:<path>`) only to describe its own evidence format, not to parse the plan. | Prose context only |
| `core/scripts/check-stale` | **`research.md` only** | Nothing in `plan.md`. The build prompt asked this to be checked as a "grounding citations" consumer — confirmed it never opens `plan.md`; its `spine:research` header lives entirely in `research.md`. Plan's own "grounding: research SHA" header line (new in Phase B) is therefore pure documentation, free-form. | N/A — not a plan.md consumer |
| `core/scripts/claims-check` (plan-time and `--diff`) | `work/<task-id>/claims.json`, `work/*/claims.json`, `work/*/state`, `work/*/owner` | Nothing in `plan.md` or `briefing.md` directly — it reads the already-assembled `claims.json`, which is *derived from* the plan by the skill layer above. | N/A — not a direct plan.md/briefing.md consumer |
| `core/scripts/second-approver-check` | `work/<task-id>/owner`, `work/<task-id>/approval.json` | Nothing in `plan.md` or `briefing.md`. Gates on `approval.json`, which is written by `/task` §3 *after* plan approval, not parsed from the plan's own text. | N/A |
| `core/scripts/contract-touch` | `workspace.json`, real git diff | Nothing in `plan.md`. (The current plan.md template's own header comment claims contract-touch "splits on the first `:`" the same way conformance does — **checked against the actual script: false**, contract-touch resolves repos from `workspace.json`'s `repos[]` list, never from parsing a plan path. Pre-existing template documentation drift, corrected in Phase B while the header comment is already being rewritten.) | N/A — corrected doc drift |
| `core/hooks/path-escalate` | `.spine/protected-paths.conf`, `.spine/current-task`, `work/<task-id>/class` | Nothing in `plan.md`. Confirms the class-escalation *mechanism* is two-part: the skill computes and writes the class (reading the plan), the hook only ever reads the already-written class file. | N/A |
| *(nothing)* | `briefing.md` | No script or skill reads a previously-written `briefing.md` back in. Confirmed by exhaustive grep — every "briefing" hit outside its own template and the `/ship` step that writes it is prose. | N/A — no consumer exists |

## Reconciliation plan for Phase B

Given the table above, here is which parts of the new templates must stay
byte-compatible, and which are free to restructure:

**Hard constraints (one real parser, `core/scripts/conformance`):**
- `## Predicted touch` must appear verbatim as a markdown `##` heading
  somewhere in `plan.md`, immediately followed by `- <path>` bullets
  (optional ` — <description>` trailing text), terminated by the next
  `## ` heading or EOF. The build prompt's own sketch shows this content
  wrapped only in `<!-- MACHINE: predicted-touch -->` / `<!-- /MACHINE
  -->` comments with no heading line inside — **that sketch is
  incompatible with conformance's awk as written and would break it.**
  Resolution: the fenced `MACHINE: predicted-touch` block will contain
  the literal `## Predicted touch` heading as its first line, e.g.:
  ```
  <!-- MACHINE: predicted-touch -->
  ## Predicted touch
  - path/to/file — why
  <!-- /MACHINE -->
  ```
  This is fully backward compatible — awk only cares about lines
  matching `^## Predicted touch` and `^- `, and ignores HTML-comment
  lines around them.

**Soft constraints (real skills, prose-enforced, but skills are edited by
this same patch so the constraint is "keep the contract skills document,
not keep the exact old wording"):**
- `## Grounds on decisions`, `## Ship order`, `## Contract change` keep
  their exact heading text and bullet/value grammar, because
  `core/skills/task/SKILL.md` and `core/skills/ship/SKILL.md` describe
  them by that literal heading text and grammar. These three sections
  will also live inside their own `<!-- MACHINE: ... -->` fences (not
  shown in the build prompt's sketch, which only fenced `header` and
  `predicted-touch` — extending the same convention to the other three
  machine-read sections is consistent with the stated principle and
  costs nothing).
- The Latitude table's three tiers get relabeled to the plain-language
  headers the build prompt specifies ("I'll just do:" / "I'll do and
  note:" / "I'll stop and ask before:"), **each still tagged with its
  fixed tier keyword** (`Decide-alone`, `Record-and-proceed`, `Halt`) so
  `core/skills/task/SKILL.md` §4's existing tier-matching prose keeps a
  literal string to match against — that step 4 prose will be
  lightly reworded in Phase B to reference the new headers directly
  rather than losing the linkage.

**Free (no consumer at all):**
- `briefing.md`'s entire structure — nothing reads it back in. The build
  prompt's template sketch is adopted as-is.
- `plan.md`'s new `<!-- MACHINE: header -->` block (task/class/owner/
  milestone/grounding) — none of `task`, `class`, `owner`, or `milestone`
  are ever read from `plan.md` by anything; they live in their own
  one-line files (`work/<task-id>/class`, `work/<task-id>/owner`,
  `work/<task-id>/milestone`) and `.spine/current-task`. This header is
  pure human-facing summary. The `grounding:` line's research SHA is
  copied from `research.md`'s own header for human cross-reference only.

**No consumer's format makes any template choice in the build prompt
impossible.** The one adaptation required (folding a real `##` heading
inside the `MACHINE: predicted-touch` fence rather than leaving the fence
as bare prose) is noted above and will be applied in Phase B without
further discussion needed.

## Full grep evidence (files touched, for reference)

```
core/agents/falsifier.md          — reads plan.md as context (prose)
core/agents/security.md           — no plan.md/briefing.md dependency found
core/rules/contracts.md           — documents the Contract change gate (prose)
core/scripts/conformance          — REAL PARSER: ## Predicted touch (awk)
core/scripts/check-stale          — research.md only, not plan.md
core/scripts/claims-check         — claims.json only, not plan.md/briefing.md
core/scripts/second-approver-check — owner/approval.json only
core/scripts/contract-touch       — workspace.json + git diff only
core/scripts/propagate            — claims.json + CLI args only
core/scripts/decision-hash        — docs/decisions/ only
core/scripts/ledger               — no plan.md/briefing.md dependency found
core/scripts/registry-sync        — no plan.md/briefing.md dependency found
core/scripts/design-gate          — no plan.md/briefing.md dependency found
core/hooks/path-escalate          — class file only, not plan.md
core/skills/task/SKILL.md         — writes plan.md; parses ## Predicted touch,
                                     ## Grounds on decisions, ## Contract change,
                                     Latitude table (LLM/skill prose)
core/skills/ship/SKILL.md         — reads ## Grounds on decisions, ## Ship order,
                                     ## Contract change; writes briefing.md
core/skills/verify/SKILL.md       — reads ## Contract change (breaking gate)
core/templates/claims.json        — documents the plan.md fields it's assembled from
core/templates/decision.md        — documents ## Grounds on decisions linkage
core/templates/verify.md          — documents "Verbatim from conformance against plan.md"
core/ADAPTER-CONTRACT.md          — prose only (briefing.md mentioned, never parsed)
docs/tradeoffs.md                 — prose only
```

No script or skill reference to `plan.md`/`briefing.md` was found outside
this list — `core/skills/tasks/SKILL.md`, `core/skills/workspace/SKILL.md`,
`core/skills/costs/SKILL.md`, `core/skills/ratchet/SKILL.md`,
`core/skills/remap/SKILL.md`, `core/skills/adopt/SKILL.md`,
`core/skills/bootstrap/SKILL.md`, `core/skills/security-checklist/SKILL.md`,
`core/skills/design/SKILL.md`, `core/agents/security.md`,
`core/agents/researcher.md`, `core/hooks/phase-gate`, `core/hooks/dep-gate`,
`core/scripts/floor`, `core/scripts/verdict-filter`,
`core/scripts/adapter-conformance`, `core/scripts/setup`, `core/scripts/q`
were all grepped and carry none.

## Demonstration source for Phase B

`~/horizon/work/20260808-fix-building-group-delete-orphans-units/` is a
real, already-shipped task with both `plan.md` (53 lines) and `briefing.md`
(92 lines) plus its full record (`research.md`, `verify.md`, `class`,
`artifacts/`) — this is the source Phase B will re-render from. It has no
`## Grounds on decisions`, `## Ship order`, or `## Contract change`
sections (single-repo, no decisions), so the demonstration will exercise
the header, gist, risks, latitude, steps, and predicted-touch sections
fully but not the multi-repo-only sections — that's a real gap in the
demonstration's coverage, noted here rather than silently glossed over in
Phase B's handoff.

---

**Stop for review.** Proceeding to Phase B (the template rewrite,
writing mandate, feedback rule, and the executed demonstration/regression)
requires confirmation of the reconciliation plan above.
