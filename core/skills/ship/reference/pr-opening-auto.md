# Ship — auto draft-PR opening (loaded on demand)

Loaded from SKILL.md when: autonomy is auto or pr_open is all.

- **`auto`** — open a **draft** PR now via the `open-pr`
  capability (`core/ADAPTER-CONTRACT.md` §3.5), body =
  `work/<task-id>/pr-description.md` (§4a), head = the current branch, so the
  human's one remaining touchpoint is reviewing/merging it:

  ```
  SPINE_PR_TITLE="<commit subject>" \
    SPINE_PR_BODY_FILE=work/<task-id>/pr-description.md \
    <project root>/.spine/adapters/open-pr
  ```

  **Always a draft — this skill never merges** (proposal §6.2; the human marks
  ready and merges). Record the returned PR URL in the briefing (§4). If `open-pr`
  is `not-applicable`/absent or exits non-zero, degrade to the `guided` behavior:
  the commit is already made, so say plainly "couldn't open the PR (<reason>) —
  push and open it by hand" and note the gap; never silently drop it. (Profile-
  gated auto-open for `guided`, or disabling it for a team that prefers
  hand-opened PRs, is Phase 5.)
