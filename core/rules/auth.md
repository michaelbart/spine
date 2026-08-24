---
paths:
  - "**/auth/**"
  - "**/authn/**"
  - "**/authz/**"
---

# Auth discipline

This rule loads whenever a file under an `auth/`-, `authn/`-, or
`authz/`-named directory is read. Like `core/rules/migrations.md` and
`core/rules/contracts.md`, it says nothing about a specific auth
technology, session model, or framework — that's the `security` adversary
agent's job (`core/agents/security.md`, its preloaded `security-checklist`
skill) and this project's own `.spine/adapters/`. This rule is the
narrower, always-applicable discipline around *which spine mechanism
already governs this*, not a security technique guide.

**Ground on the adopted auth-model decision, if one exists.** `/design`'s
six foundational categories include `auth-model` — before changing
anything under a matching path, check `docs/decisions/D-*.md` for a
record carrying `Category: auth-model` and read it in full (not just its
`docs/decisions/INDEX.md` row). A change that contradicts what was
decided isn't automatically wrong, but it's a real deviation
(`core/skills/task/SKILL.md` §4's `halt`-tier list already names "auth
logic" explicitly) — resolve it the way any other halt-tier deviation
resolves, don't silently drift from the decision. If no `auth-model`
decision exists yet (a project that skipped `/design`, or deferred the
category via `docs/decisions/DEFERRED.md`), say so plainly rather than
inventing grounding that isn't there — same discipline the researcher
agent already applies to a missing `docs/charter.md`.

**This is very likely protected-path territory.** If `.spine/
protected-paths.conf` doesn't already list a glob covering the path just
read, that's worth flagging as a probable calibration gap in this
project's own conf, not silently accepted — `core/skills/adopt/SKILL.md`'s
own `surveyor` agent already names "auth/authorization" as a first-class
protected-path candidate at calibration time (alongside public API surface
and payment/PII handling); a project where an auth path was never actually
added to the conf has a real gap between what calibration intended and
what's mechanically enforced.

**Disclosed scope limit — this rule does not cover secrets, credentials,
or PII handling**, even though `core/skills/adopt/SKILL.md`'s own
`surveyor` agent names all three as peer protected-path categories
alongside auth. Unlike `migrations/` or `auth/`, there is no comparably
reliable directory-naming convention for "code that handles a secret or a
person's PII" to scope a `paths:` glob against — a credential can be
touched from anywhere a request is authenticated, a database row is read,
or a log line is written. Inventing an unreliable glob here would be
worse than having no auto-loaded rule at all: it would create the
impression of coverage this rule can't actually deliver. The two real,
already-existing mechanisms that do cover this today are `secret-scan`
(content-based, runs in every floor invocation, not path-scoped) and
whatever paths this project's own calibration explicitly added to
`.spine/protected-paths.conf` — see `docs/tradeoffs.md`'s Known limits for
this gap named the same way every other conceded-not-bug limit there is.
