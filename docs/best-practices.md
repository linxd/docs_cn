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

## Security

The CDS Hooks security model requires thought and consideration from implementers. Security topics are never binary, are often complex, and robust implementations should factor in concepts such as risk and usability. Implementers should approach their security related development with thoughtful care.

The CDS Hooks specifications already provides some guidance in this space. The information here is supplemental to the existing specification documentation.

The CDS Hooks security model leverages several existing RFCs. These should be read and fully understood by all implementers.

- [rfc7519: JSON Web Tokens (JWT)](https://tools.ietf.org/html/rfc7519)
- [rfc7517: JSON Web Key (JWK)](https://tools.ietf.org/html/rfc7517)
- [rfc7515: JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515)
- [rfc7518: JSON Web Algorithms (JWA)](https://tools.ietf.org/html/rfc7518)

### CDS Clients

Implementers of CDS Clients should:

**Maintain an allowlist of CDS Service endpoints that may be invoked.**

Only endpoints on the allowlist can be invoked. This ensures that CDS Clients invoke only trusted CDS Services. This is especially important since CDS Clients may send an authorization token that allows the CDS Service temporary access to the FHIR server.

**Issue secure FHIR access tokens.**

*If* a CDS Client generates access tokens to its FHIR server, the tokens should:

- Be unique for each CDS Service endpoint.
- Be very short-lived.
- Provide the minimum necessary access for the CDS Service. This includes both SMART scopes as well as the patient(s)/data that can be accessed.

**Audit CDS Service's access to data**

Regardless of the use of prefetch or more typical FHIR RESTful APIs, CDS Clients should audit and report on data access. 

### CDS Services

#### JWT

Upon being invoked by a CDS Client, the CDS Service should first process the given JWT to determine whether to process the given request. In processing the JWT, CDS Services should:

1. Ensure that the `iss` value exists in the CDS Service's allowlist of trusted CDS Clients.
2. Ensure that the `aud` value matches the CDS Service endpoint currently processing the request.
3. Ensure that the `exp` value is not before the current date/time.
4. Ensure that the `tenant` value exists in the CDS Service's allowlist of trusted tenants (may not be applicable to all CDS Services).
5. Ensure that the JWT signature matches the public key on record with the CDS Service. See additional notes below.
6. Ensure that the `jti` value doesn't exist in the short-term storage of JWTs previously processed by this CDS Service.

Once the JWT has been deemed to be valid, the `jti` value should be stored in the short-term storage of processed JWTs. Values in this storage only need to be kept for the maximum duration of all JWTs processed by this CDS Service. If the CDS Clients are adhering to best practices, this should be no more than an hour.

Verifying the JWT signature is a critical step in establishing trust of the caller of the CDS Service. As part of the allowlist of trusted CDS Clients, information on the public key(s) used by the CDS Client should also be stored. In some cases, this public key may be shared out-of-band. In other cases, the public key may be available at a remote endpoint and cycled on a regular basis. It is this latter case in which CDS Services should maintain their own rotating cache of public keys for the CDS Client.

CDS Services should never store, share, or log JWTs to minimize the risk of theft and replay attacks. Information within the JWT (for instance, `iss`, `tenant`, `jti`) can be logged safely and is especially useful for analytics.

If a CDS Service deems a JWT to be invalid for any reason, it should not leak the details of why the JWT failed validation back to the caller. If the caller were a malicious threat actor, leaking detailed information as to what was invalid may give the threat actor guidance on how to shape future attacks. Instead, responding to the request with a HTTP 401 Unauthorized response status code without any additional information is recommended.

#### FHIR Access

CDS Services should never store, share, or log the FHIR access token (`fhirAuthorization.access_token`) given to it by the CDS Client. The access token should be treated as an extremely sensitive, transient piece of data.
