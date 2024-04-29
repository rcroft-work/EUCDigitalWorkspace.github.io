---
title: VMware REST API Versioning
draft: false
date: 2024-03-01
authors:
  - rcroft-work
---

## Typographical conventions

The text in this document is split into two categories — normative statements and notes. Normative statements represent requirements towards APIs and Products. Notes aim to clarify the normative statements by presenting examples, background information, opinions of engineers, references to other specifications etc. Normative statements are represented as indented information boxes. Notes use all other formats in the document. Example normative statement:

> Normative Statement

Within normative statements, the conventions of [RFC 2119](https://tools.ietf.org/html/rfc2119) are used. These define the meaning of MAY, SHOULD, SHOULD NOT, MUST and MUST NOT.

## API Specification Version

The goal of using a standard API Specification Versioning mechanism is providing clear expectations around various types of API contract changes in order to help to maintain interoperability.

> API Specification SHOULD be versioned following [Semantic Versioning](https://semver.org/).  

Given a version number `MAJOR.MINOR.PATCH`, increment the:

1. MAJOR version when you make incompatible API changes,
2. MINOR version when you add functionality in a backwards-compatible manner, and
3. PATCH version when you make backwards-compatible bug fixes.

The PATCH version SHOULD NOT be considered by API level integrations and tooling. Changes in the API specification that don't change the contract, e.g. fixes or additions in the documentation, SHOULD result in incrementing the PATCH version. 

API Specification version MAY be referred to using a short version representation, omitting the PATCH number, for documentation and compatibility purposes.

| Version (MAJOR.MINOR.PATCH) | Short Version (MAJOR.MINOR) | Description                    |
|-----------------------------|-----------------------------|--------------------------------|
| 2.3.1                       | 2.3                         | Version 2.3 GA Release         |
| 2.4.0-alpha1                | 2.4-alpha1                  | Alpha 1 release of version 2.4 |
| 2.4.0-beta                  | 2.4-beta1                   | Beta release of version 2.4    |

> API Specification version SHOULD be provided as part of the API Specification

API Specification represents the contract between the API Provider and API Consumer and is a mechanism for communicating version and contract updates.

### VMODL2

For APIs defined using VMODL2 specification API Specification version MUST be provided using the @Component annotation

> VMODL2 API Specification version SHOULD be provided as part of the Component declaration following MAJOR.MINOR format.

```python
/**
 * The {@name com.vmware.vcenter} {@term package} provides {@term services} for
 * managing VMware vSphere environments.
 */
@Component(versions= {"1.0", "1.5", "2.0"})
package com.vmware.vcenter;

import vmodl.lang.Component;
```

[https://wiki.eng.vmware.com/VAPI/Specs/VMODL2/ApiLifecycle#Component\_version\_definitions](https://wiki.eng.vmware.com/VAPI/Specs/VMODL2/ApiLifecycle#Component_version_definitions)

### OpenAPI Specification 2.0 (Swagger 2.0)

```yml
---
swagger: "2.0"
info:
  description: "This is a sample server Petstore server."
  title: "Swagger Petstore"
  version: "1.0.0"
```

> OpenAPI 2.0 API specification version MUST be provided using the [Info→version](https://swagger.io/specification/#infoObject) property following MAJOR.MINOR.PATCH or MAJOR.MINOR format.

### OpenAPI Specification 3.x

> OpenAPI 3.x API specification version MUST be provided using the [Info→version](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#infoObject) property following MAJOR.MINOR.PATCH or MAJOR.MINOR format.

```yaml
openapi: 3.0.1
info:
  title: Swagger Petstore
  description: 'This is a sample server Petstore server.'
  version: 1.0.0
```

### Other Interface Definition Languages(IDLs)

In order to help to move the standardization effort forward, when attempting to use an IDL not listed above, API Providers are expected to share their intention with the API Standards group and propose to include the IDL in the Standard. In the scope of the API Versioning Standard, that would result in including an API Specification versioning recommendation for the IDL. If a versioning mechanism is not provided by the IDL, it can be proposed by the API Provider as part of the nomination for standardization.

### API Implementation version alignment

To achieve closer alignment and consistent experience, it is recommended to use Semantic Versioning for API implementation where possible.

If both API Specification and API Implementation are versioned using Semantic Versioning, their versions should not be tightly coupled. For example, API Implementations may have multiple patch versions without affecting the API Specification version. 

However, incrementing the MINOR version of the API Implementation should result in propagating the increment to the API Specification version while exposing the change.

A single API Specification may include multiple API implementations, spanning multiple implementing services. The API Specification provides an abstraction for consumers and hides the implementation details.  

| API Specification | API implementation |
|-------------------|--------------------|
| API Specification | API implementation |
| 2.1.3             | Foo API 1.2.3      |
| Bar API 2.4.1     |
| Baz API 2.3.4     |

> The API Specification MINOR version MUST be incremented if any of underlying implementations introduce a new, non-breaking functionality exposed through the API.

> The API Specification MAJOR version MUST be incremented if any of underlying implementations introduce a breaking change.

## SDK Versioning

SDKs typically include multiple versioned components. Aligning component versions with the SDK version is not always possible. 

> SDK deliverables SHOULD be versioned using Semantic Versioning, where major versions are updated if a major version of an API covered by the SDK is incremented. Otherwise, the minor version should be incremented for feature releases.

The MAJOR version of the SDK increment should be applied when any of the included APIs undergo a major release. Alternatively, if an SDK release only includes new features, documentation updates, sample updates or fixes, should typically increment the minor version. 

Note that the addition of a new API or a component into the SDK qualifies as a major update. Major releases can also be justified in case of a major rework of the SDK content or overall structure.

> SDK versions SHOULD be independent of product versions. 

SDK deliverables typically cover multiple products each having independent versions and release schedules. Hence, the SDK versions do not have to be aligned to product versions, nor should major increments be triggered by a major release of a product. On the contrary, major releases of APIs covered by the SDK should trigger SDK major release increments.

> SDK deliverables MUST explicitly list the most recent product versions tested against and clarify compatibility expectations. 

Developers should be able to easily pick a version of the SDK that is the latest compatible with the set of products they are interested in working with. 

## API Changes and Product Releases

To maintain clear expectations only certain types of API changes are allowed per product release type. Release types outlined below are provided for semantical alignment purposes and may not match exactly the terminology defined by specific products. If such an alignment is not possible, product teams should be reaching out to the API Standards team with the details outlining the mismatch. 

### Major releases

> Major releases MAY introduce new APIs, as well as changes to existing APIs. 

> Breaking API changes introduced in a major release MUST follow the Evolution and Deprecation guidelines. 

There are no constraints on the type of API changes for major releases. So as long as API changes follow API evolution and deprecation guidelines, any change is allowed.

### Minor releases

> Minor releases MAY introduce new APIs and backward-compatible changes to existing APIs. 

Minor releases may introduce new APIs, as well as update existing APIs while maintaining backward-compatibility. For existing APIs, such changes would be accompanied by MINOR version updates.

> Minor release MUST NOT introduce breaking API changes.

Neither backward-incompatible changes nor API removal is allowed in minor releases. 

> Minor releases MAY deprecate existing APIs. 

Deprecation of an API is allowed in minor releases, following the API deprecation guidelines. 

> Minor releases MUST NOT remove deprecated APIs.

API removal is considered a breaking change and is only allowed in Major releases. 

### Patch releases

> Patch releases MUST NOT introduce new APIs. 

New APIs and new features can only be introduced at the Minor release level.

> Patch releases MAY include PATCH version updates in existing APIs.

No new features or breaking changes are allowed in Patch releases. Hence, there may not be MINOR or MAJOR version changes.

> Patch releases MUST NOT deprecate existing APIs.

APIs can only be deprecated in Minor or Major releases.

## REST API Versioning

REST API Versioning is covered in the REST Standard: [https://gitlab.eng.vmware.com/core-build/rest-specification/blob/v2.0/REST-specification.md](https://gitlab.eng.vmware.com/core-build/rest-specification/blob/v2.0/REST-specification.md)
