# Week 1 — Understanding Kubescape VEX Project

## 🎯 Week 1 Goal

This week was mainly about understanding **how Kubescape finds vulnerabilities** and where **VEX fits into the whole process**.

Before touching the code, I wanted to understand:

- How container images are scanned.
- What Syft and Grype actually do.
- How vulnerability findings are generated.
- What VEX means and why it is needed.
- How external VEX can reduce unnecessary vulnerability findings.
- Which parts of the existing Kubescape codebase can be reused.

Basically, the goal was to understand the current system first instead of randomly jumping into implementation. 

---

## 1. How Kubescape Finds Vulnerabilities

The basic vulnerability scanning pipeline looks like this:

```text
Container Image
      ↓
     Syft
      ↓
     SBOM
      ↓
    Grype
      ↓
Vulnerability Findings
      ↓
Filtering / Suppression
      ↓
Final Vulnerability Report
```

### Example

Suppose an image contains:

```text
openssl
python
nginx
```

Grype checks these components against vulnerability databases.

If `openssl` has a known CVE, Grype may produce a finding like:

```text
Package: openssl
Version: x.x.x
Vulnerability: CVE-XXXX-XXXX
Severity: High
```

The problem is that **not every reported CVE is necessarily exploitable in the actual product**.

This is where VEX becomes important.

---

## 2. Important Terms

### Container Image

A container image is the package used to run an application. It may contain:

- Operating-system packages
- Application dependencies
- Libraries
- Binaries
- Configuration files

Any of these components can have known vulnerabilities.

### Syft

**Syft = SBOM generator.**

It scans the container image and creates a list of the software components inside it.

### SBOM

SBOM stands for **Software Bill of Materials**.

It is basically the ingredient list of a container image.

For example:

```text
Image
 ├── nginx
 ├── openssl
 ├── libc
 └── python
```

The SBOM does not directly tell us whether something is vulnerable. It tells us **what exists inside the image**.

### Grype

**Grype = vulnerability matcher/scanner.**

It takes the SBOM and checks the listed components against vulnerability information.

Its output contains vulnerability findings such as:

- Package name
- Installed version
- Vulnerability ID
- Severity
- Fix information
- Match details

### Vulnerability Finding

A vulnerability finding connects:

> **A software component → A vulnerability ID → Finding details**

Example:

```text
openssl → CVE-2024-XXXX → High severity finding
```

---

## 3. Why VEX Is Needed

This is the main reason behind the project.

Container images often contain a lot of common packages from base images such as:

- Alpine
- nginx
- Python
- RHEL UBI

Because of this, scanners can report a large number of CVEs.

But here is the issue:

> A CVE being present in a package does not always mean that the application is actually affected.

A vendor may investigate a vulnerability and publish a VEX statement saying:

- The product is **not affected**.
- The vulnerability has been **fixed**.
- The product is affected, but only under specific conditions.

Without consuming that VEX information, Kubescape may continue showing findings that have already been investigated by the vendor.

### Current problem

```text
Vendor investigates CVE
          ↓
Vendor publishes VEX
          ↓
Kubescape does not consume external VEX
          ↓
Finding may still appear
```

### What our project changes

The project will allow Kubescape to:

1. Fetch external VEX documents.
2. Validate and store them.
3. Match VEX statements with vulnerability findings.
4. Suppress findings that are marked as `not_affected` or `fixed`.
5. Preserve the source and reason behind the suppression.

So the goal is not to hide vulnerabilities randomly.

The goal is to make the final report **more accurate and less noisy**.

---

## 4. VEX Basics

VEX stands for:

> **Vulnerability Exploitability eXchange**

VEX provides information about whether a vulnerability affects a particular product or component.

The important thing is that VEX is not only about the CVE.

A proper VEX statement usually connects:

- Vulnerability ID
- Product or component
- Vulnerability status
- Reason or justification
- Additional impact information

### Main VEX statuses

| Status | Meaning |
|---|---|
| `not_affected` | The vulnerability does not affect the referenced product/component in the stated context. |
| `affected` | The referenced product/component is affected by the vulnerability. |
| `fixed` | The vulnerability has been fixed in the referenced product/component/version. |
| `under_investigation` | The impact is still being investigated. |

The important matching rule is:

> We should not match only using the CVE ID. The product/component identity and version context also matter.

For example, the same CVE may exist in multiple products, but the impact may be different for each one.

---

## 5. OpenVEX

**OpenVEX** is a machine-readable format used to share VEX statements.

A simplified OpenVEX statement contains:

- Vulnerability
- Product
- Status
- Justification
- Impact statement

In simple terms:

```text
CVE + Product + Status + Reason
```

For our project, Kubescape needs to read these statements and compare them with the vulnerability findings generated by the scanner.

---

## 6. Existing VEX Storage in Kubescape

From the project proposal, I understood that kubevuln already has an existing VEX storage structure based on:

`OpenVulnerabilityExchangeContainer`

The idea is to reuse the existing structure instead of creating a completely separate and unrelated format.

Conceptually, it contains:

- VEX document information
- Vulnerability statements
- Vulnerability IDs
- Product/component identities
- Status
- Justification
- Impact information

The next step is to confirm the **actual Go type definition and fields** in the source code.

I do not want to rely only on the conceptual structure while implementing the controller.

---

## 7. How External VEX Fits Into the Pipeline

The current scanning flow produces vulnerability findings.

Our project adds another input: **external VEX documents**.

The VEX matching step needs to check:

- Is the vulnerability ID the same?
- Does the product/component match?
- Does the version or package context match?
- What status does the VEX statement provide?
- What is the reason for the status?
- Which external document provided the statement?

If the statement says `not_affected` or `fixed`, the matching finding may be suppressed according to the project rules.

##

---

## 8. Week 1 Outcome

By the end of Week 1, I understood:

- The basic Kubescape vulnerability scanning flow.
- The role of Syft.
- What an SBOM contains.
- How Grype generates vulnerability findings.
- Why scanners can produce noisy results.
- What VEX is used for.
- The meaning of important VEX statuses.
- Why product/component identity matters during matching.
- What OpenVEX means.
- Why reusing the existing VEX storage structure is useful.
- The main implementation goals of the LFX project.

The next step is to connect this understanding with the actual Kubescape/kubevuln source code and start identifying the exact implementation points.

---

## Main takeaway

The project is basically about making vulnerability reports smarter.

Instead of treating every scanner finding as equally relevant, Kubescape should also consider:

> **“Has the vendor already explained whether this vulnerability actually affects this product?”**

That is where external VEX ingestion comes in.
