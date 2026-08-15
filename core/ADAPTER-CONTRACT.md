# Adapter contract

Authoritative for every file in `core/scripts/`, every adapter in a project's
`.spine/adapters/`, and `core/scripts/adapter-conformance`. Nothing here names
a language, framework, or tool — this document, like the core, is stack-blind.

## 1. What a capability is

A capability is an executable at `.spine/adapters/<name>` in the installed
project. The core never calls a tool directly; it calls a capability name.
Fourteen capabilities exist:

`typecheck`, `lint`, `test`, `test-changed`, `secret-scan`, `dep-diff`,
`clone-scan`, `callers`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`,
`migrate-rehearse`, `contract-check` (Extension B — §3.2).

`contract-check` only ever exists in a repo that is a workspace member and
party (producer or consumer) to at least one declared contract
(`workspace.json`'s `contracts[]`, `core/skills/workspace/SKILL.md`) — a
plain single-repo project marks it `not-applicable` with reason "not a
workspace member," the same disclosed-degradation shape every other
inapplicable capability already uses. Unlike the other thirteen,
`contract-check` is never invoked by `core/scripts/floor`'s own dispatch
loop (floor's fixed capability sequence is unchanged by Extension B,
preserving single-repo behavior exactly) — `core/skills/verify/SKILL.md`'s
aggregation step invokes it directly, once per touched contract, only for
repos `core/scripts/contract-touch` puts in a task's blast radius. See §3.2.

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
| Nothing (operates on the whole working tree) | No args, no stdin. Applies to `test`, `mutate`, `smoke-seed`, `smoke-run`, `smoke-golden`, `migrate-rehearse`. |
| A changed-file set | Newline-separated relative paths on **stdin**. Applies to `test-changed`, `clone-scan`, `lint`, `typecheck`. `secret-scan` accepts the same on stdin as a scope-narrowing *option*; if stdin is empty or absent it falls back to scanning the full tracked tree. |
| An output artifact path | **First positional argument.** Applies to `callers` and `dep-diff` — both additionally emit an artifact (the caller map, the dependency diff) to that path, in whatever format the adapter defines, while pass/fail semantics stay governed by exit code alone. `callers` also reads the changed-file/symbol set from stdin. |
| A comparison base for anything diffed against history (`dep-diff`, and any adapter that needs one) | Environment variable `SPINE_BASE_REF`, default `HEAD`. Never a positional argument — keeps the positional slot reserved for the output path. |

No capability takes more than one positional argument. This is the single
convention every adapter and `floor` itself must obey; it resolves build-
prompt open question §5.7.

### 3.1 `lint`/`typecheck` rescoping (Phase E) and the `--full` escape hatch

Through Phase D, `lint` and `typecheck` were whole-tree, no-stdin
capabilities. Phase E moved them to the changed-file-set bucket: a task's
floor result must reflect the task's own change, not the repository's
accumulated history, per the architecture decision recorded in
`docs/tradeoffs.md` (the "rescope, don't just add `--full`" question the
build declined to resolve unilaterally in Phase D — now decided). A
diff-scoped adapter is free to still invoke its underlying tool
whole-tree internally where the tool needs full-project context for
correct results (a type checker resolving imports, for instance) — the
contract governs what *input* the adapter receives and what *scope* its
pass/fail verdict covers, not how it shells out internally. What must not
happen is a whole-tree-scoped verdict failing a task for a violation in a
file the task never touched.

`lint` and `typecheck` additionally accept a `--full` flag: no stdin is
read (any piped stdin is ignored), and the adapter runs its full,
whole-tree check exactly as it did through Phase D. This is the only
sanctioned way to invoke either capability without a changed-file set —
mirroring the `--self-test` extension mechanism in §4, a second invocation
shape layered on top of the base convention, not a replacement for it.
`floor --full` (below) is the intended caller; a per-task `/task` run must
never pass `--full`. This exists because whole-tree enforcement still has
a real home — CI, and a dedicated cleanup task that pays down pre-existing
debt once — it has simply been moved out of the per-task gate, which was
failing every task in horizon for reasons no single task's diff could fix.

### 3.2 `contract-check` (Extension B) — which contract, not an output path

`contract-check` needs an input §3's table has no row for: **which
declared contract to check conformance against** — a repo can be party to
more than one. Two environment variables, following the same precedent
`SPINE_BASE_REF` already sets (an environment variable, not a second
positional argument — §3's "no capability takes more than one positional
argument" rule is unchanged):

| Variable | Carries |
|---|---|
| `SPINE_CONTRACT_NAME` | The contract's `name` from `workspace.json`'s registry. |
| `SPINE_CONTRACT_SPEC_PATH` | Absolute path to the spec file the caller already resolved (`<workspace-root>/<spec_path>`) — the adapter, run with CWD at its own repo root, has no way to find the workspace root on its own and is not expected to; the caller (`core/skills/verify/SKILL.md`'s aggregation, which already knows the workspace root) resolves it. |

No stdin, no positional argument, exit code alone governs pass/fail (§2) —
`contract-check` sits in §3's "operates on nothing" row in spirit, plus
these two required env vars. The check itself is stack-specific and
symmetric: a producer's `contract-check` verifies its real implementation
still matches the spec at `SPINE_CONTRACT_SPEC_PATH`; a consumer's verifies
its own usage still matches it (a generated client is current, a query
shape matches a schema, whatever that stack's truth is) — same capability
name, same env-var contract, opposite direction of proof, exactly as
`typecheck`/`lint` are the same capability name across every stack despite
checking entirely different things.

**Self-test**: `--self-test pass`/`--self-test fail` build their own
throwaway spec fixture inside scratch space, same as every other
capability (§4) — they do not read `SPINE_CONTRACT_SPEC_PATH` at all, since
self-test's entire point is proving the adapter's own logic without
touching anything real. An adapter that requires the real env vars to be
set even in self-test mode has misunderstood the convention.

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

**For a changed-file-set capability** (§3's second row — `test-changed`,
`clone-scan`, and, since Phase E, `lint`/`typecheck`), `--self-test pass`
must additionally prove the scoping itself, not just the underlying check:
the fixture must include a second, violating artifact that is *not* part
of the stdin set fed to the self-test run, and the adapter must still
report a clean pass. A self-test that only ever builds one clean file
proves the tool wrapper works; it proves nothing about whether the
adapter actually honors the changed-file scope versus silently checking
the whole scratch directory regardless of stdin. This is the fixture that
would have caught a rescoped-in-name-only adapter — see
`docs/tradeoffs.md`'s Phase E section for the concrete horizon/synthetic
runs that exercised it.

This is a builder decision, not something the build prompt specifies
directly — it's the mechanism that makes `adapter-conformance` possible
without inventing a second, stack-specific fixture-delivery channel. An
adapter author validating a hand-written adapter gets the same self-test
modes `adapter-conformance` uses; there's only one path to "conformant."

**The self-test must exercise the adapter's real invocation path, not a
parallel check that happens to agree with it on the fixture.** A self-test
that re-implements its own ad-hoc pass/fail logic (e.g. a second, simpler
grep or comparison inlined in the `--self-test` branch) instead of calling
through the same function/command the normal-mode branch calls proves
nothing about the adapter's actual behavior — it can pass forever while
the real path is broken on inputs the toy check never represents (g1-tee-
waitlist testbed finding: a `callers` adapter's self-test grepped a bare
symbol name against a fixture, while the real adapter greps a full
repo-relative file path against real source — a shape neither the pass nor
the fail fixture ever exercised, so a heuristic that structurally could
not match any real relative or aliased import stayed "conformant"
indefinitely). Build the fixture, then invoke the same code path normal
mode uses against it — never a second implementation of the check.

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
`evidence.kind` is `file_line` (requires non-empty `file` and integer
`line`), `command` (requires non-empty `command` and `output`), or
`decision` (design-stage extension, Extension A — requires non-empty
`decision_id` and `quote`: `{"kind":"decision","decision_id":"D-<n>",
"quote":"<verbatim span>"}`). Any verdict missing a required field, with an
empty claim, an unrecognized severity, or malformed evidence is **dropped**
by `verdict-filter` before `/verify` ever reads the file — a script decides
what counts as evidence, never a model. `decision` evidence gets a second,
stronger check beyond shape: `decision_id` must resolve to exactly one real
`docs/decisions/<id>-*.md` record and `quote` must appear verbatim in it
(`core/scripts/verdict-filter`'s own pass 2) — a decision id is trivially
cheap to fabricate compared to a real `file:line` or a real command's
captured output, so it earns the extra check.

**Cross-repo verdicts (Extension B)**: a falsifier run against a multi-repo
task's diff (`core/agents/falsifier.md`'s "Cross-repo mandate") cites
undeclared coupling using the existing `file_line` kind, no new evidence
kind needed — only `evidence.file` is repo-qualified (`"<repo-name>:
<path>"`, the same convention `## Predicted touch` already uses for
multi-repo plans) so the finding is unambiguous across repos.
`verdict-filter` does not parse or validate that qualification; it only
checks non-emptiness, exactly as it already does for a single-repo
`file_line`.

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

**Multi-repo tasks (Extension B)** use the *same* task ID — generated once,
at the workspace root, per `core/skills/task/SKILL.md` — as the
`Spine-Task:` trailer on every repo's own commit for that task. A task
touching two repos produces two commits (one per repo, in the declared
ship order, `core/skills/ship/SKILL.md`'s staged-ship §), both carrying an
identical `Spine-Task: <task-id>` trailer. This is spine's existing
linkage primitive doing the cross-repo join — `ledger
scan-untracked-ratio` run against any one member repo's own `git log`
still works unmodified, since it only ever inspects that repo's own
commits for the trailer's presence; it does not need to know a commit's
trailer also appears in a sibling repo.
