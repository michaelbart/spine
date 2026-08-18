---
name: intake
description: The front door for ticketed work — fetch an issue-tracker ticket, size it against the real code, propose the right spine flow, and route into it. The first thing to run when you pick up a ticket.
disable-model-invocation: true
argument-hint: <ticket-key-or-url> | (or paste the ticket text)
---

You are running `/intake`. `$ARGUMENTS` is a ticket key/URL (e.g. `ABC-1234`)
or, if empty, a signal that the engineer will paste the ticket. Your job is to
turn a ticket into the *right* amount of spine — no ceremony an engineer has
to opt into, no rigor a real change should skip. You **classify and route**;
you never decide the class silently, and the class you propose is a revocable
hypothesis (research and `path-escalate` can still overturn it downstream).

Project root: the workspace root if `workspace.json` exists here, otherwise
this project. Scripts at `${CLAUDE_SKILL_DIR}/../../scripts/<name>`; adapters
at `<project root>/.spine/adapters/<name>`. `${CLAUDE_SKILL_DIR}` is a
placeholder you expand to this skill's own directory; hand the resulting
path — including the `../../` — to the shell verbatim. Do **not** lexically
collapse `skills/intake/../..` to `.claude/`: `.claude/skills/intake` is a
symlink into the spine core checkout, so the shell must resolve `../../`
against the symlink's real target (`<spine>/core/...`). Collapsing it as text
yields a nonexistent `.claude/scripts/...` path and a "no such file" error.

## 0. Preflight

Run the cheap health check, exactly as `core/skills/task/SKILL.md` step 0 and
`/spine` do:

```
${CLAUDE_SKILL_DIR}/../../scripts/setup --check --project <project root>
```

`ok`/`unpinned`: continue. `mismatch-warn`: show it, continue.
`mismatch-strict`: stop, same as `/task` — don't classify or fetch anything
until it's resolved. Could-not-run: note it and continue (this is low-stakes
until the task actually starts). If the install looks broken, say so and point
at `/spine` for the full diagnosis.

## 1. Get the ticket

Fetch via the tracker adapter (`core/ADAPTER-CONTRACT.md §3.4`):

```
<project root>/.spine/adapters/ticket-fetch "<key-or-url>"
```

Exit 0: parse its JSON (`{key, title, description, url, ...}`) — that's the
ticket. **Any non-zero exit, or no adapter installed, or no key given: fall
back to a manual paste** — ask the engineer to paste the ticket's title and
description. A fetch failure is never a hard stop (§3.4); the front door still
opens by hand. Either way, also derive the branch's ticket key
(`${CLAUDE_SKILL_DIR}/../../scripts/ledger ticket-from-branch --project
<project root>`) — if it disagrees with the key you fetched, say so and ask
which is right rather than guessing.

## 2. Is this even a task?

Before sizing anything, check whether this belongs in the pipeline at all
(`docs/proposals/intake-and-adaptive-autonomy.md` §2, the "not-a-task" tier):

