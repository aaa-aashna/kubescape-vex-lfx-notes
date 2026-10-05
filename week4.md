#week 4

We should work toward a PR that meaningfully implements:
external VEX source → fetch → parse → validate/normalize into internal document representation → hand off to existing VEX application machinery
while keeping failures in external VEX handling from silently corrupting the normal scan path. Your proposal specifically calls out unavailable, malformed, and unsupported VEX as cases to handle.     Kubescape_Proposal (1)
And there is a useful clue in#1108 itself: the repository already has vexsource, safefetch, vexvalidate, and vexbatch, and the merged PR reused the existing batch application rather than inventing another VEX processor.
Next, we should inspect the current main state of those VEX packages and the related issue/design history  before writing Week 4 code. That will tell us exactly what infrastructure already exists and what the PR should add rather than duplicate.

Week 3–4 is supposed to cover VEX fetching, parsing, internal representation, basic malformed/unavailable handling, and an initial fetch/parse prototype independent of the normal scan.


## Work Completed

- Added explicit VEX source format selection, with **OpenVEX retained as the backward-compatible default**.
- Extended external VEX ingestion to support **CSAF advisories** in addition to OpenVEX.
- Added CSAF validation using the upstream `gocsaf/csaf/v3` loader before documents reach the matching/application layer.
- Preserved the selected VEX format throughout the fetched document lifecycle.
- Added `BatchDocument` to safely stage validated OpenVEX and CSAF documents for Grype's existing file-based VEX processor.
- Reused the existing VEX application machinery instead of introducing a parallel VEX processor.
- Added test coverage for:
  - default OpenVEX ingestion and staging
  - real Red Hat CSAF/VEX ingestion using the existing `CVE-2024-3094` fixture
  - unsupported formats
  - invalid CSAF documents
- Validated the implementation against the existing external VEX processing flow and maintained compatibility with the normal vulnerability scanning architecture.

