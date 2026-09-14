## 9. Week 2 plan

I will move from the high-level concepts to the actual code.

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

By the end of Week 2, I want to have a clearer understanding of the relevant code paths and an initial design for the first implementation task. 

I also want to start making small, tested changes in the actual Kubescape repository instead of only working on conceptual notes.
