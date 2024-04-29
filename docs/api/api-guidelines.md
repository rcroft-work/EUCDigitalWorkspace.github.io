---
title: VMware REST API Standard
draft: false
date: 2024-03-01
authors:
  - rcroft-work
---

# VMware REST API Standard Practices

September 2021

Version 2.1

## Overview

There have been many discussions and efforts at VMware over the years around how to bring the appropriate level of consistency to our APIs as to allow customers to learn APIs faster and also provide consistent documentation and tooling support across the board. This lack of consistency has been a long-standing problem for us, and one that is complicated by VMware's vastly diverse portfolio and underlying technologies.

For example a Java developer willing to work with vRA, vCenter and NSX together as of this moment may experience difficulties as SDKs may not be readily available in Java for all products, even if SDKs are available they are very likely to use conflicting versions of 3rd party libraries and the design patterns behind the different products are different. There is also a conflicting point of view that enforcing consistency top down will stifle innovation and prevent use of the latest technologies in VMware's products.

This specification aims to strike a balance between these competing priorities by keeping consistency goals in areas important for our customers priorities, while giving sufficient freedom to individual teams to adopt innovative approaches. This specification will also defer to superseding, enterprise-wide standards where VMware has warranted in very specific areas that such requirements exist.

- Align the look and feel of APIs
- Provide consistent set of tooling and documentation across VMware APIs
- Refer to superseding standards as appropriate

To support these goals the current specification lists requirements and expectations in 6 critical areas:

- Request and Response Types
- Message Payload
- API Errors
- API Documentation
- API Evolution
- API Deprecation and Removal

It is not the goal of this document to drive architectural consistency at VMware. This document is aimed to provide guidance for designing the REST API - the protocol they use, the encoding of messages and structuring access patterns. It is understandable that some requirements here may have bearing to product design and architecture. In those cases the specification will try to provide sufficient room for product architects to make choice for their product.

## Guiding Principles

The document discusses a number of areas and while we try to make it unambiguous it is possible that the guidance is lacking sufficient detail to make design and implementation choice. We want to thus lay out some guiding principles that engineers can use:

- Favor local consistency over global consistency — when faced with a violation of requirement it is more important to preserve consistency with adjacent APIs in the same component to minimize the learning barrier for end users. On a similar note, when a major change to the API is undertaken the thinking should be to provide unification within the family of VMware APIs as to reduce the friction for the new API that will be released.

## Typographical conventions

The remainder of the document presents requirements towards VMware REST APIs. The text is split in two categories — normative statements and notes. Normative statements represent requirements towards product APIs. Notes aim to clarify the normative statements by presenting examples, background information, opinions of engineers, references to other specifications etc. Normative statements are represented as information boxes. Notes use all other formats in the document. Example normative statement

> Normative Statement