- **A question or read-only investigation** ("how does X work", "why is this
  failing") with no code change: say so and offer to just answer it — no task,
  no trace. Don't ceremony-ize a question.
- **An exploratory spike** whose point is "I don't know what I want yet":
  spine is the wrong tool (`docs/tradeoffs.md`). Say so plainly and suggest
  working outside the pipeline, then running `/intake` again once the shape is
  known.

Only if it's a real change to the codebase do you continue to sizing.

## 3. Clarify — in session, never a file

Rewrite the ticket into a short brief in your own words, **marking what's
confirmed by the ticket vs. what you're inferring**. Ask the engineer only
*material* questions — ones whose answer changes scope, risk, or approach —
and batch them into a single `AskUserQuestion`; don't interrogate. This brief
is the task description research will ground on; it lives in this session (and,
for a Class 1/2 route, in the handoff at §7), never in a `docs/` file.

## 4. Ground the size in the real code — bounded

A size estimate from ticket text alone is a guess. Do a **bounded** pre-scan,
in the spirit of `/adopt`'s survey — enough to size, not task-level research:

- Locate where the change would land (grep/read the relevant area).
- Check `<project root>/.spine/protected-paths.conf` against those paths.
- Detect the Class-2 signals (§5) — auth, schema/migration, a public/declared
  contract, more than one owned system, anything irreversible.

Stop as soon as you can classify. If you *can't* size it without going deeper,
that's the low-confidence case — §6 leads with "scope it first."

## 5. Propose a class, with its evidence

Never a bare label — carry the reason. Apply, in order:

- **Class 2 (governed)** if any Class-2 signal fired: a protected-path match,
  auth/authz, a data migration, an external/declared contract, ≥2 owned
  systems, or an irreversible release. These are non-negotiable — a change
  that touches them is Class 2 regardless of how few lines it is.
- **Class 0 (trivial)** if it looks like ≤2 files, ≈15 lines (or this project's
  `.spine/profile.json` `class0_max_files`/`class0_max_lines` if set), no new
  public symbol, no protected path — and it's a real committed change, not a
  question.
- **Class 1 (standard)** otherwise — the default, and the sweet spot for a
  small bug or enhancement.

State the proposed class *and the evidence* (predicted files, which signals
fired or didn't). This is the objective, checklist-gated classification that
keeps "it's small" from silently dodging warranted rigor
(`docs/proposals/intake-and-adaptive-autonomy.md` §4/§11).

**For a Class 1 task, also propose an autonomy** — how many *stops* the flow has
(`core/skills/task/SKILL.md`'s autonomy dial, orthogonal to class). Read it off
the same signal: a small, low-complexity Class 1 with crisp scope → `auto` (spine
runs research→plan→implement→verify→ship and hands back a draft PR, no scheduled
stops); a substantial Class 1 → `checkpointed` (approve the plan, then one finish
action); a Class 1 near the Class 2 boundary, or anything you're less sure of →
`guided` (stop at each phase). Class 2 is always `guided` (the ceiling); Class 0
is `traced`. **Cap the proposal at `.spine/profile.json`'s `autonomy_ceiling`**
if set (a `regulated` team caps at `checkpointed`, so `auto` is never offered
there); absent = no team cap beyond the class rule. Like the class, this is a
proposal the menu can dial up or down — but never above the ceiling.

## 6. The confidence-weighted menu

How much you ask scales with how sure you are — don't turn the front door into
its own ceremony:

The menu covers the class *and* — for Class 1 — the autonomy: the recommended
option is a `(class, autonomy)` flow, and its alternatives include dialing
autonomy down (more stops, e.g. `auto`→`checkpointed`→`guided`) as well as class
escalation and scope-first.

- **Confident**: state the recommendation and ask for a near-one-tap confirm
  ("This looks like Class 1, `auto` — go, or pick a flow with more stops?").
  Don't make them read three paragraphs.
- **Torn** (a real fork): present the recommended class plus the genuine
  alternatives via `AskUserQuestion` — typically *recommended class* /
  *escalate to the next class up* / *scope it first* (a bounded research spike
  whose only output is a better classification).
- **Low confidence** (you couldn't size it in §4): **lead** with "let me scope
  it first" — propose the bounded spike honestly rather than pretending to a
  class you can't defend.

Always offer escalate-up. If the engineer chooses a class **below** your
recommendation, that's their call, but record it as an override so `/costs`
can see it: note it plainly now, and it will be carried into the task's ledger
(`class_declared` below the recommended, plus a one-line deviation at task
setup). Escalation and same-as-recommended need no such record.

## 7. Route

Once the class is confirmed:

- **not-a-task** (from §2): you already handled it — answer or point outside
  the pipeline. Nothing else runs.
- **Class 0 (traced-trivial)**: make the edit, then follow the traced-trivial
  steps in `core/skills/task/SKILL.md` §1 directly — commit carrying a
  `Spine-Ticket: <key>` trailer and record it with
  `${CLAUDE_SKILL_DIR}/../../scripts/ledger trace <key> "<one line>" --project
  <project root>`. You already know the ticket key from §1, so pass it. No task
  folder, no phases.
- **Class 1 / Class 2**: write the handoff so the task flow picks up your
  classification instead of re-doing it, then continue **as the task**:

  ```
  # .spine/current-intake  (consumed and deleted by core/skills/task/SKILL.md §1)
  {"ticket":"<key|null>", "class":<0|1|2>, "autonomy":"<auto|checkpointed|guided>",
   "description":"<the clarified brief>", "class_below_recommended":<true|false>,
   "recommended_class":<0|1|2>, "at":"<iso8601>"}
  ```

  Then **follow `core/skills/task/SKILL.md` from §1 onward in this same
  session** — do *not* invoke `/task` via the Skill tool (it is
  `disable-model-invocation` and human-typed; the human already started this
  flow by running `/intake`, so you continue its written steps directly). §1
  detects `.spine/current-intake`, uses the confirmed class without
  re-prompting, records the ticket, and deletes the handoff. Everything after
  — research, plan approval, implement, verify, ship — is unchanged.

You are the front door, not a new pipeline: the research→ship flow lives in
`/task` and stays there. `/intake` only decides *which* flow a ticket deserves
and carries the ticket in.
