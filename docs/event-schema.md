# Event log schema (v1)

`<project>/.spine/events.jsonl` is a local, append-only record of what happened
and when. One JSON object per line. It is gitignored and per-machine.

**Rules (do not break these):**

- Nothing reads it to decide anything. It exists so a later recap
  (`core/scripts/spine-stats`, planned in `docs/improvement-plan.md` B4) can say
  what actually happened.
- Writing never fails a task: `core/scripts/spine-event` and
  `core/hooks/_spine-event` return 0 and print nothing on every failure path.
- No secrets, no content. Hook events record a rule id, a repo-relative path,
  the command-position words and a short hash of a Bash command, **never the
  command itself or its arguments**.
- Values are strings (the writer is `key=value`), except `v`, which is a number.
- Add fields freely; never change the meaning of an existing field. A breaking
  change bumps `v`.

## Common fields (every line)

| Field | Meaning |
|---|---|
| `v` | schema version, `1` |
| `ts` | UTC time, `YYYY-MM-DDTHH:MM:SSZ` |
| `task` | task id from `.spine/current-task`, or the id the writer was given (`set-state`); empty when no task is active |
| `event` | event name, below |
| `session` | Claude session id, when known: the hook input's `session_id`, else `CLAUDE_CODE_SESSION_ID`. Joins the event to `~/.claude/projects/<project>/<session>.jsonl` and its `subagents/` |

## Events

### Written by hooks (`core/hooks/*`)

`hook-block` (the hook denied the call) and `hook-ask` (the hook asked the
human). A Bash call can produce both, from different hooks, at the same `ts`.

| Field | Meaning |
|---|---|
| `hook` | `path-escalate`, `phase-gate`, `dep-gate` |
| `tool` | `Edit`, `Write` or `Bash` |
| `rule` | why: `protected-path`, `migration-path` (path-escalate); `pre-plan-write` (phase-gate); `manifest-path`, `install-command`, `no-project-root` (dep-gate); `unresolved-bash-target` (all three); `internal` (a hook could not load a helper and failed closed) |
| `target` | repo-relative path, when one was resolved |
| `why` | only with `unresolved-bash-target`: `var-glob-quote` (a variable, glob or quote in the target) or `relative-after-cd` (a relative target after a `cd` in the same command) or `other` |
| `cmds` | Bash only: command-position words, comma-joined, at most 6 (`cd,pnpm,tail`). Leading `VAR=value` assignments are skipped |
| `cmd_sha` | Bash only: first 10 hex of the command's sha256; groups repeats of the same command |

### Written by skills and scripts

| Event | Fields | Written when |
|---|---|---|
| `class-set` | `class` (0/1/2), `autonomy` | the human confirms the class. **The first `class-set` for a task is its start time** for cycle-time purposes |
| `class-escalated` | `from`, `to`, `when` | a task is raised to Class 2 |
| `map-refresh` | `result` (ok/failed), `was` (stale n/empty/missing) | `/task` refreshes `docs/map.md` |
| `phase` | `from`, `to` | `core/scripts/set-state` moves a task between `research`, `plan`, `implement`, `verify`, `ship`, `done`. `from=none` for the first write |
| `plan-approved` | `autonomy` | the human approves the plan |
| `deviation` | `kind` (`real` or `setup`), `tier` (`record-and-proceed` or `halt`, real only) | a deviation or setup event is recorded during implement |
| `verify-result` | `result` (PASS/FAIL), `reason` | end of each `/verify` pass. The pass number is the count of earlier `verify-result` lines for the same task |
| `adversary` | `agent` (falsifier/security/ui-fidelity), `verdicts` (kept this pass), `max_severity` | per adversary, per `/verify` pass |
| `bypass` | `reason` | `/ship --bypass` |
| `shipped` | (none yet; `tokens_*` planned, B4) | `/ship` commits |

## Derived, not logged

- **Adversary yield / finding disposition:** read `work/<task>/artifacts/<agent>-verdict.json`;
  each kept verdict carries `disposition` (`fixed` or `not_fixed`,
  `core/ADAPTER-CONTRACT.md` §5).
- **Verify round number:** order `verify-result` events per task.
- **Phase durations:** differences between consecutive `phase` events.
- **Tokens:** from the Claude transcript named by `session` (and its
  `subagents/` files), not from this log.

## Known gaps

- History before 2026-10-06 has no events at all; older tasks can only be
  analyzed from task-folder timestamps, git and transcripts.
- Events from before schema v1 (the first 2026-10-06 lines) lack `v` and
  `session`; readers must treat a missing `v` as 0.
