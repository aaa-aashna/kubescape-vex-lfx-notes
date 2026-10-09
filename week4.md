# Week 4 


---

## 1. Week 4 Objectives

The objective of Week 4 was to extend the external VEX ingestion implementation so it could process different VEX source formats while preserving compatibility with the existing vulnerability scanning architecture.

The work focused on:

- Supporting CSAF advisories alongside OpenVEX documents.
- Introducing explicit source-format selection while retaining OpenVEX as the default.
- Validating external CSAF documents before they enter the matching and application workflow.
- Preserving source-format information throughout fetching and document staging.
- Preparing validated documents for Grype's existing file-based VEX processor.
- Handling malformed CSAF structures and unsupported formats without allowing invalid external data to trigger an uncontrolled panic.
- Adding regression tests for both successful ingestion and invalid input.

The implementation builds on the existing `safefetch`, `vexvalidate`, `vexdoc`, and `vexbatch` packages instead of creating a separate VEX processing architecture.

## 2. Background and Problem Statement

Vulnerability scanners can use Vulnerability Exploitability eXchange (VEX) documents to understand whether a reported vulnerability affects a particular software product.

A VEX document may state that a vulnerability is affected, fixed, not affected, or under investigation. These assertions can help downstream tooling interpret vulnerability findings more accurately.

Kubescape's external VEX ingestion work requires retrieving VEX documents from external sources, validating the content, and passing suitable documents into the existing vulnerability scanning flow.

Before this change, the `vexsource.Source.Fetch` implementation validated documents as OpenVEX. However, external security advisories can also be supplied in CSAF format.

Treating every external document as OpenVEX creates a format-compatibility problem. CSAF documents have a different structure and require their own validation path. An unsupported or malformed document must be rejected rather than being treated as a valid VEX document.

The Week 4 implementation addresses this problem by making source-format selection explicit and validating each supported format using its appropriate validation mechanism.

## 3. Scope of the Implementation

The primary implementation changes were made in four files:

| File | Responsibility |
|---|---|
| `internal/vexsource/doc.go` | Documents the updated package behavior and format-selection contract. |
| `internal/vexsource/vexsource.go` | Adds source-format selection, format-aware fetching and validation, format preservation, and batch-document staging. |
| `internal/vexsource/vexsource_test.go` | Adds regression tests for OpenVEX, CSAF, unsupported formats, invalid advisories, and malformed CSAF arrays. |
| `internal/vexvalidate/vexvalidate.go` | Adds CSAF validation using the upstream loader and explicit checks for malformed array entries. |

The implementation also reuses existing functionality from `internal/vexdoc` and `internal/vexbatch`.

## 4. Technical Changes

### 4.1 Explicit VEX Format Selection

The `Source` structure was extended to include a format field.

```go
type Source struct {
    URL    string
    Format vexbatch.Format
}
```

Previously, callers supplied the source URL and the ingestion path validated the response as OpenVEX.

The new structure allows callers to identify the intended format explicitly:

- `openvex` — OpenVEX document.
- `csaf` — CSAF advisory.

If the format is empty, the implementation selects OpenVEX. This preserves backward compatibility for callers that construct `Source` without specifying a format.

The supported formats are represented by the existing `vexbatch.Format` type rather than a new, separate format enumeration.

**Why this matters:** The ingestion layer can identify which validation path to use without changing the behavior of existing callers that rely on OpenVEX as the default.

### 4.2 Format-Aware Fetching

The `Source.Fetch` method now performs format-specific validation after retrieving the external document.

The main processing sequence is:

1. Trim whitespace from the source URL.
2. Reject an empty URL.
3. Reject a nil fetcher.
4. Retrieve the document using the existing safe-fetching component.
5. Resolve the requested format, defaulting to OpenVEX.
6. Run the validator associated with the selected format.
7. Reject unsupported formats.
8. Return a validated document containing the URL, raw data, and selected format.

The implementation uses the existing validation functions:

```go
switch format {
case vexbatch.FormatOpenVEX:
    if err := vexvalidate.Validate(data); err != nil {
        return Document{}, fmt.Errorf(
            "validating OpenVEX source: %w", err,
        )
    }

case vexbatch.FormatCSAF:
    if err := vexvalidate.ValidateCSAF(data); err != nil {
        return Document{}, fmt.Errorf(
            "validating CSAF source: %w", err,
        )
    }

default:
    return Document{}, fmt.Errorf(
        "vexsource: unsupported format %q", format,
    )
}
```

