| Component                              | Main question it answers                                    |
| -------------------------------------- | ----------------------------------------------------------- |
| **Image**                              | What are we scanning?                                       |
| **Syft**                               | What software is inside it?                                 |
| **SBOM**                               | What is the software inventory?                             |
| **Grype**                              | Which components have known vulnerabilities?                |
| **Vulnerability Matches**              | What exact CVE/package matches were found?                  |
| **SecurityException**                  | Are there security-specific exceptions?                     |
| **External VEX**                       | Does external VEX provide vulnerability status/context?     |
| **Filtering/Suppression**              | Which findings should be suppressed/treated differently?    |
| **IgnoredMatches**                     | Which matches were suppressed?                              |
| **VulnerabilityManifest**              | How does Kubescape structure the vulnerability information? |
| **VEX**                                | What is the vulnerability status and justification?         |
| **OpenVulnerabilityExchangeContainer** | How is that VEX information packaged/represented?           |


we will be collecting all this info and then start working on it...


the expected structure is something like this :


OpenVulnerabilityExchangeContainer
│
└── Spec: VEX document
    │
    └── Statements
        │
        ├── Vulnerability
        │    └── ID → CVE
        │
        ├── Products
        │    └── Subcomponents
        │         └── ID → package/product identity
        │
        ├── Status
        │    └── not_affected / affected / fixed
        │
        ├── Justification
        │
        └── ImpactStatement
        