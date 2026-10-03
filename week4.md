#week 4

We should work toward a PR that meaningfully implements:
external VEX source → fetch → parse → validate/normalize into internal document representation → hand off to existing VEX application machinery
while keeping failures in external VEX handling from silently corrupting the normal scan path. Your proposal specifically calls out unavailable, malformed, and unsupported VEX as cases to handle.     Kubescape_Proposal (1)
And there is a useful clue in#1108 itself: the repository already has vexsource, safefetch, vexvalidate, and vexbatch, and the merged PR reused the existing batch application rather than inventing another VEX processor.
Next, we should inspect the current main state of those VEX packages and the related issue/design history  before writing Week 4 code. That will tell us exactly what infrastructure already exists and what the PR should add rather than duplicate.