The snippet illustrates the format-dispatch logic; the complete implementation remains in `internal/vexsource/vexsource.go`.

**Why this matters:** OpenVEX and CSAF documents are validated according to their respective structures. Unsupported formats produce explicit errors instead of silently entering the processing pipeline.

### 4.3 CSAF Validation with the Upstream Loader

A new `ValidateCSAF` function was added to `internal/vexvalidate/vexvalidate.go`.

```go
func ValidateCSAF(data []byte) (err error) {
    if err := rejectMalformedCSAFArrays(data); err != nil {
        return err
    }

    path, cleanup, err := vexdoc.WriteToTempFile(data)
    if err != nil {
        return fmt.Errorf(
            "vexvalidate: staging CSAF document: %w", err,
        )
    }
    defer cleanup()

    defer func() {
        if recovered := recover(); recovered != nil {
            err = fmt.Errorf(
                "vexvalidate: invalid CSAF document: upstream validator panicked: %v",
                recovered,
            )
        }
    }()

    if _, loadErr := csaf.LoadAdvisory(path); loadErr != nil {
        return fmt.Errorf(
            "vexvalidate: invalid CSAF document: %w", loadErr,
        )
    }

    return nil
}
```

The implementation uses `github.com/gocsaf/csaf/v3/csaf`, the upstream CSAF library already used by the repository's CSAF-related functionality.

The validation process has three main parts:

1. **Structural pre-validation:** Inspect known malformed array cases before invoking the upstream loader.
2. **Temporary-file staging:** Write the fetched bytes to a temporary file because the CSAF loader accepts a file path.
3. **Upstream validation:** Load the staged advisory and propagate validation failures as errors.

A deferred recovery handler also converts a panic from the upstream validator into a validation error. This is a defensive boundary around untrusted external input, not a replacement for proper input validation.

The temporary-file cleanup function is deferred immediately after staging succeeds, so the staged document is removed when validation completes.

**Why this matters:** External advisory data is validated before it is returned to downstream processing, and malformed input is handled as an error instead of an uncontrolled panic.

### 4.4 Rejecting Malformed CSAF Arrays

During review, a concrete edge case was identified in the CSAF validation path.

The upstream loader could panic when certain arrays contained `null` entries. Relying on the upstream validator alone was therefore insufficient for those malformed inputs.

The implementation added `rejectMalformedCSAFArrays`, which checks:

- `document.tracking.revision_history`
- The top-level `vulnerabilities` array

The helper first parses the input JSON into a minimal structure containing these fields. It then rejects any array entry represented by JSON `null`.

For example, an advisory containing:

```json
{
  "vulnerabilities": [null]
}
```

must not pass structural pre-validation.

The `vulnerabilities` array is a top-level property of the CSAF advisory, alongside `document`; it is not nested inside `document`.

This distinction became important during review. The original implementation and regression test incorrectly placed the vulnerability array inside the nested document object. The code and test were corrected to inspect the actual top-level field.

Regression tests now cover both malformed revision-history entries and malformed vulnerability entries. They verify that the fetch operation returns an error and does not panic.

**Why this matters:** The validator explicitly rejects known malformed structures before they reach an upstream loader that may not safely handle them.

### 4.5 Preserving the Format in the Fetched Document

The `Document` structure now retains the selected format:

```go
type Document struct {
    URL    string
    Data   []byte
    Format vexbatch.Format
}
```

After successful retrieval and validation, the returned document contains:

- The source URL.
- The original fetched document bytes.
- The validated source format.

Previously, a fetched document carried its URL and data but not an explicit format field.

**Why this matters:** Downstream code can distinguish OpenVEX from CSAF without inferring the format from the document content or losing the source-format selection during the fetch lifecycle.

### 4.6 Staging Documents for the Existing Grype Processor

The implementation adds `Document.BatchDocument()` to convert a fetched document into the existing `vexbatch.Document` representation.

```go
func (d Document) BatchDocument() (
    vexbatch.Document,
    func(),
    error,
) {
    format := d.Format
    if format == "" {
        format = vexbatch.FormatOpenVEX
    }

    path, cleanup, err := vexdoc.WriteToTempFile(d.Data)
    if err != nil {
        return vexbatch.Document{}, func() {}, err
    }

    return vexbatch.Document{
        Format: format,
        Path:   path,
    }, cleanup, nil
}
```

