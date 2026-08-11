<!--
SYNTHETIC — constructed for spine/work/.build/pr-patch-phase-B-handoff.md
to prove the empty-case line renders correctly. Not a real task; no real
project backs it. Minimal by design: just enough of each artifact to
exercise all four unions and confirm every one computes to empty.
-->

# Plan: `synthetic-empty-case-demo` — add a `formatCurrency` helper

<!-- MACHINE: header -->
task: synthetic-empty-case-demo   class: 1   owner: synthetic   milestone: none
grounding: research `n/a` (synthetic demo — no real research.md)
<!-- /MACHINE -->

## The gist

Adds a small, pure `formatCurrency(cents: number): string` helper and its
unit tests. No existing code changes.

## What could go wrong

Little — new, isolated, pure function with no callers yet.

## What I'll decide alone vs. stop and ask

**I'll just do** (`decide-alone`):
- Formatting details, exact test cases.

**I'll do and note** (`record-and-proceed`):
- n/a

**I'll stop and ask before** (`halt`):
- Anything touching a protected path.

## Steps

1. **`src/lib/format-currency.ts`** — add the helper. — **acceptance:**
   `npm test` passes.

<!-- MACHINE: predicted-touch -->
## Predicted touch

- src/lib/format-currency.ts — new helper
- src/lib/format-currency.test.ts — new unit tests
<!-- /MACHINE -->

## How we'll know it worked

`npm test` demonstrates the helper's behavior directly.
