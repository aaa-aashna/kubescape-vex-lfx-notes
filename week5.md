Week 5 

Objectives

- Integrate external VEX ingestion with Kubescape’s existing vulnerability scanning and result-handling flow.
- Strengthen error handling for unavailable sources, fetch failures, malformed documents, and unsupported formats.
- Ensure external VEX processing failures do not silently disrupt normal vulnerability scanning.

Planned Work
The external VEX processing pipeline follows this sequence:
Source identification: Identify the configured external VEX source and its format.
Fetching: Retrieve the VEX document from the external source.
Parsing: Decode the document according to its supported format.
Validation and normalization: Validate the document and convert it into the internal representation expected by the existing processing pipeline.
Application: Pass the validated document to the existing VEX application machinery.
Result handling: Apply applicable statements to vulnerability scan results while preserving relevant metadata.
Failure handling: Ensure that errors in external VEX processing do not silently corrupt or invalidate the normal scanning workflow.

