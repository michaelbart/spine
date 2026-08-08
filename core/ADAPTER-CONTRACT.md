# Adapter contract

Authoritative for every file in `core/scripts/`, every adapter in a project's
`.spine/adapters/`, and `core/scripts/adapter-conformance`. Nothing here names
a language, framework, or tool — this document, like the core, is stack-blind.

## 1. What a capability is

A capability is an executable at `.spine/adapters/<name>` in the installed
project. The core never calls a tool directly; it calls a capability name.
Thirteen capabilities exist:

`typecheck`, `lint`, `test`, `test-changed`, `secret-scan`, `dep-diff`,
`clone-scan`, `callers`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`,
`migrate-rehearse`.

`.spine/capabilities.json` marks each `implemented`, `unavailable`, or
`not-applicable`, each with a `reason`. Only capabilities marked `implemented`
are ever invoked by `floor` or any skill. An absent or degraded capability is
never silently skipped — its gate is recorded as degraded, explicitly, in
`verify.md`, the ledger, and the delta briefing.

## 2. Exit code and output discipline

Every adapter, in every mode, obeys:

- **Exit 0 means pass. Any non-zero exit means fail.**
- **On pass: stdout is exactly one line.** No exceptions. This is what keeps
  a floor run's own output small enough to stay in context.
- **On fail: stdout/stderr carries full diagnostics** — everything a human or
  an adversary subagent would need to see the failure without re-running the
  command themselves.
- Adapters never prompt. Never read from a TTY. Never mutate files outside
  what the capability inherently requires (e.g. a formatter check must not
  reformat in place; that's a different capability if it's ever wanted).

## 3. Invocation convention

All adapters run with CWD set to the project root (the directory containing
`.spine/`). Beyond that, the convention is deliberately small:

| Input needed | Convention |
|---|---|
| Nothing (operates on the whole working tree) | No args, no stdin. Applies to `typecheck`, `lint`, `test`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`, `migrate-rehearse`. |
| A changed-file set | Newline-separated relative paths on **stdin**. Applies to `test-changed`, `clone-scan`. `secret-scan` accepts the same on stdin as a scope-narrowing *option*; if stdin is empty or absent it falls back to scanning the full tracked tree. |
| An output artifact path | **First positional argument.** Applies to `callers` and `dep-diff` — both additionally emit an artifact (the caller map, the dependency diff) to that path, in whatever format the adapter defines, while pass/fail semantics stay governed by exit code alone. `callers` also reads the changed-file/symbol set from stdin. |
| A comparison base for anything diffed against history (`dep-diff`, and any adapter that needs one) | Environment variable `SPINE_BASE_REF`, default `HEAD`. Never a positional argument — keeps the positional slot reserved for the output path. |

No capability takes more than one positional argument. This is the single
convention every adapter and `floor` itself must obey; it resolves build-
prompt open question §5.7.

## 4. The self-test convention (what makes conformance possible)

Every adapter additionally supports exactly one more invocation shape,
mutually exclusive with normal operation:

```
.spine/adapters/<name> --self-test pass
.spine/adapters/<name> --self-test fail
```

In `--self-test pass`, the adapter builds or selects a fixture *inside its
own temp scratch space* (never the project's real files) that is guaranteed
to satisfy the capability, runs itself against it, reports exit 0 with the
one-line-success discipline, and cleans up before returning.

In `--self-test fail`, the adapter does the same with a fixture guaranteed to
violate the capability, and reports non-zero with full diagnostics.

This is a builder decision, not something the build prompt specifies
directly — it's the mechanism that makes `adapter-conformance` possible
without inventing a second, stack-specific fixture-delivery channel. An
adapter author validating a hand-written adapter gets the same self-test
modes `adapter-conformance` uses; there's only one path to "conformant."

Design note this implies: an adapter must be able to construct at least one
concrete pass case and one concrete fail case for its own capability, fully
self-contained. Where a capability's tool has nothing meaningful to
self-test (e.g. it always trivially passes/fails independent of input), that
is itself a sign the capability should be `unavailable` or `not-applicable`
rather than `implemented` — conformance existing to prove the teeth are real
(`core/scripts/adapter-conformance` on an always-passing adapter must fail
its own suite by construction, per §2.5 of the build prompt).

## 5. Verdict schema (adversary output, validated by `verdict-filter`)

Falsifier and security adversary subagents (Phase C) write one JSON object
per run:

```json
{
  "agent": "falsifier",
  "task_id": "20260101-example",
  "attacked": [
    "acceptance check 3 against a concurrent-write input",
    "invariant: org-scoped reads, per caller map entry X"
  ],
  "verdicts": [
    {
      "claim": "string, required, non-empty",
      "severity": "high | medium | low",
      "evidence": {
        "kind": "file_line",
        "file": "src/foo",
        "line": 42
      }
    },
    {
      "claim": "another finding",
      "severity": "medium",
      "evidence": {
        "kind": "command",
        "command": ".spine/adapters/test --stub-feature",
        "output": "captured output proving the claim"
      }
    }
  ]
}
```

`attacked` is mandatory and non-empty even when `verdicts` is empty — it's
the clean-bill enumeration ("what was attacked") the build prompt requires.
`evidence.kind` is `file_line` (requires non-empty `file` and integer `line`)
or `command` (requires non-empty `command` and `output`). Any verdict missing
a required field, with an empty claim, an unrecognized severity, or
malformed evidence is **dropped** by `verdict-filter` before `/verify` ever
reads the file — a script decides what counts as evidence, never a model.

## 6. Task-ID and commit trailer convention (resolves open question §5.1)

Task IDs are `<YYYYMMDD>-<kebab-slug>`, e.g. `20260807-shared-unit-types`.
The work folder is `work/<task-id>/`. Every commit produced by `/ship` for
that task carries the trailer:

```
Spine-Task: <task-id>
```

on its own line in the commit message footer, composing cleanly alongside
any other trailer (e.g. `Co-Authored-By:`) and any Conventional-Commits-style
subject prefix a project's own CI enforces — the trailer says nothing about
commit-message grammar and never fights it.

This makes the untracked-commit ratio (§2.6 of the build prompt) mechanical:
`core/scripts/ledger scan-untracked-ratio` greps `git log` trailers over a
window and reports commits with no `Spine-Task:` trailer and no
`Spine-Bypass:` trailer as Class 0 or off-spine work, by definition — no
model judgment involved. `/ship --bypass <reason>` writes `Spine-Bypass:
<reason>` instead of (or alongside) the task trailer, so bypassed work is
still mechanically visible and distinguishable from silent drift.
