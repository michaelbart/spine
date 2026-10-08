# Baseline: how spine is performing (2026-10-08)

Produced by `core/scripts/spine-stats` (read-only) over the two projects that use
spine most: **turnpilot** (76-77 task folders, a live project; event log since 2026-10-06) and
**tgml** (70 task folders, no event log, most transcripts already pruned). Numbers
only; no code or prompt text. Re-run with
`core/scripts/spine-stats --project <path>`.

## Correction to earlier estimates

The first token estimates quoted in conversation (spine ≈ 44% of fresh input,
verify subagents ≈ 36M) counted every transcript content block as a separate
message. A message with N blocks is logged N times with identical usage, so those
figures were about double. This report dedupes by message id (turnpilot: 43,870
usage lines are 21,624 distinct messages in the main sessions alone). All figures
below use the deduped counts.

## What the data supports

1. **Adversaries find fixable things.** About half of what the falsifier and
   security adversaries raise in turnpilot ends up marked *fixed* (53% and 50%);
   in tgml about a quarter (26% and 24%). About a quarter of security runs found
   nothing in turnpilot (23 of 88) and over half in tgml (36 of 66). "Fixed"
   undercounts value: ui-fidelity shows 9% fixed because its findings go to a
   human disposition (carry/decline) instead.
2. **Three floor layers never ran in either project.** `callers`, `clone-scan`
   and `mutate` are DEGRADED (unavailable) in every recorded task (67/67 in
   both). The README's "the floor fails on new duplication" is therefore not
   true in practice for these projects. This is a gap to close or stop
   advertising (plan item E10).
3. **Class 2 is common in turnpilot (67%) but not obviously over-assigned.** Only
   3 Class 2 tasks finished with zero fixed findings and zero deviation records,
   so the over-classification suspicion about `/intake` counting protected-file
   reads (plan D7) is mostly not supported by this metric. tgml is 34% Class 2
   with 0 such tasks.
4. **Hook friction is real: 26.6 interventions per shipped task** in the (short)
   window where the event log exists, spread over `dep-gate` asks (157),
   `path-escalate` blocks (154) and `phase-gate` blocks (8). None of these carries
   a reason because the reason-logging change is not merged into the live
   checkout yet; the cause stays unproven until it is.
5. **Spine-attributed work is the majority of fresh input in turnpilot (58%)** but
   a minority of output (30%). By line item, `/ship` on the main thread is the
   largest single spine consumer (9.0M fresh input) ahead of `/task` (5.7M) and
   `/verify` main (5.6M); the falsifier subagent totals ~11.7M across tagged and
   untagged turns, the biggest single agent. This points `/ship` (plan D8) and
   the falsifier at the top of any token-saving list.
6. **Work volume:** turnpilot's 76 tasks took a median 2.0 h of active time
   (gaps over 30 minutes excluded) and 1.0M fresh-input-plus-output tokens each;
   total active time about 7.9 days.
7. **Spine's own size:** 19 skills, 4,670 skill lines, 25 scripts + 6 hook files,
   7,240 lines, after the team/cross-repo removal (before it: 20 skills with 5,610 skill lines, and 8,079 script and hook lines).

## What it cannot tell us

- **Whether spine pays for itself.** The untagged 42% of turnpilot input is
  implementation plus any non-spine conversation, and cannot be separated. There
  is no spine-free comparison (plan B8).
- **Rework and escaped defects.** Not computed yet.
- **Floor-layer value.** `verify.md` stores only the final round's result, so a
  failure fixed in an earlier round is invisible; "nothing ever fails" is not
  evidence the layers are useless.
- **tgml tokens.** Claude Code has pruned most of tgml's transcripts (17 of 70
  tasks attributable). This is the reason the plan writes a token rollup into the
  `shipped` event at ship time.
- **Cost in dollars.** No price table yet; tokens only.
- **Attribution method.** A turn belongs to the `work/<task-id>/` path the
  transcript touched last. It is approximate, and cannot see a task before its
  folder is first touched.

## Raw output

### turnpilot

