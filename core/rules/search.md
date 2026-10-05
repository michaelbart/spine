# Search discipline

When using Grep for **discovery** — mapping what exists, finding all usages,
understanding scope — pause before drawing conclusions and use `AskUserQuestion`
to offer a choice between the targeted grep results and a broader search via the
`Explore` agent or a wider grep strategy.

Verification greps (confirming a specific known thing exists at a known location)
need no check.

**The failure mode:** treating a targeted grep as exhaustive evidence. A grep
that looks in the right directory with the right pattern still misses files in
unexpected locations, case variants, indirect references, or patterns you didn't
think to search for. The cost of a missed file isn't felt until a confident
conclusion turns out to be wrong.
