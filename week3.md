#week 3




We already know ScanService receives:
cve, err := s.cveScanner.ScanSBOM(ctx, sbom)

We're trying to answer:
Once Grype has produced CVEManifest, where can external VEX suppress not_affected / fixed findings without breaking the existing SecurityException and storage flow?

- Added external VEX application support to the Grype adapter.
- Added `ScanSBOMWithVEX` to allow an SBOM scan to receive external VEX documents explicitly.
- Apply external VEX documents after Grype vulnerability matching and before the final scan results are compiled.
- Reuse the existing `vexbatch.Apply` logic for applying VEX across supported document formats.
- Added a Docker fixture integration test covering OpenVEX suppression during a Grype scan.
- Verified that `CVE-2023-1255` from the OpenVEX fixture is removed from the final vulnerability matches.


That part depends on how vexbatch.Apply matches s.Source against the VEX products. We should not invent our own matching logic if vexbatch already has it.
So after confirming the guard, the next change will be the two regression tests:
missing identity → VEX does not suppress
mismatched identity → VEX does not suppres
