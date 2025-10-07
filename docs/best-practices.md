# Implementation Best Practices

The Clinical Decision Support Hooks standard aims to provide real-time, actionable support to clinicians with minimal intrusion. Including [existing security and safety recommendations]([url](https://cds-hooks.hl7.org/2.0/#security-and-safety)), this living document outlines best practices to support secure, minimally intrusive decision support. These best practices are not a substitute for the security and legal review conducted by a health system before integrating 3rd party software.

## CDS Service Developers:
* CDS Services must be _fast_.  
  * Expect to invest resources into optimizing the speed of your service. CDS Client FHIR APIs may be a limiting factor in service performance.
  * CDS Services should return guidance on the order of 500 ms.
  * Use & optimize **prefetch**. (If prefetch is not supported by an EHR, the HL7 "[CRMI Module Configuration Library]([url](https://hl7.org/fhir/uv/crmi/StructureDefinition-crmi-moduleconfigurationlibrary.html))" pattern provides similar benefits).
  * Use supported search parameters to **limit** the data returned. Test and analyze a variety of requests and parameters to optimize response time.
  * Make use of date search parameters whenever possible.
  * Use `_include` and `_revinclude` to minimize the number of queries performed. For example: `MedicationRequest?subject=test-1&date=ge2024-12-01&status=active,completed,stopped&category=community&intent=order&_include=MedicationRequest:medication`.
  * Often non-production test data does not accurately represent real-world data, e.g. longterm inpatient stays. Consider pilots and be prepared for performance differences in production.
  * Implement parallelization.
  * Leverage caching.
  * Virtually never should a CDS Service query for all Observation vitals. 
* CDS Services should provide only actionable feedback to the clinician, for example: discrete suggestions instead of info cards.
* Carefully target known CDS Client capabilities such as specific hooks, widely supported FHIR profiles and search parameters. Confirm capabilities at the site-level, and be prepared with configuration points to support site-specific customizations.

### Implementation Considerations for CDS Service Developers
* Provide your CDS Service in a containerized form to facilitate local implementation.
* Site-specific middleware may cause differences in EHR capabilities and impact security, performance, and communication.
* Implementation of a CDS Service requires representatives from the local site, including IT (sysadmins, security, network, EHR analysts, interop, LIS) as well as users (clinicians).
* Consider initial deployment in "silent mode" to validate service performance with actual patient records.
* Coding systems and identifies may be organization specific. Validate.

## CDS Clients (such as EHRs)
* Support pre-fetch via your pre-existing RESTful FHIR capabilities.  
* Don't create new clinical workflows for hooks; rather, trigger a CDS Hooks from appropriate pre-existing clinical workflows. 
* Integrate CDS Hooks into any pre-existing CDS rule engine; thereby supporting both local and external cds criteria together to improve performance. 
* $lastn can improve performance. 

## Health Systems and Other CDS Hook Users:
* Follow the CDS 5 Rights; including right time, and person.
* Use the right hook during the right workflow. For example, it may be more appropriate to use the order-select or order-sign hooks for facilitating decision support regarding the order of a specific medication, rather than the patient-view
* Avoid calling the CDS Service unnecessarily. Use native EHR rule engine capabilities to limit calls. For example, a CDS Hook service should never be invoked at every patient-view.
* Use non-interruptive alerts integrated into clinician workflows when reasonable, rather than blocking workflow.
* Utilize natively supported client alert feedback and tracking capabilities, when available, to better evaluate the CDS service’s impact and utilization.
* To preserve clinical user experience, CDS Hooks should be considered only for actionable clinical decision support and not as an event notification framework.
