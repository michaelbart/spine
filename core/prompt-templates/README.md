<!--
This directory is not part of the file-shape conventions in core/templates/
— those get filled in *inside* a spine session (a skill reads one and
writes the result into a project). Everything here is the opposite
direction: a prompt a human copies out and pastes into some *other*
surface (a Claude chat session, a Claude Design session, a fresh Claude
Code session with no spine context yet) to generate the raw source
material that later becomes one of those core/templates/ files.

Nothing in spine reads this directory automatically — no skill loads a
file from here, no script parses one. Purely a human convenience so
"what do I actually ask Claude Design for" doesn't have to be re-invented
per project.

Each prompt below produces a real file, saved at a real path, that the
rest of spine already knows how to consume:

| Prompt | Produces | Consumed by |
|---|---|---|
| `product-spec.md` | `docs/product-spec.md` | `/design` (auto-detected, `--handoff`), `/wayfinder` |
| `feature-handoff.md` | `docs/features/<slug>-handoff.md` | `/design --handoff`, `/wayfinder`, `/roadmap`, per its own Routing section |
| `ui-handoff.md` | `docs/ui/{tokens.json, components.md, screens/*.json, screenshots/*.png, handoff.md}` | `core/rules/ui-design-system.md`, `ui-conformance` capability |

If you use a different tool than the one a prompt names (a different chat
model, a different design tool), the prompt's actual instructions — the
target file shape and section-by-section requirements — still apply; only
the "paste this into X" framing is tool-specific.
