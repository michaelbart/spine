# Touchpoint blocks: print the text, not the markers

Skills wrap the wording they ask you to show the human in
`<!-- touchpoint:start ... -->` and `<!-- touchpoint:end -->` comment lines.
Those lines exist only so `touchpoint-lint` can find the block. Never print
them. Show the human only the `>` lines between them, with the placeholders
filled in.
