# Design — handoff and product-spec intake (loaded on demand)

Loaded from SKILL.md when: a first run was given `--handoff`, or this run was `--handoff`-grounded.

**If `--handoff <path>` was given on a first run** (§0), read it now,
alongside the charter, before proposing anything below — a product spec or
an external design document genuinely bears on these categories (a
real-time collaborative feature says something about state-management and
persistence a charter alone won't spell out). Cite it the same way you'd
cite a charter line when it's what a proposed answer actually follows
from. This doesn't add a seventh category or change what counts as
grounding for the charter DRAFT check above — it's additional context for
the same six questions, nothing more.

**If this run was `--handoff`-grounded**, stamp it consumed first — same
mechanism the re-entry mode uses (see its own "Stamp the handoff as
consumed" note above): write `.spine/handoff-consumed-sha`, line 1 the
handoff path, line 2 `git hash-object <path>`'s output. Skip this
entirely if `--handoff` wasn't used this run — there is nothing to stamp.
