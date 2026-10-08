# Ship — milestone flagged-finding triage (loaded on demand)

Loaded from SKILL.md when: work/<task-id>/milestone exists.

The routing gap this step closes: an adversary-confirmed, cross-task-relevant finding that gets
deliberately flagged rather than fixed in this task's own `/verify` pass
has, until now, had no path into the one place a future member task's
planning actually looks (`## Known gaps` in
`milestone.md`, loaded by `core/skills/task/SKILL.md`'s `--milestone`
handling) — it stayed fully documented in this task's own `verify.md`/
`notes.md` and fully invisible to whoever plans the task that needs it.

**Gather.** Read every `work/<task-id>/artifacts/<agent>-verdict.json` this
task's `/verify` produced (falsifier always, security per §2's adversary
count) **and `ui-fidelity-verdict.json` when step 1e produced one**. Collect every kept verdict whose `disposition`
(`core/ADAPTER-CONTRACT.md §5`) is `"not_fixed"` or absent — absent is
never treated as resolved, same discipline as everywhere else in this
system a gate could otherwise silently read as passing.

**Dedup against what's already tracked.** Before asking anything, read
`work/<id>/milestone.md`'s `## Known gaps for future member tasks` section
(absent or empty on this milestone's first ship — nothing to dedup
against yet). **Before treating any existing entry as well-formed, verify
its shape:** each entry must be a fenced block with an `id: gap-<n>` line
matching `core/templates/milestone.md`'s own format, and the file must
have a `<!-- next-gap-id: N -->` counter. If any entry is plain prose
(no fence, no `id:` line) or the counter is missing, **flag it to the
human before continuing** — report each malformed entry by quoting its
text, explain that it can't be machine-cited by future tasks, and ask
whether to reformat it into the proper `gap-<n>` shape now (content
unchanged, prose → fenced block) or leave it as-is and note it in
`notes.md` as non-machine-citable. Never silently absorb malformed entries
as if they were well-formed — the carry-forward mechanism degrades
invisibly if you do. For each well-formed candidate, check whether its
`claim`/`evidence.file` substantially matches an existing `gap-<n>`
entry's `source` field (a cheap containment check against the
machine-fenced block, not semantic matching — same fidelity `check-stale`'s
own file-drift comparison already uses). A match: don't include it in the
question below; instead record in `notes.md`, "already tracked as
`gap-<n>`, not re-asked" — a suppressed question is still a decision, and
must read differently from a finding nobody ever looked at. This is what
keeps two sibling member tasks that independently trip the same underlying
gap from re-triaging it twice.

**Zero candidates remain** (nothing was flagged, or everything flagged is
already tracked): nothing further to do, no section in the briefing (§4)
— this is a gate that correctly never applied, not a degraded one.

**`ui-fidelity` findings always need a human disposition, at every
autonomy.** Unlike the branch below, a kept `not_fixed` (or absent
`disposition`) `ui-fidelity` finding — including a `driver_failed` state —
is never deferred to post-hoc PR review and never dropped as not
"cross-task-relevant": stop and show each one (severity, state, claim,
evidence) and require one of three answers — *fix now* (return to
implementation, then re-run `/verify`), *carry* into `milestone.md` Known
gaps (same `gap-<n>` mechanics as below), or *decline* with a stated reason
recorded in `notes.md`. The visual-fidelity check is the only place a
screen that looks wrong gets noticed; a finding nobody answered is exactly
the miss it exists to prevent. `auto` tasks stop here too —
this is a deliberate exception to their no-scheduled-stop rule. Ask it with
`AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** what to do about a visual mismatch the screenshot review found. It's yours because only you know whether it matters to the design.
> **Need to know:** <the finding in one plain sentence>. Screenshot: `<path>`.
> **Recommend:** <Fix now | Carry it | Decline> — <one-line reason>
> 1. **Fix now** — next: I go back and fix it, then re-run the checks; cost: about <n> minutes; undo: yes
> 2. **Carry it** — next: it merges as is and is logged as a known issue for a later task to see; cost: the mismatch ships for now; undo: yes, fix later
> 3. **Decline it** — next: it merges as is and I record your reason; cost: nobody tracks it further; undo: no, it is dropped
> **Safe to ignore:** the technical evidence; the screenshot shows it.
<!-- touchpoint:end -->

**One or more other candidates remain — branch on `work/<task-id>/autonomy`**
(absent = `guided`, same convention §5's PR-opening step already uses):

- **`guided`** — ask now, interactively, before proceeding to §3b/§4: for
  each candidate, show severity + claim + evidence pointer (file:line or
  command), and ask which should carry into `milestone.md`'s Known gaps
  for future member tasks to see — "none" is a complete, valid answer, not
  a thing to talk the human out of. This blocks the same way plan approval
  already blocks; it is not a merge
  gate (§1's two checks are unchanged, adversary findings still never fail
  `/verify` by that skill's own report step), just a question that has to
  be asked before this task's ship completes. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

  <!-- touchpoint:start -->
  > **Deciding:** which of the problems found during checking should be written down for future tasks to see. It's yours because it decides what the next person here is warned about.
  > **Need to know:** <n> findings survived review: <for each: how serious, one plain sentence, where to look>. None of them blocked this task.
  > **Recommend:** <carry these | carry none> — <one-line reason>
  > 1. **Carry the ones you pick** — next: I add each to the milestone's list of known issues; cost: one line each; undo: yes, delete the line
  > 2. **Carry none** — next: I move on; cost: nothing is recorded, so the next task won't know; undo: yes, until this ship finishes
  > **Safe to ignore:** the ones you don't pick; they stay in `verify.md`.
  <!-- touchpoint:end -->
- **`auto`** — no scheduled stop exists here, so don't
  manufacture one. Draft the candidate entries (same shape the "apply the
  human's picks" step below produces for `guided`) into a new "Proposed
  milestone gap entries — undecided" section of the briefing (§4) and the
  PR description (§4a),
  explicitly not yet applied to `milestone.md`. The human's post-hoc PR
  review — the same relocated touchpoint these autonomies already use for
  plan review — is where these get triaged, by hand-editing `milestone.md`
  or leaving them. Never write to the always-loaded `milestone.md` without
  a human having actually looked, whether that look happens now (`guided`)
  or at PR review.

**For `guided`, apply the human's picks now.** For each carried finding:
allocate the next id from `milestone.md`'s own `next-gap-id` counter
(`core/templates/milestone.md`'s comment — a monotonic counter, never
"highest id currently present," so a gap-<n> §3c already removed this
milestone's history is never reused for something unrelated), then
increment that counter in the file. Append a new fenced entry per
`core/templates/milestone.md`'s own comment — `source` = this task's
`verify.md` path plus the agent/severity, prose drafted from the verdict's
own `claim`/`evidence` plus `milestone.md`'s `## Member tasks` list (never
copied verbatim from `verify.md`'s adversary-voice prose, which is written
for an attacker's audience, not a future planner's). For each declined
finding: record the decision and its stated reason in `notes.md` — a
finding the human looked at and declined must read differently,
permanently, from one nobody ever asked about.
