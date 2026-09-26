VEXSource
   │
   ▼
Source.Fetch()
   │
   ▼
validated VEX bytes
   │
   ▼
persist external VEX
   │
   ▼
VEX available to scanning pipeline
   │
   ▼
match VEX against vulnerabilities
   │
   ▼
suppress / retain findings



We already know ScanService receives:
cve, err := s.cveScanner.ScanSBOM(ctx, sbom)

We're trying to answer:
Once Grype has produced CVEManifest, where can external VEX suppress not_affected / fixed findings without breaking the existing SecurityException and storage flow?