```
spine-stats  project=turnpilot  events=393  tasks=77

== Tokens (deduped by message id; tokens only, no prices)
  31,665 assistant messages   fresh-in 92.7M   cache-read 9,410.3M   out 19.0M
  spine-attributed (skill-tagged turns + researcher/falsifier/security/ui-fidelity agents): 58% of fresh input, 30% of output
  the rest = implementation plus any non-spine conversation in this project; not separable here
  skill / side or agent  msgs    fresh-in  cache-read  out  
  (none) / main          16,147  35.4M     6,282.5M    12.8M
  ship / main            2,389   9.0M      1,200.5M    1.5M 
  verify / falsifier     2,849   6.3M      186.2M      0.6M 
  task / main            1,295   5.7M      175.6M      0.5M 
  verify / main          1,479   5.6M      775.2M      1.0M 
  (none) / falsifier     2,609   5.4M      175.5M      0.5M 
  verify / ui-fidelity   550     4.8M      30.1M       0.5M 
  task / researcher      970     4.1M      53.2M       0.2M 
  verify / security      942     4.1M      44.6M       0.2M 
  (none) / researcher    477     3.0M      36.7M       0.1M 
  (none) / security      650     2.5M      23.8M       0.2M 
  (none) / fork          322     1.8M      218.8M      0.3M 
  77 tasks attributed by the work/<task-id> path each transcript touched last (approximate)
  per task: median fresh-in+out 1.0M, max 5.5M; median active time 2.0h (gaps over 30m excluded), total active 7.9d

== Tasks
  77 task folders; states: {'done': 76, 'research': 1}
    class 1 / ?: 1
    class 1 / auto: 5
    class 1 / guided: 19
    class 2 / ?: 2
    class 2 / auto: 2
    class 2 / guided: 48
  Class 2 share: 68%
  Class 2 tasks done with 0 fixed adversary findings and 0 deviations: 3 (over-classification candidates)
  wall-clock cycle time class-set..shipped, median 1.9h (n=11; event log only, so recent tasks); see the tokens section for active time on every task
  verify first-pass PASS (event log only): 12/12   deviation records total: 87

== Hook friction (event log)
  321 interventions; 0 carry a reason; 12 shipped tasks in log; 26.8 per shipped task
  event       hook           rule                why  n  
  hook-ask    dep-gate       (no reason logged)       158
  hook-block  path-escalate  (no reason logged)       155
  hook-block  phase-gate     (no reason logged)       8  
  worst tasks: 20261006-real-inspections-deposits (52); 20261007-real-notifications-search (32); 20261007-vendor-technician-real-data-invoices-storage (31); 20261007-settings-take-effect (30); 20261007-turn-loop-web-actions (29)
  bypasses: 0   class escalations: 1

== Adversary yield (verdict artifacts: fixed / raised)
  agent        tasks  raised  high/med/low  fixed  yield  tasks w/ 0 findings
  falsifier    89     372     19/100/252    196    53%    5                  
  security     88     202     9/40/153      101    50%    23                 
  ui-fidelity  23     267     55/148/64     25     9%     1                  
  `fixed` is set only when /verify marked the finding fixed. The rest were carried, declined or left open, so a low yield
  (ui-fidelity) means findings are mostly dispositioned by the human, not that they were noise (core/ADAPTER-CONTRACT.md §5).

== Floor layers across 76 verify.md files (PASS / FAIL / DEGRADED)
  capability                             PASS  FAIL  DEGRADED
  callers                                0     0     67      
  clone-scan                             0     0     67      
  dep-diff                               67    0     0       
  lint                                   67    0     0       
  migrate-rehearse                       26    0     0       
  mutate                                 0     0     44      
  secret-scan                            67    0     0       
  smoke-golden                           65    0     0       
  smoke-run                              65    0     0       
  smoke-seed                             65    0     0       
  smoke-seed / smoke-run / smoke-golden  2     0     0       
  test                                   67    0     0       
  typecheck                              67    0     0       
  caveat: verify.md holds the FINAL round's result, so failures fixed in an earlier round do not show here.
  degraded (never ran) in every recorded task: callers, clone-scan, mutate

== Command census (turns carrying each skill tag; by messages)
  skill             messages  sessions  last used 
  verify            5824      50        2026-10-08
  ship              2410      48        2026-10-08
  task              2283      57        2026-10-08
  run               203       4         2026-10-04
  update            58        9         2026-10-06
  autopilot         37        2         2026-09-25
  design            29        1         2026-09-19
  roadmap           22        3         2026-10-06
  claude-in-chrome  3         1         2026-09-24
  spine             3         1         2026-10-05
  no tagged turns in this project: adopt, bootstrap, intake, note-issue, prompts, prototype, ratchet, remap, ticket, wayfinder

== Complexity (this spine checkout)
  19 skills, 4,670 skill lines; largest: task 699, ship 625, verify 534, design 444, roadmap 308
  25 scripts + 6 hook files, 7,336 lines
```

