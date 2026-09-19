##  Week 2 plan

I will move from the high level concepts to the actual code.

### Planned work

- Locate and read the actual `OpenVulnerabilityExchangeContainer` definition.
- Find how Kubescape currently creates and persists its own VEX.
- Trace the vulnerability manifest generation path.
- Identify where the external VEX join can be added.
- Study `WatchHandler` and `CooldownQueue` and see how they can be reused by the controller.
- Read the existing SecurityException matching logic.
- Start defining the fields and behavior of the `VEXSource` CRD.
- Choose a small first task that can be implemented and tested independently.

Okay so now , i am thinking of add a file called document.go
Which will solely focus on the actual data VEX structure and validation logic 
And then with that i have added document_test.go which will check the test and verify if all things are workking and passes the test cases 

I have added the package string cuz
STRINGS 
Used to check whether a string is empty or contains only spaces:
strings.TrimSpace(document.ID) == ""
This catches both:
""
and:
"   "

The initial VEX ingestion changes are in place, and the relevant package-level tests are passing. The full repository test suite is currently taking longer than expected, so I’m validating the remaining components separately and will keep pushing incremental updates as the implementation progresses.


'
case StatusNotAffected,
		StatusAffected,
		StatusFixed,
		StatusUnderInvestigation:
		return nil
'

this is the code snippet for the case where there are different cases present 

Deleted file 
PR is superseded by #880, and that the duplicate internal/vex implementation should be dropped.
	


