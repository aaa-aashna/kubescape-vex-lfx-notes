## Week 2 

### Goal for this part of Week 2

Week 2 was to start connecting the Week 1 concepts with the actual kubevuln codebase. I started exploring the repository structure and locating the files related to vulnerability scanning, VEX handling and the existing Kubernetes resources.

### Main Points 

- Started exploring the kubevuln repository structure.
- Located the `internal` packages and checked how the project is organized.
- Started looking for the existing VEX-related types and storage models.
- Investigated the `OpenVulnerabilityExchangeContainer` concept and where it may be used.
- Started tracing how vulnerability findings are created and passed through the scanning flow.
- Read about the existing `WatchHandler` and `CooldownQueue` components mentioned in the proposal.
- Started understanding how these components may help with periodically fetching external VEX feeds.
- Reviewed the difference between:
  - fetching external VEX documents,
  - storing the documents,
  - matching VEX statements with vulnerability findings,
  - suppressing findings.
- Started identifying which parts should be implemented first instead of trying to build the complete controller at once.

### Understanding of the implementation flow

```text
VEXSource CRD
      |
      v
Fetch external VEX feed
      |
      v
Validate and parse document
      |
      v
Store VEX information
      |
      v
Match with vulnerability findings
      |
      v
Suppress matching findings
```

Noted down this implementation flow , so that i have a roadmap in my mind about things to be done
adding files 