### tgml

```
spine-stats  project=tgml  events=0  tasks=70

== Tokens (deduped by message id; tokens only, no prices)
  1,755 assistant messages   fresh-in 4.3M   cache-read 531.9M   out 0.7M
  spine-attributed (skill-tagged turns + researcher/falsifier/security/ui-fidelity agents): 41% of fresh input, 16% of output
  the rest = implementation plus any non-spine conversation in this project; not separable here
  skill / side or agent  msgs   fresh-in  cache-read  out 
  (none) / main          1,011  2.5M      459.4M      0.6M
  (none) / falsifier     464    0.8M      29.3M       0.0M
  task / main            81     0.4M      31.8M       0.0M
  task / researcher      117    0.3M      4.9M        0.0M
  (none) / security      62     0.3M      1.4M        0.0M
  vercel:deploy / main   12     0.0M      4.3M        0.0M
  run / main             7      0.0M      0.5M        0.0M
  (none) / fork          1      0.0M      0.2M        0.0M
  17 tasks attributed by the work/<task-id> path each transcript touched last (approximate)
  per task: median fresh-in+out 0.1M, max 2.1M; median active time 22m (gaps over 30m excluded), total active 10.3h

== Tasks
  70 task folders; states: {'done': 70}
    class 1 / auto: 46
    class 2 / ?: 1
    class 2 / auto: 1
    class 2 / guided: 22
  Class 2 share: 34%
  Class 2 tasks done with 0 fixed adversary findings and 0 deviations: 0 (over-classification candidates)
  wall-clock cycle time class-set..shipped, median - (n=0; event log only, so recent tasks); see the tokens section for active time on every task
  verify first-pass PASS (event log only): 0/0   deviation records total: 61

== Hook friction (event log)
  0 interventions; 0 carry a reason; 0 shipped tasks in log
  bypasses: 0   class escalations: 0

== Adversary yield (verdict artifacts: fixed / raised)
  agent      tasks  raised  high/med/low  fixed  yield  tasks w/ 0 findings
  falsifier  68     380     20/53/307     97     26%    2                  
  security   66     91      1/19/71       22     24%    36                 
  `fixed` is set only when /verify marked the finding fixed. The rest were carried, declined or left open, so a low yield
  (ui-fidelity) means findings are mostly dispositioned by the human, not that they were noise (core/ADAPTER-CONTRACT.md §5).

== Floor layers across 70 verify.md files (PASS / FAIL / DEGRADED)
  capability        PASS  FAIL  DEGRADED
  callers           0     0     67      
  clone-scan        0     0     67      
  dep-diff          67    0     0       
  lint              67    0     0       
  migrate-rehearse  14    0     0       
  mutate            0     0     24      
  secret-scan       67    0     0       
  smoke-golden      28    0     3       
  smoke-run         28    0     3       
  smoke-seed        28    0     3       
  test              64    0     3       
  typecheck         67    0     0       
  caveat: verify.md holds the FINAL round's result, so failures fixed in an earlier round do not show here.
  degraded (never ran) in every recorded task: callers, clone-scan, mutate

== Command census (turns carrying each skill tag; by messages)
  skill          messages  sessions  last used 
  task           198       1         2026-09-02
  vercel:deploy  12        1         2026-08-31
  run            7         1         2026-08-31
  no tagged turns in this project: adopt, autopilot, bootstrap, design, intake, note-issue, prompts, prototype, ratchet, remap, roadmap, ship, spine, ticket, update, verify, wayfinder

== Complexity (this spine checkout)
  19 skills, 4,670 skill lines; largest: task 699, ship 625, verify 534, design 444, roadmap 308
  25 scripts + 6 hook files, 7,336 lines
```
