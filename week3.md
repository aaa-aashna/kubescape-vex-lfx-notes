# Week 3

## Objective

Determine where external VEX can suppress `not_affected` / `fixed` vulnerability findings after Grype generates the `CVEManifest`, while preserving the existing SecurityException and storage flow.

## Changes Made

- Added external VEX application support to the Grype adapter.
- Added `ScanSBOMWithVEX` to allow an SBOM scan to explicitly receive external VEX documents.
- Applied external VEX after Grype vulnerability matching and before final scan results are compiled.
- Reused the existing `vexbatch.Apply` logic for applying VEX across supported document formats.
- Added a Docker fixture integration test covering OpenVEX suppression during a Grype scan.
- Verified that `CVE-2023-1255` from the OpenVEX fixture is removed from the final vulnerability matches.

## Matching Logic

External VEX application depends on how `vexbatch.Apply` matches `s.Source` against the VEX products.

The existing matching logic should be reused rather than introducing custom product/identity matching in the Grype adapter.

Before adding further behavior, the identity guard needs to be confirmed.

## Next Changes

Add two regression tests to verify that VEX is not incorrectly applied when product identity is insufficient or mismatched:

1. Missing identity → VEX does not suppress the finding.
2. Mismatched identity → VEX does not suppress the finding.

## Flow
```
SBOM
↓
Grype Scan
↓
CVEManifest / vulnerability matches
↓
Apply external VEX using `vexbatch.Apply`
↓
VEX matching + identity validation
↓
Suppressed findings removed
↓
Final scan results
↓
Existing SecurityException / storage flow
```

## Likely Result

External VEX is now integrated at the correct point in the Grype scanning pipeline without replacing the existing VEX matching or downstream SecurityException/storage behavior.

What this PR does
The core problem is:
Kubescape can generate VEX, but when it scans an SBOM with Grype, it previously had no way to consume an external VEX document and use that information to suppress findings.

Your PR connects the external VEX flow to the existing Grype vulnerability-scanning pipeline.
The flow is essentially:
```
SBOM
  │
  ▼
Grype vulnerability scan
  │
  ▼
CVE / vulnerability matches
  │
  ├── External VEX documents
  │        │
  │        ▼
  │   vexbatch.Apply(...)
  │        │
  │        ├── not_affected → IgnoredMatches
  │        └── applicable   → Matches
  │
  ▼
Final CVE Manifest
```

So you're not replacing Grype's vulnerability detection. You're adding an additional decision layer after Grype has identified vulnerabilities.
The important architectural change
The key integration point is inside:
adapters/v1/grype.go

The PR introduces the ability for the SBOM scan to receive VEX documents explicitly:
ScanSBOMWithVEX(...)

Then, after Grype produces its vulnerability matches, your code calls the existing:
vexbatch.Apply(...)

This is important because you're reusing the project's existing VEX application logic instead of implementing OpenVEX matching again inside the Grype adapter.
