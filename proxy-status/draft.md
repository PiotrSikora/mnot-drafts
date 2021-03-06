---
title: The Proxy-Status HTTP Header Field
abbrev: Proxy-Status
docname: draft-nottingham-proxy-status-00
date: 2019
category: info

ipr: trust200902
area: General
workgroup:
keyword: Internet-Draft

stand_alone: yes
pi: [toc, tocindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
 -
    ins: M. Nottingham
    name: Mark Nottingham
    organization: Fastly
    email: mnot@mnot.net
    uri: https://www.mnot.net/
 -
    ins: P. Sikora
    name: Piotr Sikora
    organization: Google
    email: piotrsikora@google.com

normative:
  RFC2119:

informative:


--- abstract

This document defines the Proxy-Status HTTP header field to convey the details of errors generated by HTTP intermediaries.


--- note_Note_to_Readers

*RFC EDITOR: please remove this section before publication*

The issues list for this draft can be found at <https://github.com/mnot/I-D/labels/proxy-status>.

The most recent (often, unpublished) draft is at <https://mnot.github.io/I-D/proxy-status/>.

See also the draft's current status in the IETF datatracker, at
<https://datatracker.ietf.org/doc/draft-nottingham-proxy-status/>.

Precursors that informed this work include (but are not limited to):

* https://docs.fastly.com/guides/debugging/common-503-errors
* https://support.cloudflare.com/hc/en-us/sections/200820298-Error-Pages
* https://cloud.google.com/load-balancing/docs/https/https-logging-monitoring
* https://docs.google.com/document/d/1fMEK80KlpHcL4CwhniOupgQu4MDsxK4y4dKhthjjCMA


--- middle

# Introduction

HTTP intermediaries -- including both forward proxies and gateways (also known as "reverse
proxies") -- have become an increasingly significant part of HTTP deployments. In particular,
reverse proxies and Content Delivery Networks (CDNs) form part of the critical infrastructure of
many Web sites.

Typically, HTTP intermediaries forward requests towards the origin server and then forward their responses back to clients. However, if an error occurs, the response is generated by the intermediary itself.

HTTP accommodates these types of errors with a few status codes; for example, 502 Bad Gateway and 504 Gateway Timeout. However, experience has shown that more information is necessary to aid debugging and communicate what's happened to the client.

To address this, {{header}} defines a new HTTP response header field to convey such information, using the Proxy Status Types defined in {{types}}. {{register}} explains how to define new Proxy Status Types.


## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all capitals, as
shown here.

This specification uses Structured Headers {{!I-D.ietf-httpbis-header-structure}} to specify syntax. The terms sh-param-list, sh-item, sh-string, sh-token and sh-integer refer to the structured types defined therein.

Note that in this specification, "proxy" is used to indicate both forward and "reverse" proxies, otherwise known as gateways. "Forward" indicates the connection in the direction leading to the origin server for the request.


# The Proxy-Status HTTP Header Field {#header}

The Proxy-Status HTTP response header field allows an intermediary to indicate the nature and details of an error condition it encounters when servicing a request.

It is a Structured Headers {{!I-D.ietf-httpbis-header-structure}} Parameterised List, where each item in the list indicates an error condition. Typically, it will have only one param-item (the error condition that triggered generation of the response it occurs within), but more than one value is not prohibited.

Each param-item's primary-id is a Proxy Status Type, a registered value that indicates the nature of the error.

Each param-item can have zero to many parameters. {{params}} lists parameters that can be used with all Proxy Status Types; individual types can define additional parameters to use with them. All parameters are optional; see {{security}} for their potential security impact.

For example:

~~~ example
HTTP/1.1 504 Gateway Timeout
Proxy-Status: connect_timeout; proxy_id=FooCDN;
  origin_id=abc; origin_tries=3
~~~

indicates the specific nature of the timeout as a connect timeout to the origin with the identifier "abc", and that is was generated by the intermediary that identifies itself as "FooCDN." Furthermore, three connection attempts were made.

Or:

~~~ example
HTTP/1.1 429 Too Many Requests
Proxy-Status: request_error; proxy_id=BarReverseProxy
~~~

indicates that this 429 Too Many Requests response was generated by the intermediary, not the origin.

Each Proxy Status Type has a Recommended HTTP Status Code. When generating a HTTP response containing Proxy-Status, its http status code SHOULD be set to the Recommended HTTP Status Code. However, there may be circumstances (e.g., for backwards compatibility with previous behaviours) when another status code might be used.

{{types}} lists the Proxy Status Types defined in this document; new ones can be defined using the procedure outlined in {{register}}.

Note that there are various security considerations for intermediaries using the Proxy-Status header field; see {{security}}.

Origin servers MUST NOT generate the Proxy-Status header field.


## Generic Proxy Status Parameters {#params}

This section lists parameters that are potentially applicable to most Proxy Status Types.

* proxy_id - a sh-token identifying the HTTP intermediary generating this response.
* origin_id - a sh-token identifying the origin server whose behaviour triggered this response.
* forward_protocol - a sh-token indicating the ALPN protocol identifier {{!RFC7301}} used to connect to the next hop. This is only applicable when that connection was actually established.
* paths_tried - a sh-integer indicating the number of forward paths that have been tried before generating this error.
* forward_tries - a sh-integer indicating the number of times that the error specified has occurred on the XXX.
* note - a sh-string containing additional information not captured anywhere else. This can include implementation-specific or deployment-specific information.


# Proxy Status Types {#types}

This section lists the Proxy Status Types defined by this document. See {{register}} for information about defining new Proxy Status Types.

## Backend Not Found

* Name: backend_not_found
* Description: The intermediary cannot determine the appropriate origin to use for this request; for example, it may not be configured. Note that this error is specific to gateways, which typically require specific configuration to identify the "back end" server; forward proxies use in-band information to identify the origin server.
* Extra Parameters: None.
* Recommended HTTP status code: 500

## Forward DNS Timeout

* Name: forward_dns_timeout
* Description: The intermediary encountered a timeout when trying to find an IP address for the forward hostname.
* Extra Parameters: None.
* Recommended HTTP status code: 504

## Forward DNS Error

* Name: forward__dns_error
* Description: The intermediary encountered a DNS error when trying to find an IP address for forward connection.
* Extra Parameters:
  - rcode: A sh-string conveying the DNS RCODE that indicates the error type. See {{!RFC8499}}, Section 3.
* Recommended HTTP status code: 502

## Forward IP Prohibited

* Name: forward_ip_prohibited
* Description: The intermediary is configured to prohibit connections to the forward IP address.
* Extra Parameters: None.
* Recommended HTTP status code: 502

## Forward IP Unroutable

* Name: forward_ip_unroutable
* Description: The intermediary cannot find a route to the forward IP address.
* Extra Parameters: None.
* Recommended HTTP status code: 502

## Forward Connection Refused

* Name: forward_connection_refused
* Description: The intermediary's forward connection was refused.
* Extra Parameters: None.
* Recommended HTTP status code: 502

## Forward Connection Closed

* Name: forward_connection_closed
* Description: The intermediary's forward connection was closed before any part of the response was received. If some part was received, see forward_http_incomplete.
* Extra Parameters: None.
* Recommended HTTP status code: 502

## Forward Connect Timeout

* Name: forward_connect_timeout
* Description: The intermediary's attempt to open a forward connection timed out.
* Extra Parameters: None.
* Recommended HTTP status code: 504

## Forward Read Timeout

* Name: forward_read_timeout
* Description: The intermediary expect
* Extra Parameters: None.
* Recommended HTTP status code: 504

## Forward Write Timeout

* Name: origin_write_timeout
* Description:
* Extra Parameters: None.
* Recommended HTTP status code: 504

## Forward Node Down

* Name: forward_node_down
* Description: The intermediary considers the next hop to be unavailable; e.g., recent attempts to communicate with it may have failed, or a health check may indicate that it is down.
* Extra Parameters:

* Recommended HTTP status code: 500

## Forward Connection Limit Reached

* Name: Forward_connnection_limit
* Description: The intermediary is configured to limit the number of connections it has to the next hop, and that limit has been passed.
* Extra Parameters: None.
* Recommended HTTP status code:

## HTTP Response Status

* Name: http_response_status
* Description: The intermediary has received a 4xx or 5xx status code from the next hop, and
* Extra Parameters: None.
* Recommended HTTP status code:

## HTTP Incomplete Response

* Name: http_response_incomplete
* Description: The intermediary received an incomplete response to the request from the next hop forward.
* Extra Parameters: None.
* Recommended HTTP status code: 502

## Forward HTTP Protocol Error

* Name: foreward_http_protocol
* Description: The intermediary encountered a HTTP protocol error when communicating with the next hop forward. This error should only be used when a more specific one is not defined.
* Extra Parameters:
  - details: a sh-string containing details about the error condition. For example, this might be the HTTP/2 error code or free-form text describing the condition.
* Recommended HTTP status code: 502

## HTTP Response Header Block Too Large

* Name: http_response_header_block_size
* Description: The intermediary received a response to the request whose header block was considered too large.
* Extra Parameters:
  - header_block_size: a sh-integer indicating how large the headers received were. Note that they might not be complete; i.e., the intermediary may have discarded or refused additional data.
* Recommended HTTP status code: 502

## HTTP Response Header Too Large

* Name: http_response_header_size
* Description: The intermediary received a response to the request containing an individual header line that was considered too large.
* Extra Parameters:
  - header_name: a sh-string indicating the name of the header that triggered the error.
* Recommended HTTP status code: 502

## HTTP Response Body Too Large

* Name: http_response_body_size
* Description: The intermediary received a response to the request whose body was considered too large.
* Extra Parameters:
  - body_size: a sh-integer indicating how large the body received was. Note that it may not have been complete; i.e., the intermediary may have discarded or refused additional data.
* Recommended HTTP status code: 502

## HTTP Response Transfer-Coding Error

* Name: http_response_transfer_coding
* Description: The intermediary encountered an error decoding the transfer-coding of the response.
* Extra Parameters:
  - coding: a sh-token containing the specific coding that caused the error.
  - details: a sh-string containing details about the error condition.
* Recommended HTTP status code: 502

## HTTP Response Content-Coding Error

* Name: http_response_content_coding
* Description: The intermediary encountered an error decoding the content-coding of the response.
* Extra Parameters:
  - coding: a sh-token containing the specific coding that caused the error.
  - details: a sh-string containing details about the error condition.
* Recommended HTTP status code: 502

## Forward TLS Error

* Name: forward_tls_error
* Description: The intermediary encountered a TLS error when communicating with the next hop forward.
* Extra Parameters:
  - alert_message: a sh-token containing the applicable description string from the TLS Alerts registry.
* Recommended HTTP status code: 502

## Proxy Internal Error

* Name: proxy_internal_error
* Description: The intermediary encountered an internal error unrelated to the origin.
* Extra Parameters:
  - details: a sh-string containing details about the error condition.
* Recommended HTTP status code: 500

## Request Error

* Name: request_error
* Description: The intermediary is generating a client (4xx) response on the origin's behalf. Applicable status codes include (but are not limited to) 400, 403, 405, 406, 408, 411, 413, 414, 415, 416, 417, 429. This proxy status type helps distinguish between responses generated by intermediaries from those generated by the origin.
* Extra Parameters: None.
* Recommended HTTP status code: The applicable 4xx status code


# Defining New Proxy Status Types {#register}

New Proxy Status Types can be defined by registering them in the HTTP Proxy Status Types registry.

Registration requests are reviewed and approved by a Designated Expert, as per {{!RFC8126}} Section 4.5. A specification document is appreciated, but not required.

The Expert(s) should consider the following factors when evaluating requests:

* Community feedback
* If the value is sufficiently well-defined
* If the value is generic; vendor-specific, application-specific and deployment-specific values are discouraged

Registration requests should use the following template:

* Name: [a name for the Proxy Status Type that is allowable as a sh-param-list key]
* Description: [a description of the conditions that generate the Proxy Status Types]
* Extra Parameters: [zero or more optional parameters, typed using one of the types available in sh-item]
* Recommended HTTP status code: [the appropriate HTTP status code for this entry]

See the registry at <https://iana.org/assignments/http-proxy-statuses> for details on where to send registration requests.


# IANA Considerations

Upon publication, please create the HTTP Proxy Status Types registry at <https://iana.org/assignments/http-proxy-statuses> and populate it with the types defined in {{types}}; see {{register}} for its associated procedures.


# Security Considerations {#security}

One of the primary security concerns when using Proxy-Status is leaking information that might aid an attacker.

As a result, care needs to be taken when deciding to generate a Proxy-Status header. Note that intermediaries are not required to generate a Proxy-Status header field in any response, and can conditionally generate them based upon request attributes (e.g., authentication tokens, IP address).

Likewise, generation of all parameters is optional.

Special care needs to be taken in generating proxy_id and origin_id parameters, as they can expose information about the intermediary's configuration and back-end topology.

--- back
