<!--
Written by /ship. ≤1 page, hard — the human reads this once, at merge.
This is the mental-alignment artifact (build prompt §1, failure mode 5,
named first by the engineer): if reading this doesn't leave the engineer
knowing what their product now does and why, it failed at its one job.
Don't pad it to look thorough; cut anything without a reason to be here.

Full detail already lives in the task's own record — `plan.md`,
`verify.md`, `deviations.md`, the ledger, all pointed at by this file's
own closing line. This document's rule is "nothing omitted, only
ordered": everything material appears, headline-first, with a pointer
into the record for depth — simplifying a section into silence is a
defect here, same as burying its point three sentences down.

**No script or skill reads a previously-written briefing.md back in**
(this file is pure human output). The
**bold-label** convention below (`**Floor:**`, `**Overrides & bypasses:**`,
etc.) is kept anyway, and kept stable across tasks, on purpose: it costs
nothing today and it's what makes this file `grep`-able the day something
(`/costs`, most likely) starts reading briefings in aggregate instead of
one at a time. Don't rename a label casually.

**The two floors — never compressed below these, regardless of how tight
the one-page budget gets:**
- Adversary findings: count + max severity + one-line gist each, plus a
  pointer into `verify.md`. If the page is overflowing, cut optional prose
  elsewhere first — never this line.
- Overrides/bypasses: never folded into a single line that could bury one,
  never omitted when real, never summarized into "some overrides occurred."

Multi-repo (Extension B) and milestone sections below are each explicitly
optional — omit the whole section (not an empty one) when this task is
single-repo, touches no declared contract, or isn't part of a milestone.
-->

# Shipped: `<task-id>` — <title in plain words>

**What & why:** <One or two sentences — what is now true that wasn't, and
the reason it was worth doing. Full step-by-step detail already lives in
`plan.md`; this is the headline, not the changelog. Written for the
engineer who wasn't in this task.>

**What surprised us:** <Deviations in human terms, one line per
`deviations.md` record — what the plan assumed, what was actually true,
how it was resolved. "Nothing — the plan held." if none. Never manufactured,
never hidden.>

**Verification, honestly:**
- Floor: <pass, or what failed and how it resolved>
- Re-grounding: <ship-time re-check result — "check-stale: ok, floor
  re-run: pass" is the unremarkable case, still shown verbatim>
- Adversaries: <count + max severity + one-line gist each, with pointer:
  "details: verify.md §n". Never paraphrased below count + severity +
  pointer — see this file's own header note>
- Capability gaps: <capabilities that degraded verification this task,
  named plainly with their recorded reason, or "none">
- Tooling gaps: <quoted verbatim from verify.md's own Tooling gaps section
  — never re-derived — or "none">
- Plan accuracy: <conformance in words: "diff landed where the plan said"
  or "drifted: <where> — see verify.md">
- Approver (Class 2 only): <second-approver identity — omit this bullet
  entirely for Class 0/1, or when an override made it into "Overrides &
  bypasses" below instead>

<!-- Omit the whole "Contracts" section for a single-repo task, or a
     multi-repo task whose contract-touch run found nothing touched. -->

**Contracts:** <Per touched contract: name, spec_change classification,
which consumer repos entered blast radius, any [UNDECLARED] coupling the
falsifier's cross-repo mandate found. A touched contract with zero findings
is still worth its one line — a clean bill is not the same as an omitted
section.>

<!-- Omit "Overrides & bypasses" entirely when none occurred this task —
     never leave it present-but-empty. -->

**Overrides & bypasses:** <Any `--bypass`, claims-check `--diff`
[UNDECLARED] override, or second-approver self-approval override — each
its own line, the recorded reason included, loud. Never folded into a
single summarizing line.>

<!-- Omit "Milestone" entirely when this task isn't part of one. -->

**Milestone:** <If this isn't the milestone's completing ship: one line
noting which milestone and that member tasks remain. If it is: which
milestone, and whether its Done-definition is actually met by real state
right now — say so plainly if it isn't; a milestone reported done that
silently isn't is exactly the failure mode this line exists to make loud.
Either way, also this task's own flagged-finding triage result (/ship
§3a): which findings (if any) were carried into `milestone.md`'s Known
gaps, with their new `gap-<n>` ids (`guided`), or the drafted "Proposed
milestone gap entries — undecided" list awaiting the human's PR-time
triage (`checkpointed`/`auto`) — never omitted just because §3a found
nothing to carry; "zero flagged findings" and "N findings, none carried"
are different facts. And this task's own known-gap resolution result
(/ship §3c): which `gap-<n>` entries (if any) this task's own plan cited
and closed out of `milestone.md`, "none cited" otherwise.>

**In six months you'll want to know:** <One sentence. The non-obvious
thing — the constraint honored, the trap avoided, the decision that looks
arbitrary but isn't. Often the sharpest line in the document; don't let it
default to a restatement of "what changed.">

Record: `work/<task-id>/` — plan, verify.md, deviations.md, ledger.