The method uses the existing `vexdoc.WriteToTempFile` helper to stage the document on disk. It then returns a batch document containing the format and temporary-file path, alongside a cleanup function.

The underlying temporary-file utility already provides important safety properties:

- Unique, unpredictable filenames.
- File creation with owner-only read/write permissions (`0600`).
- Creation inside the operating system's temporary directory.
- Cleanup support for removing the file after use.
- Rejection of empty document data.

The method preserves the original format and falls back to OpenVEX for older callers that create a document without a format.

**Why this matters:** Grype's existing VEX processor consumes file paths. The new method bridges fetched in-memory documents to that interface without adding another document-processing engine.

**Important scope distinction:** This change prepares documents for the existing batch application path. It does not, by itself, implement the complete external-source controller or independently prove that every fetched advisory is applied to a matching vulnerability finding in a full end-to-end scan.

### 4.7 Reusing the Existing Processing Architecture

The existing `vexbatch` package supports OpenVEX and CSAF formats and applies groups of documents through Grype's VEX processor.

It accepts staged documents with a format and path, validates supported format values, groups documents by format, and uses `vex.NewProcessor` and `ApplyVEX` for application.

The Week 4 source-ingestion change builds on this design.

Rather than creating a new matching engine or implementing a second VEX application mechanism, it prepares validated external documents for the established batch-processing interface.

This keeps fetching, validation, staging, and VEX application as distinct responsibilities.

The resulting architectural flow is:

```text
External VEX Source
        |
        v
Safe Fetching
(internal/safefetch)
        |
        v
Format Selection
(OpenVEX / CSAF)
        |
        v
Format-Specific Validation
(internal/vexvalidate)
        |
        v
Validated Document
(URL + Bytes + Format)
        |
        v
Temporary-File Staging
(internal/vexdoc)
        |
        v
Batch Document
(Format + Path)
        |
        v
Existing VEX Application
(internal/vexbatch / Grype)
```

The flow describes the implemented ingestion-to-staging contract and its intended downstream handoff. Wiring every step into the full external VEX controller and scan lifecycle remains part of the broader project scope.

## 5. Test Coverage Added

The changes extend `internal/vexsource/vexsource_test.go` with tests for successful ingestion, validation failures, and malformed input.

| Test | What it verifies |
|---|---|
| `TestSourceFetch` | Default OpenVEX ingestion, returned format, and batch-document staging. |
| `TestSourceFetch_CSAF` | Successful CSAF ingestion using the existing Red Hat advisory fixture for `CVE-2024-3094`, with format preservation and staging. |
| `TestSourceFetch_UnsupportedFormat` | An unsupported format such as `xml` is rejected with an explicit error. |
| `TestSourceFetch_InvalidCSAF` | An invalid CSAF document is rejected. |
| `TestSourceFetch_MalformedCSAFRevisionHistory` | A `null` entry in `document.tracking.revision_history` is rejected without a panic. |
| `TestSourceFetch_MalformedCSAFVulnerabilities` | A `null` entry in the top-level `vulnerabilities` array is rejected without a panic. |

The existing OpenVEX validation tests continue to exercise malformed JSON, missing or invalid context, invalid statements, missing products, and insufficiently identified products.

The CSAF tests use the repository's existing Red Hat fixture rather than relying exclusively on a synthetic happy-path example.

This helps establish that the ingestion path can validate and stage a real advisory while still rejecting explicitly malformed variants of that advisory.

## 6. Review Feedback and Corrections

The review process identified an important robustness issue and helped tighten the implementation.

### Initial issue: upstream validator panic

A reviewer demonstrated that malformed CSAF array entries could cause the upstream validator to panic instead of returning a normal validation error.

**Correction:** Added malformed-array pre-validation and a recovery handler around the upstream loader, with regression tests that verify rejection without panicking.

### Follow-up issue: incorrect vulnerability-array location

A subsequent review found that the initial pre-validation structure and test mutation placed `vulnerabilities` under the nested `document` property. The actual CSAF field is at the advisory root.

**Correction:** Moved the field into the top-level validation structure and changed the regression test to mutate the root `vulnerabilities` array. This ensures the test exercises the real malformed-input case.

