
# Week 8 Report — External VEX Ingestion for Kubescape

## Objectives

- Improve reliability of external VEX batch preparation.
- Ensure OpenVEX and CSAF documents are processed using the correct format-specific workflow.
- Preserve vulnerability findings that do not match any VEX statement.
- Strengthen regression coverage for mixed-format VEX processing.
- Keep vulnerability matching and evaluation delegated to Grype's existing processing logic.



- [PR #1157 — Support CSAF external source ingestion](https://github.com/kubescape/kubevuln/pull/1157)
- [PR #1108 — Apply external VEX during Grype scanning](https://github.com/kubescape/kubevuln/pull/1108)
