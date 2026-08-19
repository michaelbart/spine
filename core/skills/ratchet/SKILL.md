---
name: ratchet
description: Convert a finding that's recurred twice into a deterministic check, and delete the prose rule it supersedes. The only skill allowed to add to CLAUDE.md or the rules directory.
disable-model-invocation: true
argument-hint: <description of the recurring finding>
---

You are running `/ratchet` on `$ARGUMENTS` — a finding that has come up
**twice**: two `deviations.md` records (any tasks) citing substantially the
same fact, two adversary verdicts across different `verify.md` runs with the
same underlying claim, or two instances of an engineer relaying the same
review comment. If you can't point at two real prior instances, this isn't
ratchet-eligible yet — say so and stop; a rule added on a single instance is
prose accretion with a `/ratchet` label on it, not what this command is for.

Once for real: convert it into exactly one of these, cheapest first —

1. **An adapter-level check.** If the finding is something a capability
   should have caught, edit the relevant `.spine/adapters/<name>` (project-
   owned, stack-specific — this is fine to be stack-specific, it lives in
   the one place stack-specificity belongs). Re-run
   `${CLAUDE_SKILL_DIR}/../../scripts/adapter-conformance <name>` after —
   an edited adapter that breaks its own self-test isn't done.
2. **A test.** If the finding is a specific behavior that should never
   regress, add the test that pins it.
3. **A `.spine/protected-paths.conf` entry.** If the finding is "this path
   needed more scrutiny than it got," tag it appropriately (`#migration`,
   `#manifest`, or plain) so `path-escalate`/`dep-gate` catch it mechanically
   from now on.
4. **A project-owned `.claude/rules/*.md` path-scoped rule (a real file, not
   a symlink — this is project-specific, not universal, so it does not
   belong in the shared spine core at `core/rules/`), or a `CLAUDE.md`
   line** — only when none of the above can express it, because both are
   always-loaded or load-on-path-read surface, and that surface is capped
   and audited (build prompt §2.6). This is the only skill permitted to add
   to either. (A finding general enough to belong in the *shared* stack-
   blind `core/rules/` — rare — is a spine-maintainer change to the spine
   checkout itself, not a routine `/ratchet` run inside one project.)

**Additions pay a deletion.** If (and only if) you added to `CLAUDE.md` or a
rules file, find the prose this new deterministic check makes redundant —
a guideline telling the model to remember to do the thing the check now
does mechanically — and delete it in the same change. If nothing qualifies
for deletion, that's a sign the addition doesn't actually supersede
anything and belongs at tier 1–3 instead; reconsider before committing to
tier 4. Options 1–3 carry no deletion obligation — they don't touch the
taxed surface.

If you touched `CLAUDE.md`, recount the lines inside its
`<!-- spine:begin -->`/`<!-- spine:end -->` block — every CLAUDE.md
bootstrap or `/adopt` ever writes has this block (`core/skills/bootstrap/
SKILL.md` §5, `core/skills/adopt/SKILL.md` §5), never just a bare
template — and the ≤60 cap applies only to lines inside it. This is the
only skill allowed to edit inside the block, and never touch anything
below `<!-- spine:end -->` — that's the engineer's own content (real, on
an adopted project, or simply not yet written, on a bootstrapped one) and
it never counts against the cap. If the count doesn't hold after the
paired deletion, the deletion wasn't equivalent; find a better trade,
don't just truncate.

Report what you converted the finding into, what (if anything) you deleted,
and the two prior instances that made this ratchet-eligible.
