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

## Important Matching Logic

External VEX application depends on how `vexbatch.Apply` matches `s.Source` against the VEX products.

The existing matching logic should be reused rather than introducing custom product/identity matching in the Grype adapter.

Before adding further behavior, the identity guard needs to be confirmed.

## Next Changes

Add two regression tests to verify that VEX is not incorrectly applied when product identity is insufficient or mismatched:

1. Missing identity → VEX does not suppress the finding.
2. Mismatched identity → VEX does not suppress the finding.

## Flow

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

## Key Week 3 Result

External VEX is now integrated at the correct point in the Grype scanning pipeline without replacing the existing VEX matching or downstream SecurityException/storage behavior.
