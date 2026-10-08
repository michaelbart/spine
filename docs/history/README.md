# History

Records of how spine got here. Not current documentation: files here describe the
repo as it was when they were written and are exempt from `repo-lint`'s citation check.

- `audit-2026-08-22*.md`: six adversarial audit rounds over one day. They found the
  hook bypasses (`../` traversal, symlinks, root aliases, Unicode, a runaway
  path-normalizer fork loop) that the fuzz suite and hook regression tests in
  `core/scripts/core-selftest` now guard. The lesson that carried forward: a test
  that passes after a fix proves nothing unless it also fails without the fix.
  Round-numbered audits were replaced by continuous checks (the fuzz suite,
  `core/scripts/repo-lint`, the complexity budget).
- `proposal-committed-hooks.md`: a proposal to commit `.claude/hooks/` instead of
  symlinking it. Moot since team and cross-repo support (the main reason to want
  it) were removed on 2026-10-08.