### Review outcome

The final review approved commit `81ed6369357f5cd07c131fa14d07205890c11043`. The approval reported that the corrected implementation, local tests for the relevant VEX packages, and CI workflows passed.

The pull request was subsequently merged as commit `bec07e1e6e00fdcf7842325fcc68fe93bc01755d`.

## 7. Merge Summary

| Attribute | Details |
|---|---|
| Pull request | [#1157](https://github.com/kubescape/kubevuln/pull/1157) |
| Title | `feat(vex): support CSAF external source ingestion` |
| Target branch | `main` |
| Feature branch | `feat/vex-ingestion-week4` |
| Files changed | 4 |
| Additions | 219 |
| Deletions | 12 |
| Commits | 7 |
| Merge commit | `bec07e1e6e00fdcf7842325fcc68fe93bc01755d` |
| Merged on | October 5, 2026 |

These are the pull request's recorded metadata values; the additions and deletions are not a measure of test coverage or the amount of functionality completed.

## 8. Key Technical Learnings

### Format-specific validation is necessary

A common retrieval mechanism does not mean all fetched documents share the same schema. The format determines the correct validation procedure.

### Untrusted input requires defensive error handling

External security advisories should be treated as untrusted input. Validation should reject malformed structures, and known failure modes in upstream libraries should not become uncontrolled panics.

### Internal representations preserve information across layers

Carrying the source format alongside the document bytes makes the handoff explicit. This avoids depending on implicit format assumptions in later processing.

### Existing abstractions should be reused

`vexdoc` provides safe temporary-file staging, and `vexbatch` supplies the existing batch-application interface. Reusing these packages keeps responsibilities modular and avoids duplicate processing code.

### Regression tests should reproduce the actual input structure

A test can pass while missing a defect if it modifies the wrong JSON field. The CSAF vulnerability-array review demonstrated why tests must reflect the real advisory schema, not merely create a superficially similar malformed object.

## 9. Week 4 Outcome

Week 4 delivered a format-aware external VEX ingestion extension that supports OpenVEX and CSAF.

The implementation introduced explicit format selection, preserved the format through the fetch lifecycle, validated CSAF advisories using the upstream loader, handled known malformed-array cases, and prepared validated documents for the existing Grype-based batch processing path.

It also improved the reliability of the ingestion boundary by adding regression coverage for malformed CSAF data and addressing review feedback before the pull request was merged.

This work establishes an important foundation for the larger external VEX ingestion objective: retrieving external security advisories, validating them safely, and preparing them for use in vulnerability scanning.

## 10. Remaining Work

The merged pull request completes the Week 4 scope described above, but it does not establish that every requirement in the full external VEX ingestion proposal is finished.

The next steps are:

1. Continue the integration of external VEX sources with the complete scan and result-handling lifecycle.
2. Strengthen coverage of source unavailability, HTTP and fetch failures, malformed payloads, and unsupported formats.
3. Expand integration testing across fetching, validation, staging, matching, and application.
4. Verify that external VEX failures are isolated appropriately and do not silently corrupt or invalidate normal scan results.
5. Continue addressing any remaining review feedback and project requirements.

## 11. References

- [PR #1157 — Support CSAF external source ingestion](https://github.com/kubescape/kubevuln/pull/1157)
- [Merged implementation](https://github.com/kubescape/kubevuln/commit/bec07e1e6e00fdcf7842325fcc68fe93bc01755d)
- [External VEX source implementation](https://github.com/kubescape/kubevuln/blob/main/internal/vexsource/vexsource.go)
- [External VEX source tests](https://github.com/kubescape/kubevuln/blob/main/internal/vexsource/vexsource_test.go)
- [VEX validation implementation](https://github.com/kubescape/kubevuln/blob/main/internal/vexvalidate/vexvalidate.go)
- [VEX batch application](https://github.com/kubescape/kubevuln/blob/main/internal/vexbatch/vexbatch.go)
- [Safe VEX temporary-file staging](https://github.com/kubescape/kubevuln/blob/main/internal/vexdoc/vexdoc.go)

---

**Week 4 status: Completed and merged.**

**Primary deliverable:** Format-aware OpenVEX and CSAF source ingestion, CSAF validation, malformed-input protection, document-format preservation, and batch-document staging for the existing VEX application architecture.