Within normative statements the conventions of [RFC 2119](https://tools.ietf.org/html/rfc2119) are used. These define the meaning of MAY, SHOULD, SHOULD NOT, MUST and MUST NOT.

## Related Work

- [NSX-T APIs: Developer Guides & Standards](https://confluence.eng.vmware.com/display/NSBU/NSX-T+APIs%3A+Developer+Guides+and+Standards)
- [CMBU API Standardization](https://confluence.eng.vmware.com/display/CAEng/CMBU+API+Standardization)
- [VAPI Best Practices](https://confluence.eng.vmware.com/display/VAPI/vAPI+Best+Practices) & [VMODL2 Documentation Guidelines](https://confluence.eng.vmware.com/display/VAPI/VMODL2+Documentation+Guidelines)
- [vCloud Director — API development guide](https://confluence.eng.vmware.com/display/VCD/API+Development+Guide)

### 3rd party

- [OpenAPI 3.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md), [legacy v.2.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md)
- [Microsoft REST API Guidelines](https://github.com/Microsoft/api-guidelines)
- [Google API Design Guide](https://cloud.google.com/apis/design/)
- [OpenStack Nova Compute API reference](http://developer.openstack.org/api-ref/compute/#compute-api)
- [AWS documentation index](https://aws.amazon.com/documentation/)

## Request and Response Types

This chapter sets out the basic requirements towards the request/response exchange between REST client and a service:

> RR-1: All requests to the REST API MUST follow HTTP 1.1 specification

HTTP 1.1. protocol is defined in RFC 7230, RFC 7231, RFC 7232, RFC 7233, RFC 7234 and RFC 7235

HTTP protocol is in the heart of the internet and thus all REST APIs are expected to use it. The HTTP protocol specifies a lot of semantics around requests which help standardized clients to operate the API. It is thus expected that API authors are familiar with the HTTP specification and use it appropriately. Key parts of the HTTP specifications relevant to APIs are HTTP verb semantics, response status codes, standard headers, authentication mechanisms, safe, idempotent and cache-able methods.

The remainder of this specification defines constraints on top of the HTTP protocol to make it easier for modeling APIs. No exhaustive effort shall be made in this document to reiterate concepts of HTTP described in relevant RFC documents.

This specification is aimed at REST APIs that form the management path of our API surface. There may be other APIs that work with binary data and are not subject to this specification. Example of such APIs may include: file transfer, screen shots, push notifications, south bound driver interfaces, console access etc.

> RR-2: APIs MUST be able to consume and produce JSON (application/json) [RFC 8259 - The JavaScript Object Notation (JSON) Data Interchange Format](https://tools.ietf.org/html/rfc8259) unless they deal with file or stream data

JSON is the de-facto standard for REST API message format as of 2016. It is beneficial to standardize around this syntax and enable common serialization infrastructure.

This rule applies to all current JSON based REST APIs and future APIs. There is a set of legacy XML APIs which we hope will support JSON as the respective products evolve. There is realization that transition will take time and for some products that are nearing end of life the transition may not happen.

There are APIs that transfer files contents to or from client. Those use cases are covered in the [File Transfers](#file-transfers) chapter. Such APIs need to consume or produce binary data e.g. obtain screen shot of a virtual machine. Those APIs may consume or produce other MIME types that are more appropriate to their use cases e.g. `GET /vm/{vm id}/screen-shot` may return image/jpeg

> RR-3: All API functionality MUST be accessible using single request and response pattern (i.e. clients MUST not be required to use streaming, web sockets etc.)

It is a common assumption in the existing REST tooling that an API interaction is defined in terms of a single request and single associated response conducted in the same HTTP request. While new standards like web sockets and comet provide utility in specific use cases they enable communication patterns that many off the shelf tools currently do not support. It is thus required to provide the full functionality of the API over simple request/response interface. It is permitted to provide alternate forms of communication as needed.

> RR-4: API operations MUST be uniquely identified through HTTP verb and URI combination

In REST APIs users interact with resources identified by URI and use standard verbs to specify the operation on the resource e.g. HTTP GET on resource URI to retrieve the representation and HTTP PUT/PATCH to update etc. HTTP headers such as Content-type and Accept may be used to select among alternative resource representations e.g. XML/JSON rendering of a resource. It goes against this basic principle to encode API operations in the message header or body ([REST APIs must be hypertext driven, R. Fielding, 2008](http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)). Encoding operation name in the message body makes the protocol opaque to the HTTP infrastructure and prevents potential benefits like caching and operation through proxies.

This requirement will allow us to use off the shelf REST tooling for invoking, serving and documenting APIs. Examples of such tooling include RAML, Swagger, Spring MVC, Apache CXF. Protocols such as XML RPC or SOAP do not fit this requirement as the operations are identified in the message body and require further interpretation of the message body to determine how to handle the request.

### Security requirements

REST APIs may be utilized by different types of clients in variety of environments. These pose various types of threads towards the security of the APIs. A simple set of rules in this chapter aims to provide guidance to API authors on how to avoid common pitfalls that may be detrimental to their service or its clients.

> RR-5: Encryption MUST be used for communication that is not confined to the loopback interface.

All communication to APIs must be conducted over secure channels using approved algorithms. This prevents unauthorized access to security tokens and/or API payload data.

Use of unencrypted channels is acceptable for local loopback communication as to minimize the cost of communication. For example unencrypted local loopback channel may be used to communicate to sidecar proxy that is off loading TLS from service implementations.

> RR-6: API SHOULD NOT return sensitive data like secrets/passwords

APIs that need to manage client secrets MUST only accept secrets as inputs and not return them as outputs. Returning passwords may lead to undesired leakage of information.

This risk is often seen in APIs for configuration of remote connections.

It is feasible to return secrets and passwords when those are generated or owned by given API and/or used for integration purposes. For example the OAuth 2 API returns access and refresh tokens as part of the authentication workflow. See [RFC 6570 Section 4. Example Access Token Response](https://tools.ietf.org/html/rfc6750#section-4)

> RR-7: APIs MUST be hardened against well-known attacks

Interpreting inputs from the network should be done with extreme caution. A common vector for attack on APIs is to exploit various deserialization weaknesses. Those include overflowing input buffers, flooding a service with requests, XML expansion and code execution attacks. As an API author make sure to choose mature communication and deserialization libraries and address known risks with the chosen deserialization approach.

Examples of common problems to check include:

- Make sure that appropriate request size filters are applied to prevent memory pressure due to oversized API requests.
- When using XML, make sure to disable various [XML expansion mechanisms](http://www.ws-attacks.org/XML_Entity_Expansion).
- Make sure that deserialization to objects in a language runtime cannot be used for arbitrary code execution by invoking vulnerable methods or instantiating classes that are not part of the API surface.
- Use rate and concurrent request limits that guarantee the health of an API endpoint

> RR-8: API access SHOULD be properly authenticated

API calls should require authentication. Exception to this rule are APIs that allow a client to discover the capabilities of an API endpoint. For example API to retrieve API release number may be open to public unauthenticated access.

> RR-9: APIs SHOULD use standard HTTP headers for authentication

Authentication tokens should be transferred using standard compliant HTTP headers to enable interoperability. See [RFC 7235 Section 4.2. Authorization](https://tools.ietf.org/html/rfc7235#section-4.2)

> RR-10: API Authentication MUST NOT use cookies

APIs must not use cookies to hold credentials as cookies are automatically sent with the request by browsers when a cross origin request is made. Cookies must not be used to hold session keys. If this can't be avoided, sufficient anti-CSRF measures should be taken. See the [Cross-Site Request Forgery Prevention Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md).

> RR-11: CORS support SHOULD be restricted

CORS should be avoided. In some cases there are overriding business requirements that will necessitate the use of CORS. In those cases CORS may only be used with appropriate restrictions including:

- allowed headers
- allowed methods
- allowed origin if possible

Avoid sending credentials when using CORS.

> RR-12: APIs MUST specify `content-type` header in responses when response body is present.

`application/json` with UTF-8 character encoding should be used and properly manifested in API responses. See [RFC 8259 Section 8.1. Character Encoding](https://tools.ietf.org/html/rfc8259#section-8.1), [RFC 8259 Section 11. IANA Considerations](https://tools.ietf.org/html/rfc8259#section-11), [RFC 7231 Section 3.1.1.5. Content-Type](https://tools.ietf.org/html/rfc7231#section-3.1.1.5)

> RR-13: APIs SHOULD NOT accept sensitive data in query parameters in URI

Query parameters may be accessible to rouge scripts or visible in the browser window and must not be used to transfer sensitive or secretive data. URIs including query parameters are often cached, persisted and included in application logs.

There are rare exceptions to this rule when short-lived tokens need to be sent using URI. For example in the OAuth 2 specification the authorization code grant workflow uses query parameter. See [RFC 6749 Section 4.1.2 Authorization Response](https://tools.ietf.org/html/rfc6749#section-4.1.2). Clients sending sensitive data in the URI should send `cache-control` header with `no-store` option and server 2xx responses are to include `cache-control` with `private` option. (from [RFC 6750 Section 2.3](https://tools.ietf.org/html/rfc6750#section-2.3)).

### Design for robust operation in distributed systems

Product APIs are used by customers, second parties, and for integration between VMware products; and thus the actual completion of an API request often involves multiple services. For this reason APIs should be designed with distributed execution in mind. API designers should at minimum consider the following factors in this regard:

> RR-14: API responses SHOULD be bounded in size

API servers and clients should be able to set some expectations for the amount of resources needed to handle API calls. This problem specifically manifests with list and query functionality whose response size may grow substantially in large environments. Clients should be able to specify page size in a list/query request. Further, servers are free to reject the request or respond with reduced result sets to preserve server stability.

For certain lists like SCSI devices where list size has implicit size limit it may not be necessary to implement complex pagination interface. This applies as long as the maximum possible size for the list is manageable.

Notable exceptions to this rule are streaming APIs, data transfer APIs, console access

> RR-15: Individual HTTP requests SHOULD be designed to complete fast (< 0.5 second)

The basic requirement is to design all remote API interactions to complete within short interval of time. This is important to develop responsive user interfaces as user interactions may need to block for the duration of an individual API call (see [User Interface Timing Cheatsheet](http://www.stevenseow.com/papers/UI%20Timing%20Cheatsheet.pdf)). If a back-end processing is expected to extend beyond the recommended limits then it should be split into several API interactions. For example a "create report" operation that may last 30 minutes can be split into "create report request" operation that accepts the inputs and return 202 (Accepted) with a tracking token identifying the request and a read operation to check the status of the request and obtain reference to the newly created report upon success.

It is not uncommon that API operations become more complex as the product evolves. In case an operation duration extends in a subsequent product version alternative API may be offered that allows asynchronous execution

Execution time may also depend on the size of environment or size of data relevant to the request. In this case an API can offer choice of synchronous and asynchronous execution.

A notable exception to this rule are HTTP streaming APIs using technologies such as [Comet](https://en.wikipedia.org/wiki/Comet_(programming)) or [Server Sent Events](https://en.wikipedia.org/wiki/Server-sent_events).

> RR-16: Protocol & framework timeouts SHOULD be used appropriately

At VMware we have sometimes neglected those mechanisms and been left with hard to diagnose and solve issues e.g. [PR 1591780](https://bugzilla.eng.vmware.com/show_bug.cgi?id=1591780) and [PR 1506115](https://bugzilla.eng.vmware.com/show_bug.cgi?id=1506115). For example clients may find < 15 seconds TCP connect timeout and TCP read timeouts of < 2 minutes to work well. Similarly servers must be designed such that they can answer requests swiftly and report a re-triable error in cases when things go wrong. These may be dangerous if they spawn many re-tries so implementations should also be able to recognize such conditions and avoid failure.

> RR-17: API operations SHOULD provide means to recover from network failure

Even with APIs designed to complete fast it is not impossible for network failure to occur while request is in progress. In those circumstances client should have a reliable way to resume its intended workflow once network connectivity is restored. One way to achieve such reliability is to design mutation APIs such that they allow to be retried without negative consequences.

> RR-18: API operations SHOULD provide adequate concurrency controls

As scale is increased the probability of an entity being read and modified near simultaneously from concurrent workflows increases. To avoid race conditions in such scenarios APIs should define clear concurrency semantics and provide adequate protection mechanisms against concurrency errors. Examples of such mechanisms include last writer wins, optimistic locking through ETags (see [HTTP Etag Wikipedia article](https://en.wikipedia.org/wiki/HTTP_ETag))

## Basic Constructs

### Data Types

> CNST-1: JSON representations MUST NOT include type information except for discriminating between alternative object types.

As the REST APIs may be used by humans directly or developers may want to craft simple SDKs by hand it is important to make data representations simple and intuitive. To this end we expect APIs to not obfuscate JSON with extensive schema information. If schema information is provided it SHOULD be out of band e.g. in an API definition such as Swagger, RAML or WADL file/resource. It is ok to include type identifiers and auxiliary data in server responses that is needed to operate the API. (See [Example Xenon response](#example-xenon-response))

#### Primitive types

> CNST-2: REST services SHOULD apply the robustness principle (aka Postel's law) to tolerate slight variations in input while being strict in producing output

> CNST-3: REST services SHOULD use the primitive types defined in JSON [RFC 8259](https://tools.ietf.org/html/rfc8259) and [JSON Schema](http://json-schema.org/specification.html)

The following table lists the JSON types.

| Type         | Sample            | Description                                                                                                                                                                                                                                                                                        |
|--------------|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| true / false | true              | Two state variable. While RFC 8259 is pretty clear regarding the values representing true and false services often tolerate other commonly accepted representations as inputs e.g. 1 and 0, "True" and "False" string literals etc.                                                                |
| string       | "Hello World!"    | A quoted unicode sequence of characters                                                                                                                                                                                                                                                            |
| number       | 1.61803398875     | JSON specifies only one number format. Services may specify additional validation constraints - whole numbers, byte, 32 bit integer etc.                                                                                                                                                           |
| null         | null              | useful for denoting unset value e.g. to disable a policy it may be useful to unset it. In PATCH partial updates null may be interpreted differently from missing value i.e. null resets field to null, missing leaves the old value. The behavior should be specified for each products/component. |
| object       | { "answer" : 42 } | represents an object or map                                                                                                                                                                                                                                                                        |
| array        | [30, 42]          | represents ordered list of elements                                                                                                                                                                                                                                                                |

On top of the basic JSON types through syntax restrictions JSON schema and VMware practice define additional useful data types

| Restriction | Sample                                  | Description                                                                                                                                                                                                |
|-------------|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| date-time   | 2004-07-25T06:18:20.521-04:00           | String representing date and/or time values encoded as per JSON Schema (ISO 8601, RFC 3339 section 5.6)                                                                                                    |
| email       | jdoe@aol.com                            | email address as per JSON schema                                                                                                                                                                           |
| hostname    | nimbus.eng.vmware.com                   | hostname as per JSON schema                                                                                                                                                                                |
| ipv4        | 192.168.4.1                             | IPv4 address as per JSON Schema                                                                                                                                                                            |
| ipv6        | FEDC:BA98:7654:3210:FEDC:BA98:7654:3210 | IPv6 address as per JSON Schema                                                                                                                                                                            |
| uri         | ftp://joe:secret@ftp.mysite.com         | URI as per JSON schema                                                                                                                                                                                     |
| Binary      | SGVsbG8gV29ybGQh                        | String value containing base64 encoded binary data. This is suitable for small amounts of binary data like hashes, public keys etc. This is not defined in JSON Schema but is used in several VMware APIs. |
| integer     | 42                                      | a number without fractional component. In JSON schema defined as multiple of 1                                                                                                                             |

#### Objects

JSON objects are to be used for representing objects and maps.

In most cases the schema of an object is clear from its context. For example virtual machine create operation accepts a JSON object with predefined set of members.

In some occasions the type of object may not be clear from context. In those cases a type discriminator member is to be used to identify the correct type.

Typical use cases that require use of type discriminator include:

1. Errors - error types typically share most of their members however the semantics are vastly different based on the type of error.
2. Extensible Polymorphic resources - when a given object can be one of several complex semantic types

> CNST-4: If type discriminator is to be included it SHOULD be the first member of a serialized object.

Below is example polymorphic object from NSX API.

```json
// OK - resource_type member indicates the type of an object that may be one of several types
{
    "resource_type": "SpoofGuardSwitchingProfile",
    "display_name": "spoof-guard-lswitch-bindings",
    "white_list_providers": "LSWITCH_BINDINGS"
}
```

Alternative way to represent type discriminators is by using member name i.e. in a parent object only one of a set of members is set at a time and each member name is associated with specific type. For example a schema may contain `result` and `error` fields to represent the result of an operation.

```json
{
   "error" : {
      "error_type": "com.vmware.vapi.std.errors.invalid_argument"
   }
 }
```

> CNST-5: Type discriminators SHOULD be used sparingly.

For example it is NOT OK to use type discriminators for primitive types across the board, thereby embedding schema information in the payload. Instead out of band schema of the containing object should define the type of a member.

Good example

```json
{
    "age" : 35
}
```

NOT OK

```json
{
  "age": { "int" : "35" }
}
```

See [NSX T Switching Profiles](http://build-squid.eng.vmware.com/build/mts/release/bora-3788284/publish/nsx-manager/api.html#Sections.Logical%20Switching.Switching%20Profiles)

### Naming Conventions

In June 2014 a discussion was held in the OCTO API group around naming of parameters in REST APIs (see [Field naming debate](related/field_naming_decision.md)). A consensus was reached that

> CNST-6: JSON object member names SHOULD use lower\_snake\_case

Example: `"resource_description": "A description"`

> CNST-7: URIs SHOULD use kebab-case

Example: `/my-resources/123`

The above rule applies only to the predefined segments of the URI path. When identifier or enumeration values are part of a URI as parameters their values should not be modified to follow the above guidance.

> CNST-8: Enumerated values SHOULD use UPPER\_SNAKE\_CASE

Example: `"state": "POWERED_OFF"`

Following the guidance for JSON object member names

> CNST-9: Query parameter names SHOULD use lower\_snake\_case

Example: `/virtual-machines?power_state=POWERED_OFF`

The HTTP RFCs define a number of standard headers using kebab-case. For example the conditional headers defined in [RFC 7232](https://tools.ietf.org/html/rfc7232) last-modified, if-match etc. HTTP header hames should be treated as case insensitive as per [HTTP 1.1 RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.2). As per [HTTP/2 RFC 7540](https://tools.ietf.org/html/rfc7540#section-8.1.2) header names must be lower cased before transmission. To clearly define the API contract the following guidance to header names applies

> CNST-10: Header parameter names SHOULD use lower-kebab-case

Example: `record-offset: 13`

Below is an example illustrating full request:

PREFERRED

```http
PUT /my-resources/123?update_details=true
if-match: "182584fa-1767-410e-9b63-ea8bc07b8d2f"
```

```json
{
    "resource_name": "test",
    "resource_description": "A description",
    "state": "POWERED_OFF"
}
```

Acceptable for PRE EXISTING APIs - as previously discussed PascalName and camelCase have certain issues we want to avoid. It is not nice to use mixed case URIs from usability perspective

```http
PUT /myResources/123
{
    "resourceName": "test", // camelCase identifier is problem
    "ResourceDescription": "A description", // PascalName identifier is problem
    "state": "powered-on" // kebab-case enumerated value is problem
}
```

Some of our products use CamelCase and/or mixedCase identifiers. In those cases for consistency it is preferable to retain the existing convention.

### URI Structure

> CNST-11: The URI SHOULD identify the component/product and resource type being accessed

Example: `/vcenter/folders`

> CNST-12: The URIs SHOULD use plural names for resources

Example: `/content/libraries`

Abbreviations in APIs should be generally avoided except when those are industry standard. In cases when abbreviations are used they should be pluralized according to Chicago Manual of Style.

> CNST-13: Components SHOULD be able to handle a base URI prefix dependent on the packaging and deployment (suite , product etc.) The suggested structure of the URI is as follows `{base URI}/{component}/{resource}`

```http
GET https://sddc.vmware.com/content/libraries/{library-id}
// https://sddc.vmware.com/ - base URI
// content - product/component namespace
// libraries/{library-id} - resource
```

> CNST-14: URIs SHOULD NOT contain redundant names

Bad example

```http
GET https://sddc.vmware.com/vcenter/vm/vm/{vm}
```

> CNST-15: Resource names SHOULD be simple nouns expressing only the semantics of the resource

Avoid using suffixes in resource URIs like **\-resource**, **\-service** etc.

Good example

```http
GET https://sddc.vmware.com/vcenter/vm/vm-104
```

Bad example

```http
GET https://sddc.vmware.com/vcenter/vm-resource/vm-104
```

### Hyperlinks

Important aspect of REST APIs is hyperlinking of resources. Hyperlinks allow clients to navigate the API with [limited or no out of band knowledge of the system](http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven). VMware APIs and the use cases we see from clients imply out of band knowledge of the API semantics and syntax. Thus the hyperlinks serve a secondary purpose for the API consumer. For example a hyperlink may navigate the consumer to particular node or service instance in a scaled out environment. Hyperlinks may also be useful to traverse a collection, locate parent resource and a number of other common use cases. Yet even in those cases compiled clients like SDKs and reflective clients like CLIs may find it easier to use application defined identifiers that are to be placed inside subsequent request in RPC manner instead of utilizing hyperlinks. The goal of this chapter of the standard is to set out syntax guidelines for rendering hyperlinks such that clients know how to find, understand the semantics of and use the hyperlinks in an API response.

> CNST-16: An API response MAY provide hyperlinks to related resources.

In defining the link syntax we want to leverage existing standards. There is not yet a widely accepted JSON standard for hyperlink representation. Following review of industry practices [Hypertext Application Language (HAL)](http://stateless.co/hal_specification.html) seems to come closest to a standard for JSON link representation.

> CNST-17: Links MUST be rendered within `_links` JSON object member.

> CNST-18: Links MUST be represented as object or array members of the `_links` object.

> CNST-19: The names of members of the `_links` object MUST use the name of the member they relate to or standard [IANA link relation names](https://www.iana.org/assignments/link-relations/link-relations.xhtml).

> CNST-20: Multiple links sharing the same relation value MUST be represented as array of link objects.

> CNST-21: Link objects MUST have `href` member containing target URL.

It may be beneficial to include additional data about a linked resources in hyperlinks. Such data may include the type of linked resource or properties of the linked resource needed for display purposes among others.

> CNST-22: Link objects MAY have additional members to complement their semantics.

Example:

```json
{
   "host": "host-12",
   "disks": ["IDE0:1", "IDE1:0"],
   "_links": {
      "self": {
         "href": "http://vc.coke.com/api/vcenter/vms/vm-42"
      },
      "host" : {
         "href": "http://vc.coke.com/api/vcenter/hosts/host-12",
         "title": "RDVC_ESX13"
         },
      "disks" : [ {
            "href": "http://vc.coke.com/api/vcenter/vms/vm-42/disks/IDE0:1",
            "id": "IDE0:1"
         }, {
            "href": "http://vc.coke.com/api/vcenter/vms/vm-42/disks/IDE1:0",
            "id": "IDE1:0"
         }
      ]
   }
}
```

> CNST-23: `href` values SHOULD be absolute URLs or absolute paths as defined by [RFC 1808 Section 2.2](https://tools.ietf.org/html/rfc1808#section-2.2).

Absolute URLs start with a scheme and authority (hostname, IP, port number and possibly authentication data). Absolute paths start with a slash and represent resource on the same endpoint.

For example:

```Absolute URL: http://cloud.vmware.com/sddcs/sddc-15```

```Absolute Path: /sddcs/sddc-15```

In case a service is deployed behind a reverse proxy whose hostname may be hard to figure absolute paths may work better. Absolute paths may be useful as resource identifiers that clients cache and are immune to changes of service hostname. Absolute URLs may be useful in scaled out environments to navigate the users between different shards of a service.

Use of paths relative to particular resource is discouraged. Those need to be interpreted in the context of a preceding request and require state on the client.

## HTTP Verbs

A key aspect of modern APIs is uniformity of the interface ([5.1.5 Uniform Interface; Architectural Styles and the Design of Network-based Software Architectures; R. Fielding, 2000](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm#sec_5_1_5)). When modeling a REST API the majority of design should be focused on modeling the resource representation and how these will be used to drive the application state. This is in sharp contrast to modeling classic RPC APIs that focus on modeling operations through which the application state is manipulated. In a REST API a consumer interacts with a resource representation through standard HTTP operations. So consumers need not to care as much what operations are available as they would do in a traditional system. Instead API consumers focus on the resource model, resource representations and navigation to related/linked resources. Resources accessed via standardized set of verbs form the basis for automation and policy driven systems. The following rules capture the above intent when mapped to HTTP:

> VERB-1: API SHOULD be structured around resources identified through nouns in a resource URI

> VERB-2: Interaction with the API SHOULD happen through manipulation of resource representations via standard HTTP verbs

Details about the patterns used with the various verbs follows

> VERB-3: APIs' contract MUST specify only one success response type for given request

A response type is the combination of response body schema, HTTP status code and headers. Multiple types of success responses for given operation require a client to implement multiple handlers for the success of each operation. For example if API implementations can either return JSON or CSV it should be specified by the client accept header or default for the API contract. Another example is an API that can return task or block until the actual result is produced, in this case the selection should be made based on API default or client parameter. In the later example it would be inappropriate for server to make the choice of task or blocking based on internally estimated duration.

> VERB-4: If a URI supports PUT, POST or PATCH then it SHOULD support HTTP GET

With very few exceptions all resource URIs should support HTTP GET operation. There may be rare exceptions for example a resource may support creation and lack list/query capability.

Modeling API interaction through standard CRUD (Create, Read, Update, Delete) operations eases integration with desired state automation tools like Chef, Puppet, Ansible, Microsoft Desired State automation as generation of the integration code may be used. Thus it is recommended to model fully functional resource abstractions that allow POST to create, GET to read, PUT or PATCH for update and DELETE.

> VERB-5: Operations with side effects SHOULD accept client operation identifier tokens. These tokens allow clients to reconnect to an operation when client fail to receive the original response.

The client token approach is employed by [Amazon EC2](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/Run_Instance_Idempotency.html) and [vSphere Content Library API](http://pubs.vmware.com/vsphere-60/topic/com.vmware.ICbase/PDF/vcs_java_prog_guide_6_0.pdf) to assure at most once semantics.

### GET

Retrieves a resource representation. Client requests a resource living on given URI and receives back a resource representation.

> VERB-6: GET request MUST retrieve resource representation

> VERB-7: GET operations MUST be safe to retry. (see [RFC 7231 Section 4.2.1 Safe Methods](https://tools.ietf.org/html/rfc7231#section-4.2.1) )

> VERB-8: GET requests SHOULD not expect request body

Safe to retry means that no significant side effects are exerted on the entity addressed by the request. It is possible that side effects are realized in form of log entries, last accessed time stamp etc.

Sometimes functionality like lease/lifespan of objects may be affected by last accessed time. For example upload session lifespan may be managed by tracking access to it. Using GET requests for this use case may be tricky as GET result may potentially be cached by some intermediate result. It is recommended to increase leases with a POST request.

Typical Patterns:

- **GET /{singleton}** - retrieve singleton representation e.g. /tagging/settings
- **GET /{resource collection}** - list/query entities e.g. /identity/api/tenants
- **GET /{resource collection}/{id}** - retrieve instance representation e.g. /identity/api/tenants/MYCOMPANY
- **GET /{resource collection}/{id}/{fragment}** - retrieve a fragment of a complex resource e.g. /vcenter/vm/vm-15/cpu
- **GET /{resource collection}/{id}/{sub-resource collection}** - list/query collection of sub-resources e.g. /identity/api/tenants/MYCOMPANY/directories.
- **GET /{resource collection}/{id}/{sub-resource collection}/{sub-resource id}** - retrieve a sub-resource instance e.g. /vcenter/vm/vm-15/disks/disk-2.

Success response status codes:

- **200 (Ok)** on success completion
- **304 (Not Modified)** when client has the latest version of the resource and caching is supported (See [RFC 7232 section 4.1](https://tools.ietf.org/html/rfc7232#section-4.1))

Example:

```http
GET /tagging/categories/c78b51b9-da37-44c9-9ac4-14ed788cc583
```

```http
HTTP/1.1 200 OK

{
 "id" : "c78b51b9-da37-44c9-9ac4-14ed788cc583",
 "associable_types" : [ "VirtualMachine" ],
 "description" : "Desc Cat",
 "name" : "MyCategory",
 "used_by" : [ ],
 "cardinality" : "MULTIPLE"
}
```

### PATCH

Creates or partially updates a resource. HTTP PATCH was introduced in [RFC 5789](https://tools.ietf.org/html/rfc5789) as an add-on to HTTP.

Developers of PATCH APIs may be concerned about the availability of HTTP PATCH in specific tools or deployment environments. In those circumstances alternative access to the HTTP PATCH functionality SHOULD be provided. One way to achieve this is through [POST (non-CRUD operations)](#post-non-crud-operations). Another example is using [HTTP POST and X-HTTP-Method-Override](https://cloud.google.com/compute/docs/instance-groups/updater/v1beta1/how-tos/performance#patch-alt-notation).

HTTP PATCH APIs are susceptible to concurrency issues when used in read-modify-write scenarios. It is recommended that [optimistic concurrency control](#concurrency) is employed to avoid mid-air collisions.

HTTP PATCH is the preferred way to update a resource as it solves several complications present in HTTP PUT. PATCH allows only relevant subset of properties to be updated thus minimizing the opportunity for conflicts. PATCH input schema can be extended as the product evolves. JSON body members that are not specified by the client in a partial update remain unchanged.

> VERB-9: PATCH requests MAY be non-idempotent.

> VERB-10: PATCH requests MUST accept resource identifier(s) in the URI path and a document describing the changes in the body of the request

> VERB-11: PATCH requests SHOULD on success return either 200 (OK) with updated resource document alternatively it MAY return 204 (No-content) with empty document OR 202 (Accepted) with a task (tracking resource) OR 201 (created) code when a new instance is created.

There are two general approaches to implement PATCH API in the industry that have gained prominence. The recommended in the general case practice is to use partial documents.

- **Accept partial document with only subset of the resource properties present. (RECOMMENDED)** This is described in [RFC 7386](https://tools.ietf.org/html/rfc7386). The server only updates the properties found in the request document. JSON null in this case is used to signal the server to reset a property. This approach is not quite suitable for updating arrays as indiviudal array elements cannot be addressed. One approach for handling arrays is to accept three lists — new, to be deleted and to be updated. This approach of partial documents is adopted by [VAPI Best Practices](https://wiki.eng.vmware.com/VAPI/Providers/VAPIBestPractices#There_is_a_difference_between_Set_and_Update._Use_the_correct_one) and [Google Compute Engine API](https://cloud.google.com/compute/docs/api/how-tos/performance#patch). The input for PATCH with partial document should resemble the output of a GET operation on the same resource.

- **Accept list of instructions for updating the resource.** This approach expects a list of statements each describing a modification to the resource representation e.g. update property X, unset property Y, add element to the end of a list, remove the third element etc. This approach to implementing PATCH makes it non-idempotent (e.g. remove third element). [RFC 6902 - JavaScript Object Notation (JSON) Patch](https://tools.ietf.org/html/rfc6902) and [VMware View](http://wiki.eng.vmware.com/View/ViewAPI/GettingStarted#Updates) employ this approach. This approach is more efficient if list manipulation is frequent use case. See [example update from RFC 6902](#partial-update-using-list-of-instructions).

Typical Patterns:

- **PATCH /{singleton}** — update singleton representation e.g. /tagging/settings
- **PATCH /{resource collection}/{id}** — update or create a resource instance e.g. /vcenter/vm/vm-15
- **PATCH /{resource collection}/{id}/{fragment}** — update a fragment of a complex resource e.g. /vcenter/vm/vm-15/cpu
- **PATCH /{resource collection}/{id}/{sub-resource collection}/{sub-resource id}** — update or create a sub-resource e.g. /vcenter/vm/vm-15/disks/disk-2.

Success response status codes:

- **200 (Ok)** in success update case when the updated resource representation is returned
- **201 (Created)** in case new resource has been created and its representation is returned
- **202 (Accepted)** when operation is performed asynchronously.
- **204 (No Content)** on success in case no message body is returned

Example update using partial document:

```http
PATCH /content/libraries/bbc7dd4f-0ece-4ad3-ae74-5547a0f2a5c9 HTTP/1.1
{
 "name" : "New library name"
}

HTTP/1.1 200 OK
```

### PUT

Creates or replaces a resource.

In accordance with [RFC 7231 section 4.3.4](https://tools.ietf.org/html/rfc7231#section-4.3.4) PUT performs full update (replacement) on a resource.

PUT APIs SHOULD consider the impact of concurrent modifications and employ appropriate [concurrency control mechanisms](#concurrency).

HTTP PUT APIs are not extensible as the product evolves i.e. no new optional members can be added to their input. This is significant limitation for complex APIs. It is thus recommended to use HTTP [PATCH](#patch) for updates to allow the resource schema to be extended in the future.

With HTTP PUT there is inherent risk of unrelated concurrent changes clobbering each other's effect. For example if 2 users make nearly concurrent updates to different properties of a resource it is possible that the changes of the first user are reverted by the second user. Optimistic locking may be employed to avoid collisions. HTTP standard headers ETag, If-Match may be used for optimistic locking. [HTTP PATCH](#patch) or fragment pattern allow for focused modifications within complex resources.

A common use case involving PUT APIs is read-modify-write. An API user fetches resource representation using HTTP GET modifies some of the resource properties and PUTs back the modified resource document. This pattern allows developers to quickly apply the API for routine tasks.

A subset of properties returned by HTTP GET may be read-only e.g. `resource identifier`, `revision` etc. HTTP PUT contract SHOULD be clear about handling of those properties. The general rule is to throw an error when attempt to modify read-only property is made. Throwing error will allow the contract to be updated in later version to allow modification of the property.

A special kind of read only properties are system properties exposed by the programming framework. Those properties may never become writable. Such properties may include things like `last_modified`, `revision` etc. Servers SHOULD ignore such properties on input to PUT. Further servers may want to use special syntax for the names of such properties to distinguish them from properties innate to particular resource. For example [NSX use underscore prefixed names](http://build-squid.eng.vmware.com/build/mts/release/bora-3788284/publish/nsx-manager/api.html#Methods.GetAssociations) for system properties. See also a [Xenon example response](#example-xenon-response)

```http
GET /api/realization-state-barrier/config
```

```json
{
  "resource_type" : "RealizationStateBarrierConfig",
  "interval" : 5000,
  "_revision" : 0,
  "id" : "564dab50-63a0-8b4f-a1f8-20e4d36efc3b",
  "_last_modified_user" : "admin",
  "_last_modified_time" : 1414088642536,
  "_create_time" : 1414088642536,
  "_create_user" : "admin"
}
```

PUT requests should adhere to the following guidelines

> VERB-12: PUT requests MUST update the whole resource configuration with the supplied configuration. Properties that are not included in the request MUST be reset to default value.

> VERB-13: PUT requests MUST be idempotent i.e. it is safe to re-try a request multiple times

> VERB-14: PUT requests MUST accept resource identifier(s) in the URI path and a resource document in the body of the request

> VERB-15: PUT requests SHOULD on success return either 200 (OK) with updated resource document alternatively it MAY return 204 (No-content) with empty document OR 202 (Accepted) with a task (tracking resource) OR 201 (created) code when a new instance is created.

> VERB-16: PUT request body SHOULD accept the response body of GET on the same resource.

> VERB-17: PUT APIs SHOULD document the semantics of fields — writable, ignored or unmodifiable.

Typical Patterns:

- **PUT /{singleton}** — set singleton representation e.g. /tagging/settings
- **PUT /{resource collection}/{id}** — set instance representation e.g. /vcenter/vm/vm-15
- **PUT /{resource collection}/{id}/{fragment}** — set a fragment of a complex resource e.g. /vcenter/vm/vm-15/cpu
- **PUT /{resource collection}/{id}/{sub-resource collection}/{sub-resource id}** — set a sub-resource e.g. /vcenter/vm/vm-15/disks/disk-2.

Success response status codes:

- **200 (Ok)** in success update case when the updated resource representation is returned
- **201 (Created)** in case new resource has been created and it's representation is returned
- **202 (Accepted)** when operation is performed asynchronously.
- **204 (No Content)** on success in case no message body is returned

Example:

```http
PUT /tagging/categories/c78b51b9-da37-44c9-9ac4-14ed788cc583 HTTP/1.1
{
 "associable_types" : [ "VirtualMachine" ],
 "description" : "Desc Cat",
 "name" : "MyCategory",
 "used_by" : [ ],
 "cardinality" : "MULTIPLE"
}

HTTP/1.1 200 OK
{
 "associable_types" : [ "VirtualMachine" ],
 "description" : "Desc Cat",
 "name" : "MyCategory",
 "used_by" : [ ],
 "cardinality" : "MULTIPLE"
}
```

### DELETE

Deletes a resource instance.

HTTP DELETE APIs are susceptible to concurrency issues and is possible for one user to delete an entity based on assumptions from recent read that are not true at time of deletion. It is recommended that [optimistic concurrency control](#concurrency) is employed to avoid mid-air collisions.

> VERB-18: DELETE requests MUST accept resource identifier(s) in the URI path

Delete calls may accept additional optional parameters, for example to set expiration date for purging the resource.

> VERB-19: DELETE requests SHOULD on success return HTTP code 204 (no content) on success (200 OK or 202 Accepted are also acceptable but not recommended)

> VERB-20: DELETE requests SHOULD not expect request body

Typical Patterns:

- **DELETE /{resource collection}/{id}** — delete a resource representation
- **DELETE /{resource collection}/{id}/{sub-resource collection}/{sub-resource id}** — delete a sub-resource instance e.g. /vcenter/vm/vm-15/disks/disk-2.

Success response status codes:

- **200 (Ok)** in success delete case when document is returned
- **202 (Accepted)** when operation is performed asynchronously and task (tracking resource) is returned.
- **204 (No Content)** on success in case no message body is returned

Example:

```http
DELETE /tagging/categories/bbc7dd4f-0ece-4ad3-ae74-5547a0f2a5c9 HTTP/1.1

HTTP/1.1 204 No Content
```

### POST (create)

Creates new resource instance.

> VERB-21: POST create request SHOULD on success return 201 (Created) code alongside resource document with populated auto-generated fields like ID, last update time etc. or 201 (Created) code alongside the ID of the new resource or 202 (Accepted) alongside a task (tracking resource) for asynchronous execution

> VERB-22: On success create operations SHOULD return a **Location** header with the URI of the new resource

Typical Patterns:

- **POST /{resource collection}** — create new instance
- **POST /{resource collection}/{id}/{sub-resource collection}** — create new instance of a sub-resource e.g. /vcenter/vm/vm-15/disks.

Success response status codes:

- **201 (Created)** in case new resource has been created and its representation is returned. It is acceptable to use 200 in legacy APIs.
- **202 (Accepted)** when operation is performed asynchronously and task (tracking resource) is returned.

Response Headers:

- **Location** header SHOULD specify the location of the primary resource created by this operation [RFC 7231 section 7.1.2](https://tools.ietf.org/html/rfc7231#section-7.1.2)

Example:

```http
POST /tagging/categories HTTP/1.1
{
 "name" : "Category name",
 "description" : "Category Description",
 "associable_types" : [ "Folder" ],
 "cardinality" : "SINGLE"
}

HTTP/1.1 200 OK
Location: http://sddc.vmware.com/tagging/categories/9580d673-442d-44de-8839-dd7778c6179a


{
 "id" : "9580d673-442d-44de-8839-dd7778c6179a"
}
```

### POST (non-CRUD operations)

As we model complex systems there is the occasional case where modeling interactions purely through manipulating resource representations may be inefficient or increase API complexity substantially.

Few examples that are not mapping well to resource property manipulation are:

1. **restart operation of a virtual machine** operation performs a state transition of a virtual machine from "powered on" through some intermediate state, like "off", back to "powered on" state. Typically REST APIs will accept desired end state as value for a given property and it is thus not natural to model restart as value of power state property. In particular integrations with desired state systems could be made more difficult.
2. **batch operations** act upon multiple resources
3. **complex queries** require sophisticated specifications of projection, filter and pagination that may not be practical to include in a query string

Industry experience also shows that sometimes modeling certain transition ought to happen through non-idempotent POST operation. (See [It is ok to use POST, R. Fielding, 2009](http://roy.gbiv.com/untangled/2009/it-is-okay-to-use-post))

Non-idempotent POST APIs may be susceptible to concurrency issues. Specific problem with non-idempotent APIs is that in case of network failure the client has to have way to recover through retrying the request or checking if the failed action has been enacted. It is recommended that [optimistic concurrency control](#concurrency) is employed to avoid mid-air collisions.

> VERB-23: APIs MAY model interactions through HTTP POST when modeling through resource manipulation is not natural

> VERB-24: Interactions modeled via HTTP POST MUST specify the operation as query string in the URI and accept parameters in the message body

Typical Patterns:

- **POST /{singleton}?action={action}** - perform custom action on a singleton
- **POST /{resource collection}?action={action}** - batch operation on a set e.g. /vcenter/vm?action=reset
- **POST /{resource collection}/{id}?action={action}** - perform custom action on a resource representation e.g. /vcenter/vm/vm-24?action=reset
- **POST /{resource collection}/{id}/{sub-resource collection}/{sub-resource id}?action={action}** - custom action on sub-resources e.g. /vcenter/vm/vm-15/disks/disk-2?action=snapshot.

Success response status codes:

- **200 (Ok)** in success completion case when document is returned
- **202 (Accepted)** when operation is performed asynchronously and task (tracking resource) is returned.
- **204 (No Content)** on success in case no message body is returned

To illustrate the above a good example for POST API that does not fit well into CRUD is as follows:

```http
POST https://{server}/vcenter/vm/{vm}/power?action=reset
```

Batch example:

```http
POST /vcenter/vm?action=reset
{
    "ids" : ["vm-15", "vm-234", "vm-111"]
}
```

We should avoid this pattern when API is better modeled via CRUD manipulation. Consider the following poor example that is much cleaner represented with HTTP DELETE:

```http
POST /vcenter/vm/{vm}?action=delete
```

## API Patterns

This chapter presents useful API modelling patterns.

### Long-running operations

Often back end processing to achieve certain goal will exceed acceptable time limits for an HTTP request. In those cases APIs must define their interaction model to return fast from individual HTTP transactions and allow clients to monitor progress and obtain final result asynchronously. There are several design patterns that are commonly employed depending on the use case and product specifics. These include tasks (tracking resources), resource status, desired/realized state. API implementations may offer one of those or a combination. For example NSX is offering mix of tasks (tracking resources) and Desired/Realized state.

There are some common good practices when exposing long-running operations

> PTRN-1: Clients MUST have an option to invoke a long-running API using one of the long-running operation models described above so that the client can implement for example, a responsive UI or pipelined automation script.

From API implementer's view it is hard to assess the duration and reliability of given API operation as APIs may evolve to have more complex processing in future versions or clients may be on particularly unstable networks. It is thus expected that all mutation APIs should provide the option to start an operation with initial call and poll for progress and completions using subsequent requests. Certain APIs may provide additional option to wait for the result on the first request while others may only have the polling model.

> PTRN-2: Long-running APIs SHOULD perform as much as possible validation and report most errors during the initial interaction.

#### Resource State

One way to track completion of long-running operations on a resource is to provide resource state field or fragment that indicates if the resource is undergoing reconfiguration. This technique is used by [OpenStack Nova API](http://developer.openstack.org/api-guide/compute/server_concepts.html#server-status), [NSX API](http://build-squid.eng.vmware.com/build/mts/release/bora-3788284/publish/nsx-manager/api.html#Methods.GetLogicalSwitchState).

A simple example of this type of APIs is OpenStack Nova server (virtual machine) provisioning. (See [full example](#example-openstack-nova-server-provisioning))

1.  End user requests new server via `POST /v2.1/servers`

```http
POST http://os.vmware.com/v2.1/servers
```

2.  OpenStack returns skeleton object with self link to the server resource

```http
202 Accepted
Location: http://os.vmware.com/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb

{
  "server":{
     "id":"b85ebd3b-df19-47f7-8116-ce99f00363cb",
     "links":[
        {
           "href":"http://os.vmware.com/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
           "rel":"self"
           .........
```

3.  The server is in `BUILD` state

```http
GET http://os.vmware.com/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb

200 OK

{
  "server":{
     "status":"BUILD",
     . . . .
  }
  . . . .
}
```

4.  Actual provisioning completes and server goes to `ACTIVE` state or if creation fails the server goes to `ERROR` state.

```http
GET http://os.vmware.com/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb

200 OK

{
  "server":{
     "status":"ACTIVE",
     ........
  }
}
```

In order to use this technique for create operations the back end services should be able to swiftly validate the request, allocate resource identifier, persist the entity description and return to the client. It is still possible that the actual creation in the backing system may fail due to conditions that cannot be validated upfront. If create failure is to occur the resource should be moved to ERROR state and the API client is then responsible to delete it or modify their requirement such that system can fulfill it.

The state property may be included inside the resource document (OpenStack) or in a fragment (NSX). The fragment approach has the advantage that the status properties are cleanly separated from the desired state properties of a given resource.

Example of state fragment for logical switch from NSX API

```http
GET https://<nsx-mgr>/api/logical-switches/cc5ff938-6f09-4841-8f0f-294e86415472/state

Successful Response:

{
  "state" : "in_progress",
  "logical_switch_id": "cc5ff938-6f09-4841-8f0f-294e86415472",
  "details": [
    {
      "state" : "in_progress",
      "sub_system_id" : "366048ba-89d9-435e-ac2e-2c7cf6ed0f33",
      "sub_system_type" : "TransportNode"
    }
  ]
}
```

#### Tasks

This is the most widely used approach at VMware for dealing with long-running operations also employed by [Google Compute Engine](https://cloud.google.com/compute/docs/reference/latest/zoneOperations), some [Microsoft](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#1325----operation-resource) and some [Amazon APIs](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_ImportInstance.html). The essence of the approach is to return HTTP 202 code and a tracking task resource to the API client after some basic request validation. Further the client can obtain progress updates and final operation status from the tracking resource.

> PTRN-3: Operation contracts MUST specify whether the operation return type is task or actual result.

An API can provide two operations with different HTTP request signatures for API consumers to select between task and actual result return types.

```http
// Operation that will block the request until completion
POST /vm?action=power-off

// Operaiton with task
POST /vm?action=power-off&vmw-async=true
```

There are two ways to include tasks in operation contracts

1. **Client preference** in NSX API clients may request task to be used for any API operation by passing additional "vmw-async" query parameter in the request. This model allows API clients to assess the risks of network issues or ability to retry given call and make choice if the added overhead of polling is worth it.
2. **API designer preference** for some APIs the complexity of an operation may be such that it will rarely complete in one HTTP request/response cycle. In those cases API designer may declare an API to explicitly return only tasks and not have option of execution in single request/response.

> PTRN-4: The task resource SHOULD contain at minimum the following information:
>
> - state of the operation - at minimum the following states need to be present > - running, success, error
> - creation time of the task
> - completion time of the task
> - error object if the task execution fails
> - identifier of the resource upon which the tracked operation is working.
> - the operation-specific result of the invoked operation

The results of many write operations in REST API are void or single resource identifier. In those cases pointer to the resource upon which the task operation is carried out suffices. Clients can use subsequent `GET` operation on the modified resource to learn more.

Some REST operations produce actual return payload e.g. complex queries (reports). When those are executed using tasks the response field is to contain the actual operation result. The type of object included in the response should be identified with a type discriminator, so API clients know how to interpret it (see [Objects](#objects)). The result may be embedded in the task resource or provided as separate fragment resource e.g. `GET /api/tasks/{task-id}/response`. When the result is embedded in the task resource the task read operation may allow clients to request or suppress inclusion of result field.

Example of task from NSX API

```http
Request:
GET https://<nsx-mgr>/api/tasks/ab265781-c826-4da7-9487-48a5c713a481

Response:
200 OK

{
  "progress" : 100,
  "id" : "ab265781-c826-4da7-9487-48a5c713a481",
  "end_time" : 1416959364977,
  "status" : "success",
  "async_response_available" : false,
  "cancelable" : false,
  "start_time" : 1416959362874
}
```

Example of task response fragment from NSX API for a Transport Zone creation operation. The `resource_type` field tells the client the type of entity returned

```http
Request:
GET /api/tasks/860f8b82-7983-4274-b275-ccdebb666ec7/response

Response:
{
    "resource_type": "TransportZone",
    "description": "Transport Zone 1",
    "id": "a3097a9b-6429-499d-af21-7ae2bea5b1b9",
    "display_name": "tz1",
    "host_switch_name": "test-host-switch-1",
    "transport_type": "OVERLAY",
    "transport_zone_profile_ids": [
        {
            "profile_id": "52035bb3-ab02-4a08-9884-18631312e50a",
            "resource_type": "BfdHealthMonitoringProfile"
        }
    ],
    "_create_time": 1474469755392,
    "_last_modified_user": "admin",
    "_system_owned": false,
    "_last_modified_time": 1474469755392,
    "_create_user": "admin",
    "_revision": 0,
    "_schema": "/api/schema/TransportZone"
}
```

List of recent tasks is often displayed in user interfaces.

> PTRN-5: Service MAY implement task collection resource. The task collection SHOULD allow sorting and filtering tasks.

Example from NSX API

```http
GET /api/tasks
```

> PTRN-6: As long-running operations may be taking up valuable system resources API to request task cancellation MAY be implemented

It is up to the backend system to decide if task cancellation is to be honored or not.

Example from NSX API

```http
POST /api/node/tasks/{task-id}?action=cancel
```

Additional utility APIs for easier communication of the task changes may be implemented for example push APIs or long poll APIs.

See [NSX Task Management API](https://www.vmware.com/support/nsxt/doc/nsxt_20_api.html?ClickID=asltttsakzzlnwp9lzs0sl550rtata0onlnw#Sections.Api%20Services.Task%20Management)

### File Transfers

It is not uncommon for control and management APIs to require file or data transfer functionality. Such use cases include certificate files, configuration files, installation images, screenshots etc. This chapter contains guidance how to expose file & large data transfer functionality in standard compliant and uniform manner.

File transfer API is alternative to embedding base64 encoded binary content inside API JSON documents. As a rule of thumb large pieces of data need to use file transfer. File transfer semantics allow developers to recognize the demand for optimizing memory use. For example REST client implementations may assume JSON payloads fit in RAM while file transfers be streamed to disk before processing.

> PTRN-7: Binary data larger than several KiBs (at most 100KiB) MUST use file transfer pattern instead of embedding content in JSON

#### File or Data Download

Download is natural for the HTTP protocol and REST APIs. Download functionality is implemented by setting the appropriate media type of response and returning the file payload in the response body.

Depending on the requirements download could be initiated with [GET](#get) or [POST (non-CRUD)](#post-non-crud-operations) operation.

> PTRN-8: Downloads SHOULD be tied to GET or POST (non-CRUD) operations

Download of multiple pieces of data or files can be achieved using popular archive format like zip or by exposing a resource containing hyperlinks to downloadable files.

#### File or Data Upload

Uploads are typically bound to HTTP [PUT](#put) or [POST (non-CRUD)](#post-non-crud-operations) operations.

> PTRN-9: Uploads SHOULD be tied to PUT or POST (non-CRUD) operations

> PTRN-10: Upload endpoints SHOULD accept requests with the appropriate media types e.g. `application/octet-stream`, `image/jpeg`, `application/xml` etc.

Example:

```http
PUT /vcenter/nsx/appliance/image
content-type: application/octet-stream
```

> PTRN-11: Multiple file upload APIs SHOULD use POST and `multipart/form-data` media type as defined in [RFC 7578](https://tools.ietf.org/html/rfc7578#section-4.3).

Example:

```http
POST /vcenter/nsx/appliance/images
content-type: multipart/form-data; boundary=----content-273def1ef51c
content-length: 12345671

------content-273def1ef51c
content-disposition: form-data; name="files"; filename="settings.xml"
content-type: application/xml
content-length: 45670

[[File content here]]
------content-273def1ef51c
content-disposition: form-data; name="files"; filename="image.ova"
content-type: application/vmware
content-length: 12299800

[[File content here]]
------content-273def1ef51c
```

See also:

- [OpenAPI 3.0 — considerations for file uploads](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#considerations-for-file-uploads)
- [OpenAPI 3.0 — considerations for multipart content](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md#special-considerations-for-multipart-content)
- [OpenAPI file upload guidelines](https://swagger.io/docs/specification/describing-request-body/file-upload/)
- [OpenAPI multipart request guidelines](https://swagger.io/docs/specification/describing-request-body/multipart-requests/)

### API Errors

> PTRN-12: Errors reported by APIs MUST use the HTTP Response Status Code semantics [RFC 7231 Section 6 HTTP Response Status Codes](https://tools.ietf.org/html/rfc7231#section-6)

Errors reported through the API must respect the HTTP Response Code semantics. This guarantees that widest range of clients can handle the error conditions in appropriate manner such as report the error to the end user, retry the operation later, remove non-existent entries from cache etc..

There may be need for some additional consideration when selecting an error code. For example to prevent leakage of information about resource existence **404 Not Found** error may be reported when user lacks visibility of a resource. **403 Forbidden** may be reported when user has visibility of a resource and specific action on the resource is disallowed e.g. deleting the resource. (See [RFC 7231, Section 6.5.3. 403 Forbidden](https://tools.ietf.org/html/rfc7231#section-6.5.3))

> PTRN-13: Error payload SHOULD contain additional error identification details

In many scenarios the HTTP response code semantics are insufficient to communicate in sufficient detail the nature of the error that has occurred. API authors should add a member in the error payload that disambiguates such conditions.

> PTRN-14: Error payload SHOULD contain appropriate error messages

VMware software is often used in data center context by system administrators and when errors occur it is important to report sufficient detail about the error to enable troubleshooting. Thus it is important to include human readable messages in the error payload.

Localization is particular concern with error messages in APIs. With REST APIs at VMware we have adopted several distinct approaches to localizing error messages:

- **API Server** localizes - in this case the client sends desired locale in a **Accept-Language** header. The server generates the messages in the desired locale and sends them back. (See [vCloud Air API - vCloud API REST Requests](https://pubs.vmware.com/vca/index.jsp?topic=%2Fcom.vmware.vcloud.api.doc_56%2FGUID-0EAF1523-CA02-41FC-A7F3-B103108F5E4A.html))
- **API client** localizes - in this case the API returns message keys and parameters. The API client has to download localization bundles, look up the received message keys and format the parameters in the message.
- **Combined** (recommended) - In this case the API returns both localized messages, message keys and parameters necessary for delayed localization

Server side localization is obvious choice when creating system administrator focused user interfaces. This approach is not very usable if the API is invoked through a workflow solution that may need to report errors at a later time possibly in a yet to be determined locale.

To cater for the complex cloud stack that includes workflow solutions client side localization is useful. For example vRealize Automation can store messages from vCenter and ESX for a specific failed service request. vRA could then present the error messages to a system administrator in their desired locale at later point of time.

The client side localization approach makes it simpler to code service oriented application where the actual execution and potential error messages may originate from various services.

Thus we suggest that VMware REST APIs allow for both mechanisms to exist simultaneously. It is acceptable to have the server side localization done in one locale fixed per product or deployment.

> PTRN-15: Error payload MAY contain additional data relevant to the error

Error handling code often depends on certain supplementary data. Such data may at the discretion of the API designer be added to the error payload and SHOULD be appropriately documented.

Below is example error response following the above rules:

```http
POST /tagging/tag HTTP/1.1

{
   "name" : "test"
}


HTTP/1.1 400 Bad Request

{
    "messages": [
        {
            "id": "vapi.invoke.input.invalid",
            "default_message": "Unable to validate input to method com.vmware.cis.tagging.tag.create",
            "args": [
                "com.vmware.cis.tagging.tag.create"
            ]
        },
        {
            "id": "vapi.data.structure.field.invalid",
            "default_message": "Found invalid field create_spec in structure.",
            "args": [
                "create_spec"
            ]
        },
        {
            "id": "vapi.data.structure.field.missing",
            "default_message": "Structure 'com.vmware.cis.tagging.tag.create_spec' is missing a field: description",
            "args": [
                "com.vmware.cis.tagging.tag.create_spec",
                "description"
            ]
        }
    ],
    "type": "com.vmware.vapi.std.errors.invalid_argument"
}
```

#### Error Response Codes

[Section 6.5 of RFC 7231](https://tools.ietf.org/html/rfc7231#section-6.5) provides explanations of which response codes are appropriate in various error conditions. The RFC texts are domain agnostic and on many occasions further guidance to the use of HTTP codes is required. This section of the standard further clarifies the use of HTTP response codes for API design at VMware by discussing domain specific scenarios.

In the absence of explicit guidance, RFC 7231 is considered authoritative.

> PTRN-16: APIs MUST respond with 403 when functionality is unavailable due to licensing

If an API client makes a request requiring functionality that is available in some instances of the service but is not available in the requested instance because the current license does not support it, use a 403 response to indicate that access to the feature is currently forbidden.

## API Documentation

> DOC-1: There MUST be an API guide for the product which is delivered via the company web site without requiring access to the product so a customer can review the API without access to the product itself (i.e. on an airplane).

> DOC-2: There MUST be API documentation delivered as part of the product. This SHOULD be delivered as a URI available on the product's management system web server and SHOULD be linked from the help section of the GUI.

> DOC-3: The intended audience of the API guide are developers or operators intending to automate or integrate. The API guide is generally considered a reference document describing how to use the API and any relevant contracts about the behavior of the API. The API guide SHOULD NOT be a detailed document about how a feature or component is used and how it inter-operates with other system capabilities. This information is useful to the overall user base and SHOULD be in the administration guide.

> DOC-4: API examples MAY be delivered as part of developer network code examples and MAY be in the API guide.

> DOC-5: To save in effort and reduce the chance of errors the API guide MUST be automatically generated upon the build of the product.

> DOC-6: The API guide MUST document all publicly facing API resources and methods.

> DOC-7: There SHOULD be a general "how to use the API" section at the start of the API Guide document.

> DOC-8: There MUST be a way to search the API guide. If delivered in a chapter format this search MUST search across all chapters. If a single page in browser then the search MAY use the built in browser search. If delivered in a PDF then the search MAY use the built in PDF reader search.

> DOC-9: All deprecated API resources, methods, call, etc MUST be documented as such.

> DOC-10: Each resource SHOULD have a "released in" or "introduced in" field in the documentation.

> DOC-11: All API docs that are written by development SHOULD be approved by the documentation team.

> DOC-12: API doc styling SHOULD match the corporate standards in terms of colors, fonts, borders, etc.

> DOC-13: Some API documentation MAY be considered private or for registered partners only. These APIs MUST be documented in a separate API guide(s) and released only through appropriate channels i.e. the VMware developer center web site.

> DOC-14: The API guide SHOULD describe all error codes that could be experienced when operating the product and an associated description of the error code.

## API Evolution

The fundamental goal is that evolving an API and its implementation should not break existing client code that is being used by VMware's customers or other VMware products. The value of VMware's products stems from operating together in an integrated environment with other VMware and partner products. When VMware releases a new product version that breaks this integrated environment customers will not be able upgrade. Such incidents are costly for customers, partners, and VMware.

The best way to avoid breaking of existing client code is to only make backward compatible changes to the API definition and its implementation. Those changes include:

- Adding new resource URI
- Adding new HTTP verb on existing resource URI
- Adding new optional parameter (member of JSON body, header or query parameter)
- Adding new response element (member of JSON body, header parameter)

Note that while many "additive" changes (e.g. new resources, new operations) are clearly backward compatible, additions to data structures (request/response bodies in REST) are a little more subtle. Additions to output data structures (response bodies in REST) are backward compatible, assuming that client code (including the de-serialization code used by the stubs) will ignore data they don't understand. Additions to input data structure (request bodies in REST) must be optional additions. Additions to HTTP PUT body data structures even when optional may introduce backwards incompatible behavior. In other words requests from old clients (that won't include the additions)must be accepted by the server and treated exactly as they were earlier.

While we strongly discourage evolving the API and its implementation in ways that are not backward compatible and it should be avoided when there are alternatives, there are situations where there is no other alternative. To avoid breaking existing client code that is being used by VMware's customers when making incompatible changes to the API definition and its implementation, we need to communicate our intentions to our customer and partners and stage the changes in a way that allows them to migrate with us. Typically this involves:

- Making the new API available so that client developers can use it, while continuing to support the existing API.
- Documenting that the existing API is being deprecated and the the new API should be used instead
- Continuing to support the existing (deprecated) API for a period of time sufficient for the developers of the client code to switch to the new API and release new versions of the clients that don't use the deprecated API.
- Remove support for the deprecated API.

Thus the formal requirements are:

> VER-1: API changes between versions MUST only include (1) Expanding the API surface in backwards compatible manner AND (2) Removing deprecated APIs when absolutely necessary and approved by product management.

When an application is pointed to a REST API endpoint it is important for the application to know the version of API it is interacting with so as to select appropriate behavior. For example an API client built against release 7 has to avoid APIs added in release 7 when talking to release 6. The standard practice at VMware is to expose a resource that contains the version of the product and its API ([See Multiple Versions of vSphere API](http://pubs.vmware.com/vsphere-6-5/index.jsp?topic=%2Fcom.vmware.wssdk.pg.doc%2FPG_Client_App.5.6.html))

> VER-2: APIs MUST expose a resource that clearly identifies the current API version

Removing APIs introduces risk to product interoperability. It should be based on careful business analysis possibly using telemetry data. In essence removal of an API however unimportant it may seem makes the whole system API incompatible with prior releases and renders all existing VMware and 3rd party solutions incompatible until re-certified.

### Dealing with incompatible changes

Sometimes substantial changes need to be made to existing resources that cannot be represented only as adding new properties and/or operations to existing resources. Such changes should be introduced as new APIs while the existing APIs preserve their compatible syntax and behavior.

Introducing such new APIs depending on the scope of change may be limited to a single resource or as big as introducing new API for a given component.

> VER-3: Incompatible changes MUST be exposed under new URI and http verb combinations there by preserving the compatibility of prior APIs. Incompatible changes MUST NOT be introduced under same URI and verb combination using HTTP header.

For example if one needs to introduce incompatible change to virtual machine migrate `POST /vm/{vm-id}?action=migrate` one should implement a new operation e.g. `POST /vm/{vm-id}?action=relocate`. This approach has been used widely in the industry e.g. Java introduced `ArrayList` in order to augment in incompatible way the behavior of `Vector`.

In a different example if the vCenter content library component is to replace the local library concept with subscribable library leading to ripple changes to many other bits of the API it may be better to introduce the changes under new path as a brand new API set. For example the transition could be from `/content/local-library` to `/content/subscribable-library`. This later example should be used only in cases where major product changes are to happen e.g. merging two product lines. It is not acceptable to wholesale clone APIs for minor changes e.g. new fields to few resources. Industry example of this approach are Microsoft Azure APIs where the _classic_ APIs were improved in the _normal_ APIs. Another industry example is the evolution of Jackson API where new versions use new name space.

### API Versioning

The goal of API versioning is to help managing the impact of breaking changes. When a large scale re-architecture of a product is taking place it is likely that impact on API will span multiple resources. API providers may choose to host the new APIs under a new semantically named hierarchy. Alternatively, often when dealing with a natural iteration of an existing API, API Providers may choose to adopt a versioning scheme on top of the existing semantical hierarchy.

Following the general intent of this Standard, REST API versioning strives for achieving alignment across VMware products, while minimizing the effort required for adopting the new guidelines. This means that, in areas where there is more than one acceptable option, the Standard is biased towards the option that requires the least effort for adoption. As it currently stands, majority of VMware products either do not version REST APIs or version REST APIs using a versioning scheme, where the version information is present in a URI path segment, also referred as URI-based versioning. URI-based versioning is accepted as a standard pattern by multiple companies, such as Google and Microsoft, and is widely adopted by a large population of REST APIs. Hence, this Standard calls for adoption of URI-based versioning.

> VER-4: REST APIs SHOULD include version information in a URI path segment by using the lowercase letter 'v' followed by a string representing the REST API version.

Using versions in URLs from the beginning is not required. APIs can adopt URI-based versioning whenever the need for that arises. API providers are not required to introduce versioning into existing APIs merely to comply with the guidelines. Instead, it is recommended to align with the versioning guidelines when a backward incompatible API change is introduced.

> VER-5: Released/GA versions of REST APIs MUST be represented by a single integer following the lowercase letter 'v', for example, 'v2'.

Early/pre-GA versions may follow these conventions to clarify the in-development state of an API, for example:

version

description

v2alpha

Alpha release of version 2

v2beta1

Beta 1 release of version 2

> VER-6: Existing REST APIs that are not versioned MUST be considered version 1 (v1). If a new incompatible change is introduced using URI-based versioning it SHOULD start from version 2 (v2).

This is to accept the possibility that the existing, non-versioned API can be perceived by API consumers as version 1. Hence, the next explicitly versioned API should be bumped up to version 2 to avoid confusion.

> VER-7: REST APIs MUST NOT use a different version negotiation mechanism, such as Accept and Content-Type headers or query parameters.

The reason behind this is alignment and bias towards minimization of effort, as well as compatibility with commonly used interface definition languages (IDLs) and frameworks, such as VMODL2, OpenAPI 2.0 and 3.0. As mentioned above, most of the existing APIs either use URI-based versioning or do not version REST APIs. Alternative versioning approaches, such as supplying version information using a query parameter, are not natively supported by IDLs, such as OpenAPI 3.0.

API providers are expected to align with the versioning standard when a new backward incompatible version is introduced. In such situations, it is the responsibility of the API provider to collaborate with the SDK team on communicating the change of the API versioning scheme to the developer community.

> VER-8: REST API versions MUST NOT be coupled with product or implementing service versions.

REST API version is expected to change out of absolute necessity, at a much slower pace than product or implementing service versions. An example of that could be the necessity to introduce a breaking change or completely re-architect an existing API. These conditions are expected to be extremely rare as the REST API matures and gets adoped by consumers. In typical scenarios changes to APIs are expected to be backward compatible and should not require REST API version changes.

> VER-9: REST APIs URIs SHOULD be formatted in the following way: `http[s]://<authority>/[api/][<namespace>/][<version>/]<path>`
> Where:
>
> - authority - `[userinfo@]host[:port]`. A single authority can be shared among multiple API providers.
> - namespace - (optional) An arbitrary name pre-registered by an API provider as a grouping for a subset of APIs.
> - version - The lowercase letter 'v' followed by a version identifier, for example: 'v3'.

Namespaces may be omitted if the feature set is uniquely identified by the authority and the entire set of REST APIs hosted under the root level is expected to be versioned together.

> VER-10: REST API URIs SHOULD use `/api` as a URI path segment if explicit separation of API paths from non-API paths is required.

The use of `/api` URI path segment is not necessary if such separation is clear from the API root or the namespace. For example:

```http
    https://api.vmware.com/namespace/v1/path
```

If the base URL is shared between API and non-API resources, the use of the `/api` URI path segment is recommended.

```http
    https://vmc.vmware.com/api/namespace/v1/path
```

> VER-11: REST API versioning MAY be managed at an individual namespace level.

Considering the differences in product and feature release cadence, some products or services may run into the need to change the REST API version while others may not. Namespaces provide a level of separation between groups of APIs.

APIs under a specific namespace can be versioned independently. It is expected to have varying versions of APIs behind the same authority, for example:

```http
https://gateway.vmware.com/api/namespace-a/v1/policies
https://gateway.vmware.com/api/namespace-b/v2/resources
```

> VER-12: Any version of a REST API MUST be complete and self contained and MUST NOT depend on other versions of the same REST API.

When a new version of a REST API under a specific namespace is introduced, it must include all the features and capabilities and must not rely on the existence or make references to a different, such as older, version of the same API. APIs under different namespaces are versioned independently and can refer to each other.

## API Breaking Changes and Deprecation

APIs are the capability that make a product larger than itself. They are the gateway to becoming an integrated part of a much larger ecosystem. This not only represents a large strategic value to VMware, but also a substantial risk to our customer's ecosystem that relies on our APIs to run their business. Customers also recognize this importance which is why there is an ever-increasing demand for a consistent deprecation (including all breaking changes) experience across all VMware products and services that they use. Because of this strategic need, VMware has standardized (via [the BOSS Forum](https://confluence.eng.vmware.com/x/4mF1M)) on an universal, enterprise-wide API deprecation policy as summarized below. The formal standardization overview with all the details can be found [here](https://confluence.eng.vmware.com/x/8qO6M). This standard puts VMware in-line with industry practice of 12 months as seen in AWS, GCP and the Kubernetes Project. In future we may strengthen this promise to match VMware peers such as Azure and Salesforce.com with as long as 2 year deprecation periods.

> DR-1: Public APIs MUST be registered with developer.vmware.com along with associated lifecycle phase for each endpoint.

This enterprise-wide requirement ensures that customer-facing APIs are clearly presented to customers and sets the appropriate expectation around the lifecycle phase of each API. This also allows product teams to clearly deliniate between Generally Available (GA)/Supported APIs, and those in an Unsupported/Pre-GA phase such as Beta or Experimental.

> DR-2: A 12-month advance End-of-Life (EOL) notice MUST be given for all breaking changes on APIs in a Supported/Generally Available (GA) state.

With the clarity around which APIs are Supported/Generally Available (GA), VMware will be delivering contractual assurances to customers of 12-months advance notice before introducing a breaking change to all Supported/GA APIs. This advance notice will be limited to Supported/GA APIs; there is no guarantee associated with Pre-GA/Unsupported/Experimental APIs to allow product teams engineering agility early in the lifecycle of each API.

> DR-3: Customer notification of pending End-of-Life (EOL) or breaking changes MUST be given via Product Line Release Notes which starts the 12-month period

While notice of API deprecation inside of an OpenAPI/Swagger Spec or other method is welcomed, there is a baseline requirement to notify VMware customers of pending API deprecation in the Product Line Release Notes. Publication of this document starts the clock on the 12-month period before introducing the breaking change.

> DR-4: Responses from deprecated APIs MAY use the `Deprecation` HTTP header to signal their API lifecycle status

In addition to the above VMware wide policy authors of REST APIs may use the `Deprecation` and related headers defined in [The Deprecation Header Field](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-deprecation-header) candidate RFC to convey details about the API deprecation status and expected sunset date.

Example:

```
  Deprecation: Sun, 11 Nov 2020 23:59:59 GMT
  Sunset: Wed, 11 Nov 2021 23:59:59 GMT
  Link: <https://api.example.com/v2/customers>; rel="successor-version",
        <https://developer.example.com/deprecation>; rel="deprecation"
```

_**Note**_: Once the RFC is approved this guideline will change to SHOULD and be updated according to the published RFC.

_**Note**_: This requirement is not part of the VMware enterprise policy.

## Appendix A - Reviewer Committee

### v 1.0

|                      |                      |                  |
|----------------------|----------------------|------------------|
| Christos Karamanolis | Mitesh Pancholy      | Alan Renouf      |
| Gordon Good          | Greg Burk            | Kostadis Roussos |
| Jim Stabile          | Chris Wagner         | Jeff Hu          |
| Alain Dumesny        | Pallavi Pangarkar    | Asaf Kariv       |
| Ken Ringdahl         | Andrew Voltmer       | Shaomin Chen     |
| Dale Olds            | Gopala Suryanarayana | Robert Bosch     |
| Christophe Decanini  | Emad Benjamin        | William Lam      |
| Yashika Deva         |                      |                  |

### v 2.0

|                    |                    |                  |
|--------------------|--------------------|------------------|
| Tim Binkley-Jones  | Jeff Hu            | Gordon Good      |
| Kashfat Khan       | Keith Farkas       | Yashika Deva     |
| Yavor Boychev      | Ravi Soundararajan | Tim Whiffen      |
| Steven McAllister  | Karthik Murthy     | Brian Williams   |
| Dylan Thomas       | Fanny Strudel      | Kostadis Roussos |
| Christian Dickmann | Jason Chu          | Suren Balikyan   |
| Karthik Seshadri   | Eduard Zakaryan    | Zahari Ivanov    |
| Ina Uzunova        | Alex Rankov        | Eddie Dinel      |
| Patrick Barker     | George Dimitrov    | Peter Canning    |
| Martin Cvetanov    |                    |                  |

Document Editors - Kiril Karaatanassov, Avetik Hovhannisyan

## Appendix B - Additional Examples

### Partial Update Using List of Instructions

Example for partial update using list of instructions from [RFC 6902](https://tools.ietf.org/html/rfc6902#appendix-A.4)

```markdown
An example target JSON document:

   { "foo": [ "bar", "qux", "baz" ] }

A JSON Patch document:

   [
     { "op": "remove", "path": "/foo/1" }
   ]

The resulting JSON document:

   { "foo": [ "bar", "baz" ] }
```

### Example Xenon Response

The `document*` fields are system properties in the following Xenon response. Those are generated by the framework.

```json
    {
       "enumerationServiceReference":"http://172.20.0.221:62041/common/node-groups/default",
       "enumerationAgentReference":"jsonrpc://172.20.0.221:62042",
       "systemInfo":{
          "properties":{

          },
          "environmentVariables":{

          },
          "availableProcessorCount":0,
          "freeMemoryByteCount":0,
          "totalMemoryByteCount":0,
          "maxMemoryByteCount":0,
          "ipAddresses": [
             "172.20.0.221"
          ]
       },
       "status":"UNKNOWN",
       "serverId":"87084a1d-8d77-4295-aef4-f6bb957995d1",
       "isRefreshRequired":false,
       "documentSignature":"4322961ce952655c9ba7345e448e556a602e2c934c6f56ee72d1c07fb85907fe",
       "documentUpdateTimeMicros":1407472558278011,
       "documentVersion":0,
       "documentKind":"com/dcentralizedsystems/services/peernodestate",
       "documentSelfLink":"/common/node-groups/default/87084a1d-8d77-4295-aef4-f6bb957995d1"
    }
```

### Example OpenStack Nova Server provisioning

Example Contributed by Radoslav Gerganov (rgerganov@vmware.com)

#### CREATE SERVER REQUEST

```http
POST http://10.26.33.242:8774/v2.1/servers
Accept: application/json
User-Agent: python-novaclient
OpenStack-API-Version: compute 2.35
X-OpenStack-Nova-API-Version: 2.35
X-Auth-Token: {SHA1}4c9fd04a28c8b1df81de0689cbda4e2054439132
Content-Type: application/json

{
   "server":{
      "min_count":1,
      "flavorRef":"1",
      "name":"fira12",
      "imageRef":"83872a57-51e1-4546-b26f-bc73ebc5d26e",
      "max_count":1
   }
}
```

RESPONSE:

```http
202 Accepted
Content-Length: 376
Location: http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb Content-Type: application/json
Openstack-Api-Version: compute 2.35
X-Openstack-Nova-Api-Version: 2.35
Vary: OpenStack-API-Version, X-OpenStack-Nova-API-Version X-Compute-Request-Id: req-53098894-aad9-4541-96b2-d4e886be88c0
Date: Tue, 18 Oct 2016 12:13:40 GMT
Connection: keep-alive

{
   "server":{
      "security_groups":[
         {
            "name":"default"
         }
      ],
      "OS-DCF:diskConfig":"MANUAL",
      "id":"b85ebd3b-df19-47f7-8116-ce99f00363cb",
      "links":[
         {
            "href":"http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"self"
         },
         {
            "href":"http://10.26.33.242:8774/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"bookmark"
         }
      ],
      "adminPass":"4Y3xdtLb8qAN"
   }
}
```

#### POLL SERVER UNDER CREATION

```http
GET http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb
OpenStack-API-Version: compute 2.35
User-Agent: python-novaclient
Accept: application/json
X-OpenStack-Nova-API-Version: 2.35
X-Auth-Token: {SHA1}4c9fd04a28c8b1df81de0689cbda4e2054439132"
```

RESPONSE:

```http
200 OK
Content-Length: 1667
Content-Type: application/json
Openstack-Api-Version: compute 2.35
X-Openstack-Nova-Api-Version: 2.35
Vary: OpenStack-API-Version, X-OpenStack-Nova-API-Version
X-Compute-Request-Id: req-c87efe63-c714-440e-9eda-951332b4e635
Date: Tue, 18 Oct 2016 12:13:41 GMT
Connection: keep-alive

{
   "server":{
      "OS-EXT-STS:task_state":"scheduling",
      "addresses":{

      },
      "links":[
         {
            "href":"http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"self"
         },
         {
            "href":"http://10.26.33.242:8774/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"bookmark"
         }
      ],
      "image":{
         "id":"83872a57-51e1-4546-b26f-bc73ebc5d26e",
         "links":[
            {
               "href":"http://10.26.33.242:8774/images/83872a57-51e1-4546-b26f-bc73ebc5d26e",
               "rel":"bookmark"
            }
         ]
      },
      "OS-EXT-SRV-ATTR:user_data":null,
      "OS-EXT-STS:vm_state":"building",
      "OS-EXT-SRV-ATTR:instance_name":"instance-00000001",
      "OS-EXT-SRV-ATTR:root_device_name":null,
      "OS-SRV-USG:launched_at":null,
      "flavor":{
         "id":"1",
         "links":[
            {
               "href":"http://10.26.33.242:8774/flavors/1",
               "rel":"bookmark"
            }
         ]
      },
      "id":"b85ebd3b-df19-47f7-8116-ce99f00363cb",
      "security_groups":[
         {
            "name":"default"
         }
      ],
      "OS-SRV-USG:terminated_at":null,
      "os-extended-volumes:volumes_attached":[

      ],
      "user_id":"5a83b2b6a1114dfc8523aa603a45e115",
      "OS-EXT-SRV-ATTR:hostname":"fira12",
      "OS-DCF:diskConfig":"MANUAL",
      "accessIPv4":"",
      "accessIPv6":"",
      "OS-EXT-SRV-ATTR:reservation_id":"r-3t0rugpm",
      "progress":0,
      "OS-EXT-STS:power_state":0,
      "OS-EXT-AZ:availability_zone":"",
      "metadata":{

      },
      "status":"BUILD",
      "OS-EXT-SRV-ATTR:ramdisk_id":"",
      "updated":"2016-10-18T12:13:41Z",
      "hostId":"",
      "OS-EXT-SRV-ATTR:host":null,
      "description":null,
      "tags":[

      ],
      "key_name":null,
      "OS-EXT-SRV-ATTR:kernel_id":"",
      "locked":false,
      "OS-EXT-SRV-ATTR:hypervisor_hostname":null,
      "name":"fira12",
      "OS-EXT-SRV-ATTR:launch_index":0,
      "created":"2016-10-18T12:13:40Z",
      "tenant_id":"b2ce2b442de94434b7ac40bf733db46a",
      "host_status":"",
      "config_drive":""
   }
}
```

#### POLL SERVER AFTER PROVISIONING IS COMPLETED

```http
GET http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb
OpenStack-API-Version: compute 2.35
User-Agent: python-novaclient
Accept: application/json
X-OpenStack-Nova-API-Version: 2.35
X-Auth-Token: {SHA1}4c9fd04a28c8b1df81de0689cbda4e2054439132"
```

RESPONSE:

```http
200 OK
Content-Length: 1934
Content-Type: application/json
Openstack-Api-Version: compute 2.35
X-Openstack-Nova-Api-Version: 2.35
Vary: OpenStack-API-Version, X-OpenStack-Nova-API-Version
X-Compute-Request-Id: req-7d5eb2a7-b2da-4710-aab1-ab48457daf7d
Date: Tue, 18 Oct 2016 12:14:53 GMT
Connection: keep-alive

{
   "server":{
      "OS-EXT-STS:task_state":null,
      "addresses":{
         "private":[
            {
               "OS-EXT-IPS-MAC:mac_addr":"fa:16:3e:5e:04:eb",
               "version":4,
               "addr":"10.0.0.2",
               "OS-EXT-IPS:type":"fixed"
            }
         ]
      },
      "links":[
         {
            "href":"http://10.26.33.242:8774/v2.1/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"self"
         },
         {
            "href":"http://10.26.33.242:8774/servers/b85ebd3b-df19-47f7-8116-ce99f00363cb",
            "rel":"bookmark"
         }
      ],
      "image":{
         "id":"83872a57-51e1-4546-b26f-bc73ebc5d26e",
         "links":[
            {
               "href":"http://10.26.33.242:8774/images/83872a57-51e1-4546-b26f-bc73ebc5d26e",
               "rel":"bookmark"
            }
         ]
      },
      "OS-EXT-SRV-ATTR:user_data":null,
      "OS-EXT-STS:vm_state":"active",
      "OS-EXT-SRV-ATTR:instance_name":"instance-00000001",
      "OS-EXT-SRV-ATTR:root_device_name":"/dev/sda",
      "OS-SRV-USG:launched_at":"2016-10-18T12:14:50.000000",
      "flavor":{
         "id":"1",
         "links":[
            {
               "href":"http://10.26.33.242:8774/flavors/1",
               "rel":"bookmark"
            }
         ]
      },
      "id":"b85ebd3b-df19-47f7-8116-ce99f00363cb",
      "security_groups":[
         {
            "name":"default"
         }
      ],
      "OS-SRV-USG:terminated_at":null,
      "os-extended-volumes:volumes_attached":[

      ],
      "user_id":"5a83b2b6a1114dfc8523aa603a45e115",
      "OS-EXT-SRV-ATTR:hostname":"fira12",
      "OS-DCF:diskConfig":"MANUAL",
      "accessIPv4":"",
      "accessIPv6":"",
      "OS-EXT-SRV-ATTR:reservation_id":"r-3t0rugpm",
      "progress":0,
      "OS-EXT-STS:power_state":1,
      "OS-EXT-AZ:availability_zone":"nova",
      "metadata":{

      },
      "status":"ACTIVE",
      "OS-EXT-SRV-ATTR:ramdisk_id":"",
      "updated":"2016-10-18T12:14:50Z",
      "hostId":"07493bd1d8cb9abaa5897b2f6f1143d6a1e75951a8ba4414588d5f48",
      "OS-EXT-SRV-ATTR:host":"sof2-vapi-1-214",
      "description":null,
      "tags":[

      ],
      "key_name":null,
      "OS-EXT-SRV-ATTR:kernel_id":"",
      "locked":false,
      "OS-EXT-SRV-ATTR:hypervisor_hostname":"domain-c7.EB3DD91F-0A06-4068-8B9F-64841FBF9C75",
      "name":"fira12",
      "OS-EXT-SRV-ATTR:launch_index":0,
      "created":"2016-10-18T12:13:40Z",
      "tenant_id":"b2ce2b442de94434b7ac40bf733db46a",
      "host_status":"UP",
      "config_drive":"True"
   }
}
```
