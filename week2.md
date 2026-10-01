## Week 2 Plan

I will move from the high-level concepts to the actual code and start working on the VEX data flow and ingestion components.

### Planned Work

- Locate and read the actual `OpenVulnerabilityExchangeContainer` definition.
- Find how Kubescape currently creates and persists its own VEX.
- Trace the vulnerability manifest generation path.
- Identify where the external VEX join can be added.
- Study `WatchHandler` and `CooldownQueue` and see how they can be reused by the controller.
- Read the existing SecurityException matching logic.
- Start defining the fields and behavior of the `VEXSource` CRD.
- Choose a small first task that can be implemented and tested independently.

### VEX Document Structure and Validation

I am planning to add a `document.go` file that will focus solely on the actual VEX document data structure and its validation logic.

Alongside it, I have added `document_test.go` to verify the document structure and ensure that the validation logic works correctly across the relevant test cases.

The implementation also uses the `strings` package for validating string fields.

For example:

```
strings.TrimSpace(document.ID) == ""

This checks whether the string is empty or contains only whitespace.
This catches both:
""

and:
"   "

VEX Status Validation
The document validation also handles the different VEX status values:
case StatusNotAffected,
    StatusAffected,
    StatusFixed,
    StatusUnderInvestigation:
    return nil
```
This ensures that the supported VEX statuses are explicitly handled during validation.
Initial VEX Ingestion Changes
The initial VEX ingestion changes are now in place, and the relevant package-level tests are passing.
The full repository test suite is currently taking longer than expected, so I am validating the remaining components separately and will continue pushing incremental updates as the implementation progresses.
Existing Implementation Cleanup
One previously added file was removed because the PR was superseded by #880. The duplicate internal/vex implementation is therefore no longer required and should be dropped.
External VEX Ingestion Flow
The current flow being implemented is:
```
External VEX URL
        ↓
vexsource.Fetch()
        ↓
vexbatch.Document
        ↓
vexbatch.Apply()
        ↓
Grype matches
        ↓
kubevuln CVE results
```
The new flow is being traced through the existing vulnerability processing path, starting from the files that currently handle the relevant stages and then updating the required components according to the new architecture.
Updated Processing Flow
```
The current flow can be represented as:
SBOM
 ↓
Grype scanner
 ↓
match.Matches
 ↓
vexsource.Fetch()
 ↓
vexbatch.Apply()
 ↓
filtered match.Matches
 ↓
Grype → kubevuln conversion
```
Grype is used as the starting point for tracing how VEX documents interact with vulnerability matches. Based on this flow, additional files and components can be updated where the external VEX ingestion needs to be integrated.

So grype is usually helped to trace documents and then we add additional files 

Core Idea
The vexsource package handles retrieval and validation, while persistence and Grype-specific file handling remain outside the package.
This keeps the responsibility of each component clearly separated and provides a smaller, independently testable boundary for external VEX ingestion.

External VEX Ingestion — Changes Made
- Refactored internal/vexsource so that Source.Fetch() is responsible only for fetching and validating an external VEX document.
- Removed temporary-file creation from Source.Fetch(). It now returns a validated Document containing the source URL and raw VEX bytes.
- Added an Ingestor abstraction that uses the existing safefetch implementation to retrieve external VEX sources.
- Reused the existing vexvalidate package to validate the fetched VEX data instead of introducing new validation logic.
- Added focused tests covering:
  - successful VEX fetching
  - empty source URL
  - nil fetcher
  - fetch failures
  - invalid VEX documents
  - successful Ingestor ingestion
- Added package documentation explaining the intended boundary: the future VEXSource controller from #387 will handle persistence and temporary-file preparation for Grype.
- Updated docs/VEX.md to document the role of the new internal/vexsource package.
- Kept the implementation dependency-free by using the existing safefetch and vexvalidate primitives.
Architecture After the Change

```
External VEX URL
        ↓
    safefetch
        ↓
   Source.Fetch()
        ↓
   vexvalidate
        ↓
 Validated VEX bytes
        ↓
 VEXSource controller (#387)
        ↓
Persistence / temp-file staging
        ↓
      Grype
```

Core Idea
The vexsource package handles retrieval and validation, while persistence and Grype-specific file handling remain outside the package.
This keeps the responsibility of each component clearly separated and provides a smaller, independently testable boundary for external VEX ingestion.

