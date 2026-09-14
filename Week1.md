# Week 1 — Kubescape VEX Project

## Objective

The main objective of Week 1 was to understand the existing Kubescape vulnerability-scanning and VEX flow before making implementation changes for the **External VEX Ingestion** project.

The focus was on understanding the existing pipeline, the main vulnerability/VEX data structures, and where external VEX can eventually be integrated.

---

## 1. Vulnerability Scanning Pipeline

The vulnerability flow can be understood as:

```text
Container Image
      |
      v
    Syft
      |
      v
     SBOM
      |
      v
    Grype
      |
      v
Vulnerability Matches
      |
      +----------------------+
      |                      |
      v                      v
SecurityException      External VEX
      |                      |
      +----------+-----------+
                 |
                 v
       Filtering / Suppression
                 |
                 v
       Vulnerability Manifest
```

### Image

The container image is the object being scanned. The image contains the operating-system packages, libraries, binaries, and other software components that may have known vulnerabilities.

### Syft

Syft is used to generate an **SBOM (Software Bill of Materials)** from the container image. The SBOM provides an inventory of the components present in the image.

### SBOM

The SBOM is the inventory that tells the vulnerability scanner what packages/components exist in the image. Vulnerability matching is performed against these components.

### Grype

Grype is the vulnerability scanner used in the pipeline. It takes the software/component inventory and matches components against known vulnerability information to produce vulnerability findings.

### Vulnerability Matches

A vulnerability match connects a vulnerable component/package with a vulnerability identifier such as a CVE. A finding therefore represents that a particular component in an image matches a known vulnerability.

### SecurityException

SecurityException represents an exception that can prevent a vulnerability finding from being treated as an active finding. Its matching model is relevant to this project because an external vendor VEX statement can provide a similar outcome, especially for `not_affected` cases.

### External VEX

External VEX is the main focus of this LFX project. Vendors such as Red Hat or Chainguard can publish statements describing the status of vulnerabilities affecting their products/images.

Kubescape currently produces VEX from its own scan results, but the selected project is intended to add **read-only ingestion of third-party VEX** so that vendor statements can be applied to Kubescape vulnerability findings.

### Filtering / Suppression

After vulnerability findings are produced, Kubescape needs to determine which findings should remain visible and which can be suppressed or treated differently based on applicable security exceptions and VEX information.

For this project, external `not_affected` and `fixed` statements are expected to suppress matching findings before they are presented to the user.

---

## 2. VEX Concepts

**VEX (Vulnerability Exploitability eXchange)** provides machine-readable information about the status of a vulnerability for a particular product/component.

The important statuses for this project are:

- **`not_affected`** — the vulnerability does not affect the referenced product/component in the stated context.
- **`fixed`** — the vulnerability has been fixed in the referenced product/component/version.
- **`affected`** — the vulnerability does affect the referenced product/component.

For external VEX ingestion, we are primarily interested in vendor statements that can justify suppressing a scan finding.

---

## 3. OpenVEX

OpenVEX is an open, machine-readable format for communicating VEX statements.

A simplified representation is:

```text
VEX Document
   |
   +-- Vulnerability
   |      └── CVE / vulnerability ID
   |
   +-- Product
   |      └── Product/component identity
   |
   +-- Status
   |      └── not_affected / affected / fixed
   |
   +-- Justification
   |
   └-- Impact Statement
```

The important idea is that a VEX statement connects a **vulnerability** to a **specific product/component** and describes the vulnerability status for that product/component.

---

## 4. OpenVulnerabilityExchangeContainer

Kubescape/kubevuln already has a storage shape for VEX documents: `OpenVulnerabilityExchangeContainer`.

Conceptually, the structure can be thought of as:

```text
OpenVulnerabilityExchangeContainer
└── Spec
    └── VEX document
        └── Statements
            ├── Vulnerability
            │    └── ID (for example, CVE)
            ├── Products / Subcomponents
            │    └── Product/component identity
            ├── Status
            │    └── not_affected / affected / fixed
            ├── Justification
            └── Impact Statement
```

This existing storage model is important because the project proposal says the external VEX controller can persist fetched documents using the existing `OpenVulnerabilityExchangeContainer` shape, or a clearly scoped sibling kind if necessary.

---

## 5. Why External VEX Is Needed

A large number of container vulnerabilities can originate from distribution base images such as nginx, Alpine, Python, or RHEL UBI.

Vendors already perform their own vulnerability triage and may publish VEX/CSAF information stating that a particular CVE is `not_affected` or has been `fixed` for their products.

The problem is that Kubescape's current vulnerability pipeline does not consume these external vendor statements.

Therefore, the scanner can report vulnerabilities that an upstream vendor has already determined are not applicable or already fixed.

The selected LFX project aims to close this gap by adding external VEX ingestion and applying the relevant vendor statements to Kubescape scan results.

---

## 6. Connection to the Selected LFX Project

The selected project is **External VEX Ingestion for Kubescape Vulnerability Scanning**.

The major pieces identified from the proposal are:

1. A namespaced `VEXSource` CRD to declare an external feed.
2. A controller to fetch, validate, refresh, and persist external VEX documents.
3. A join step in the vulnerability scan pipeline to match external VEX statements to scan findings.
4. Suppression of matching `not_affected` / `fixed` findings.
5. Preservation of provenance so we know which document/statement caused a suppression.
6. Deduplication and normalization across feeds.
7. End-to-end testing against at least one real vendor feed.

---

## 7. Week 1 Outcome

By the end of Week 1, the main goal was to establish the vocabulary and high-level architecture needed for implementation.

The current understanding is:

```text
Image
  -> SBOM generation
  -> vulnerability matching
  -> vulnerability findings
  -> VEX / exception information
  -> filtering / suppression
  -> vulnerability manifest / user-visible results
```

The next step is to move from the conceptual flow into the actual implementation points in the Kubescape source code.

---

## 8. Week 2 Plan

The Week 2 focus will be on the existing implementation and the first concrete design decisions.

### Planned work

- Locate and study `OpenVulnerabilityExchangeContainer` in the source tree.
- Trace how Kubescape currently generates and persists its own VEX.
- Trace the vulnerability manifest generation path and identify the best point for the external-VEX join.
- Study the existing `WatchHandler` and `CooldownQueue` machinery that may be reused by the external feed controller.
- Review the existing SecurityException matching model in more detail.
- Start defining the fields and behavior of the proposed `VEXSource` CRD.
- Identify the smallest first implementation task that can be tested independently.

### Expected Week 2 outcome

A concrete understanding of the relevant code paths and an initial implementation/design proposal for the first part of external VEX ingestion.
