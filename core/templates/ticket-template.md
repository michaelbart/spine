<!--
Human-owned template — lives at .spine/ticket-template.md in each installed
project (scaffolded there by /ticket on first run from this file).

Edit .spine/ticket-template.md to match your team's JIRA fields: rename
section headings, reorder them, add extra {{token}} placeholders for fields
/ticket won't fill (priority, component, labels, etc.) — those stay verbatim
in the output as manual-fill blanks.

Tokens /ticket fills automatically:
  {{summary}}              — what the task made true (from plan.md ## The gist)
  {{acceptance_criteria}}  — the numbered acceptance checks (from plan.md ## Steps)
  {{qa_notes}}             — adversary review findings + deviations (from verify.md / deviations.md)
  {{files_changed}}        — files the task touched (from plan.md ## Predicted touch)
  {{epic_link}}            — placeholder reminding you to fill in the JIRA epic key

Any other {{...}} token you add is left verbatim in the output for manual fill.
Remove any section you don't need — /ticket won't re-insert it.
-->

h2. Summary

{{summary}}

h2. Acceptance criteria

{{acceptance_criteria}}

h2. QA notes

{{qa_notes}}

h2. Files changed

{{files_changed}}

----

Epic: {{epic_link}}
