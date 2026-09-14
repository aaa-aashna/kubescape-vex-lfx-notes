# Week 1 — Kubescape VEX Project

## Project

**CNCF - Kubescape: External VEX Ingestion for Kubescape Vulnerability Scanning**

## Week 1 goal

The goal for Week 1 was to understand how Kubescape finds vulnerabilities and where VEX information fits in the existing flow. Before changing the code, I wanted to understand the main components, the data passed between them, and the parts that may be reused for external VEX ingestion.

I also started keeping notes about the existing VEX structure and the possible connection with the selected LFX project.

---

## 1. Vulnerability scanning flow

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

### Container image

This is the image that we want to scan. It can contain operating-system packages, libraries, binaries and other software components. Some of these components may have known CVEs.

### Syft

Syft is used to create an SBOM from the image. It identifies the software packages and components present in the image.

### SBOM

SBOM means **Software Bill of Materials**. It is the inventory of the components inside the image. Grype uses this information to check which components have known vulnerabilities.

### Grype

Grype is the vulnerability scanner used in the vulnerability pipeline. It matches the components from the SBOM with vulnerability data and produces vulnerability findings.

### Vulnerability match

A vulnerability match connects a component/package with a vulnerability ID, for example a CVE. A finding should therefore be understood as:

```text
Image
  -> Package
      -> Vulnerability ID / CVE
          -> Finding details
```

### Filtering and suppression

After findings are created, Kubescape has to decide which findings should be shown to the user. Security exceptions and VEX information can affect this decision.

For our project, external VEX statements with statuses such as `not_affected` or `fixed` may allow matching findings to be suppressed. The match must be done against the correct vulnerability and product/component, not only against the CVE name.

---

## 2. VEX basics

VEX means **Vulnerability Exploitability eXchange**. It gives information about the status of a vulnerability for a particular product or component.

The important statuses for this project are:

- **`not_affected`** — the vulnerability does not affect the referenced product/component in the stated context.
- **`fixed`** — the vulnerability has been fixed for the referenced product/component/version.
- **`affected`** — the product/component is affected by the vulnerability.

The important point is that VEX is not just a list of CVEs. It connects a vulnerability with a product/component and gives the status and, where available, the reason for that status.

Example:

```text
Vulnerability: CVE-XXXX-YYYY
Product: example-image:1.0
Status: not_affected
Justification: The vulnerable code is not present or reachable
```

This is only an example to understand the relationship between the fields.

---

## 3. OpenVEX

OpenVEX is a machine-readable format used to communicate VEX statements.

A simplified view is:

```text
VEX Document
   |
   +-- Vulnerability ID
   |
   +-- Product / Component
   |
   +-- Status
   |      +-- not_affected
   |      +-- affected
   |      +-- fixed
   |
   +-- Justification
   |
   +-- Impact Statement
```

The main idea is:

```text
Vulnerability + Product + Status + Reason
```

For external VEX ingestion, Kubescape will need to read these statements and compare them with the findings produced by the vulnerability scanner.

---

## 4. Existing VEX storage shape

The project proposal mentions that kubevuln already persists VEX using the `OpenVulnerabilityExchangeContainer` shape.

Conceptually, I understand it as:

```text
OpenVulnerabilityExchangeContainer
└── Spec
    └── VEX document
        └── Statements
            ├── Vulnerability
            │    └── ID / CVE
            ├── Products / Subcomponents
            │    └── Product identity
            ├── Status
            │    └── not_affected / affected / fixed
            ├── Justification
            └── Impact Statement
```

This is important because the new controller should preferably reuse the existing storage shape instead of creating another unrelated representation.

The next step is to locate the real type definition in the Kubescape source code and check the exact fields instead of relying only on this conceptual diagram.

---

## 5. Small code example

The following is a simplified example of the kind of data we need to understand. It is not copied from Kubescape and is not the final implementation.

```go
// Simplified example only.
type VEXStatement struct {
    Vulnerability string
    Product       string
    Status        string
    Justification string
}

func shouldSuppress(statement VEXStatement) bool {
    return statement.Status == "not_affected" ||
        statement.Status == "fixed"
}
```

This example shows the basic idea, but real Kubescape code will also need to check whether the statement applies to the exact image/component and vulnerability finding. We should not implement the real matching logic until we understand the existing types and matching model.

Another simplified view of the future join step is:

```go
for _, finding := range findings {
    statement := findMatchingVEX(finding, externalVEX)

    if statement != nil && shouldSuppress(*statement) {
        // Keep the source document and statement information.
        // Do not lose the reason for suppression.
        continue
    }

    visibleFindings = append(visibleFindings, finding)
}
```

This is only a high-level example of the expected behavior. The actual implementation and function names need to come from the Kubescape codebase.

---

## 6. Why external VEX is needed

Many container images use common distribution base images such as Alpine, nginx, Python or RHEL UBI. These images can produce a large number of CVEs during scanning.

Vendors may already have investigated some of these CVEs and published VEX or CSAF statements saying that a product is `not_affected` or that a vulnerability is `fixed`.

Kubescape currently produces VEX from its own scan results, but the selected project is about consuming VEX published by other parties.

The problem we are trying to solve is:

```text
Vendor has already triaged a CVE
                |
                v
Vendor publishes VEX
                |
                v
Kubescape does not consume it yet
                |
                v
Finding may still be shown to the user
```

Our project will add read-only ingestion of these external documents and apply the relevant statements to scan findings.

---

## 7. Connection with the LFX project

The selected project has these main parts:

1. Create a namespaced `VEXSource` CRD.
2. Store the feed URL, format, image scope and refresh interval.
3. Build a controller that fetches, validates and refreshes external VEX documents.
4. Persist the documents using the existing VEX storage shape, or a clearly scoped sibling kind if required.
5. Add a join step between external VEX statements and vulnerability findings.
6. Suppress matching `not_affected` and `fixed` findings.
7. Keep provenance, including which document and statement caused the suppression.
8. Handle deduplication and normalization across feeds.
9. Add end-to-end tests against at least one real feed.

The project is read-only ingestion. Creating or editing VEX documents inside the cluster is out of scope.

---

## 8. Week 1 outcome

During Week 1, I worked on understanding the vocabulary and high-level flow needed for the project:

- How a container image is scanned.
- How Syft creates an SBOM.
- How Grype produces vulnerability matches.
- Where SecurityException and external VEX fit into the flow.
- What VEX statuses mean.
- Why the product/component identity is important when matching a VEX statement.
- Why the existing `OpenVulnerabilityExchangeContainer` shape may be reused.
- What the main deliverables of the selected project are.

The main thing still to be done is to connect this understanding with the actual Kubescape source code and confirm the exact implementation points.

---

## 9. Week 2 plan

In Week 2, I will move from the high-level concepts to the actual code.

### Planned work

- Locate and read the actual `OpenVulnerabilityExchangeContainer` definition.
- Find how Kubescape currently creates and persists its own VEX.
- Trace the vulnerability manifest generation path.
- Identify where the external VEX join can be added.
- Study `WatchHandler` and `CooldownQueue` and see how they can be reused by the controller.
- Read the existing SecurityException matching logic.
- Start defining the fields and behavior of the `VEXSource` CRD.
- Choose a small first task that can be implemented and tested independently.

### Expected outcome for Week 2

By the end of Week 2, I want to have a clearer understanding of the relevant code paths and an initial design for the first implementation task. I also want to start making small, tested changes in the actual Kubescape repository instead of only working on conceptual notes.
