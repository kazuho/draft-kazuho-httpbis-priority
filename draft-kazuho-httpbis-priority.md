---
title: The Priority HTTP Header Field
docname: draft-kazuho-httpbis-priority-latest
category: std

ipr: trust200902
area: Transport
workgroup: HTTP
keyword: Internet-Draft

stand_alone: yes
pi: [toc, docindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
  -
    ins: K. Oku
    name: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com
  -
    ins: L. Pardue
    name: Lucas Pardue
    org: Cloudflare
    email: lucaspardue.24.7@gmail.com

normative:
  RFC2119:
  RFC7230:
  RFC7540:

informative:
  QUIC-HTTP:
    title: "Hypertext Transfer Protocol Version 3 (HTTP/3)"
    date: 2019-04-23
    seriesinfo:
      Internet-Draft: draft-ietf-quic-http-20
    author:
      -
        ins: M. Bishop
        name: Mike Bishop
        org: Akamai
        role: editor

--- abstract

This document describes the Priority HTTP header field.  This header field can
be used by endpoints to specify the absolute precedence of an HTTP response
in an HTTP-version-independent way.

--- middle

# Introduction

It is common for an HTTP resource representation to have relationships to one or
more other resources. Clients will often discover these relationships while
processing a retrieved representation, leading to further retrieval requests.
Meanwhile, the nature of the relationship determines whether the client is
blocked from continuing to process locally available resources. For example,
visual rendering of an HTML document could be blocked by the retrieval of a CSS
file that the document refers to. In contrast, inline images do not block
rendering and get drawn progressively as the chunks of the images arrive.

To provide meaningful representation of a document at the earliest moment, it is
important for an HTTP server to prioritize the HTTP responses, or the chunks of
those HTTP responses, that it sends.

HTTP/2 ({{RFC7540}}) provides such a prioritization scheme. A client sends a
series of PRIORITY frames to communicate to the server a “priority tree”; this
represents the client's preferred ordering and weighted distribution of the
bandwidth among the HTTP responses.  However, the design has shortcomings:

* Its complexity has led to varying levels of support by the HTTP/2 client and
  servers.
* It is hard to coordinate with server-driven prioritization.  For example, a
  server, with the knowledge of the document structure, might want to prioritize
  the delivery of images that are critical to user experience above others
  images, but below the CSS files.  But with the HTTP/2 prioritization scheme,
  it is impossible for the server to determine how such images should be
  prioritized against other responses that use client-driven prioritization
  tree, because every client builds the HTTP/2 prioritization tree in a
  different way.
* It does not define a method that can be used by a server to express the
  priority of a response.  Without such a method, intermediaries cannot
  coordinate client-driven and server-driven priorities.
* The design cannot be ported cleanly to HTTP/3 ({{QUIC-HTTP}}). One of the
  primary goals of HTTP/3 is to minimize head-of-line blocking. Transmitting the
  evolving representation of a "prioritization tree" from the client to the
  server requires head-of-line blocking.

Based on these observations, this document defines the Priority HTTP header
field that can be used by both the client and the server to specify the
precedence of HTTP responses in a standardized, extensible, protocol-version-
independent, end-to-end format.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

Example HTTP requests and responses use the HTTP/2-style formatting from
{{RFC7540}}.

# The Priority HTTP Header Field

The Priority HTTP header field can appear in requests and responses. A client
uses it to specify the priority of the response. A server uses this to inform
the client that the priority was overwritten. An intermediary can use the
Priority information from client requests and server responses to correct or
amend the precedence to suit it (see {{merging}}).

The value of the Priority header field is a Structured Headers
{{!I-D.ietf-httpbis-header-structure}} Dictionary.  Each dictionary member
represents a parameter of the Priority header field.  The following parameters
are defined.

Values of the parameters MUST always be present for the parameters defined in
this document.  When any of the defined parameters is omitted, or if the
Priority header field is not used, their default values SHOULD be applied.

Unknown parameters MUST be ignored.

## urgency

The `urgency` parameter takes one of the following sh-tokens as the value, that
indicates how an HTTP response affects the processing of other responses:

* `blocking` indicates that the response blocks the processing of others.
* `document` indicates that the response contains the document that is being
  processed.
* `non-blocking` indicates that the response does not block the processing of
  the document even though the response is being used or referred to by the
  document.

The default value is `document`.

A server that supports the Priority header field SHOULD transmit HTTP responses
in the order of their urgency: `blocking` first, followed by `document`,
followed by `non-blocking`.

The following example shows a request for a CSS file with the urgency set to
`blocking`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = urgency=blocking
~~~

## progressive

The `progressive` parameter takes an sh-boolean as the value, that indicates if
a response can be processed progressively, i.e. provide some meaningful output
as chunks of the response arrive.

This boolean parameter indicates if a response can be processed progressively,
i.e. provide some meaningful output as chunks of the response arrive.

The default value of the `progressive` parameter is `0`.

A server that supports the Priority header field SHOULD distribute the bandwidth
of a connection between progressive responses that share the same urgency.

The following example shows a request for a JPEG file with the urgency parameter
set to `non-blocking` and the progressive parameter set to `1`.

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /image.jpg
priority = urgency=non-blocking, progressive=?1
~~~

# Merging Client- and Server-Driven Parameters {#merging}

It is not always the case that the client has the best view of how the HTTP
responses should be prioritized.  For example, whether a JPEG image should be
served progressively by the server depends on the structure of that image file
- a property only known to the server.

Therefore, a server is permitted to send a "Priority" response header field.
When used, the parameters found in this response header field override those
specified by the client.

For example, when the client sends an HTTP request with

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /image.jpg
priority = urgency=non-blocking, progressive=?1
~~~

and the origin responds with

~~~ example
:status = 200
content-type = image/jpeg
priority = progressive=?0
~~~

the intermediary's view of the progressiveness of the response becomes negative,
because the server-provided value overrides that provided by the client.  The
urgency is deemed as `non-blocking`, because the server did not specify the
parameter.

# Coexistence with HTTP/2 Priorities

When connecting to an HTTP/2 ({{RFC7540}}) server, a client that uses this
header-based prioritization scheme SHOULD send a
`SETTINGS_HEADER_BASED_PRIORITY` settings parameter (0xTBD) with a value of
zero.  An intermediary SHOULD set the settings parameter for a connection it
establishes when and only when all the requests to be sent over that connection
originates from a client that utilizes this header-based prioritization scheme.

The existence of this settings parameter instructs the server that recognizes
the settings parameter to use the header-based prioritization scheme instead of
the frame-based prioritization scheme defined by HTTP/2.  A server that does not
recognize this settings parameter would respect the frame-based prioritization
scheme.

# Considerations

## Why use an End-to-End Header Field?

Contrary to the prioritization scheme of HTTP/2 that uses a hop-by-hop a frame,
the Priority header field is defined as end-to-end.

The rationale is that the Priority header field transmits how each response
affects the client's processing of those responses, rather than how relatively
urgent each response is to others.  The way a client processes a response is a
property associated to that client generating that request.  Not that of an
intermediary.  Therefore, it is an end-to-end property.  How these end-to-end
properties carried by the Priority header field affect the prioritization
between the responses that share a connection is a hop-by-hop issue.

Having the Priority header field defined as end-to-end is important for caching
intermediaries.  Such intermediaries can cache the value of the Priority header
field along with the response, and utilize the value of the cached header field
when serving the cached response, only because the header field is defined as
end-to-end rather than hop-by-hop.

It should also be noted that the use of a header field carrying a textual value
makes the prioritization scheme extensible; see the discussion below.

## Why are there Only Three Levels of Urgency?

One of the aims of this specification is to define a mechanism for merging
client- and server-provided hints for prioritizing the responses.  For that to
work, each urgency level needs to have a well-defined meaning.  As an example, a
server can assign the highest precedence among the non-blocking responses to an
HTTP response carrying an icon, because the meaning of "non-blocking" is
shared among the endpoints.

This specification restricts itself to defining just three levels of urgency, in
order to provide sufficient guranularity for prioritizing responses for ordinary
web browsing, at minimal complexity.

However, that does not mean that the prioritization scheme would forever be
stuck to the three levels.  The design provides extensibility.  If deemed
necessary, it would be possible to divide any of the three urgency levels into
sub-levels.

As an example, a server could assign an `importance` attribute to the priority
of each image that it provides, so that an intermediary could prioritize
certain images above others.  Or, a graphical user-agent could send a `visible`
attribute to indicate if the resource being requested is within the viewport.

# Security Considerations

TBD

# IANA Considerations

TBD

--- back
