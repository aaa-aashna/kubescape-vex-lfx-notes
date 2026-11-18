# External VEX Ingestion for Kubescape

## Overview
This project enables Kubescape to consume and process externally sourced VEX documents during vulnerability scanning.

## Objectives
- Support OpenVEX and CSAF formats.
- Securely fetch and validate external VEX documents.
- Normalize documents into an internal representation.
- Apply external VEX through the existing Grype workflow.
- Improve error handling and test coverage.

## Workflow
External Source → Fetch → Parse → Validate → Normalize → Apply VEX → Scanning Results

## Tech Stack
- Go
- OpenVEX
- CSAF
- Grype

## Project Details
- **Organization:** CNCF – Kubescape
- **Program:** LFX Mentorship Term 3, 2026
- **Repository:** [kubescape/kubevuln](https://github.com/kubescape/kubevuln)
