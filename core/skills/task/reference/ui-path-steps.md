# Task — UI-path steps (loaded on demand)

Loaded from SKILL.md when: the task touches a declared UI or content path.

**UI-touching tasks: the handoff is required grounding.** If the task's
description or likely touch set includes a declared UI path or content
path (`.spine/ui-paths.conf`, `.spine/ui-content-paths.conf`), tell the
researcher to read and cite, in the header's `files:` list, the affected
screen's `docs/ui/screens/<id>.json` **and** each of that screen's
screenshots (every path in its `screenshots` map), plus
`docs/ui/components.md` where a component's behavior matters. A UI task
researched only from code is how content gets invented; the headers are
also what makes `check-stale` notice when a mockup is replaced.


**UI-touching plans: run `content-sources-check` before presenting the plan.**

```
${CLAUDE_SKILL_DIR}/../../scripts/content-sources-check <task-id> --project <project root>
```

Exit 0 (including "not applicable") proceeds. Exit 1 means the plan's
`## Content sources` is missing, cites a nonexistent or untracked source,
omits a fixture/content file, or contains `source: none`. **Do not present
the plan.** For each `source: none`, stop and ask the human what the
content is or where it comes from (halt tier — the same stop-and-ask a
`halt` deviation is), then rewrite the entry as a real path or
`source: human — <what they said>` and re-run. This applies at every
class and autonomy, `auto` included: there is no self-approving your way
past content nobody defined. Ask it with `AskUserQuestion`, in this form (per `core/templates/human-touchpoint.md`):

<!-- touchpoint:start -->
> **Deciding:** what a piece of on-screen text should say, or where it comes from. It's yours because nothing in the project defines it, and I won't invent wording or sample data.
> **Need to know:** The plan puts "<the text or file>" on screen, and I found no source for it.
> **Recommend:** Just tell me the wording — it's the fastest way to be right.
> 1. **Give me the text** — next: I record it as your answer and carry on to the plan; cost: a minute; undo: yes
> 2. **Point me to where it lives** (a file or link) — next: I read it, cite it, and carry on; cost: a minute; undo: yes
> **Safe to ignore:** every entry that already has a source.
<!-- touchpoint:end -->
