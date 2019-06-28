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
be used by the endpoints to specify the absolute precedence of a HTTP response
in an HTTP version-independent way.

--- middle

# Introduction

It is often the case for the processing of an HTTP response to depend on the
retrieval of another HTTP response.

As an example, visual rendering of an HTML document could be blocked by the
retrieval of a CSS file that the document refers to. Inline images is another
example, though the difference is that they typically do not block rendering.
Instead, they are rendered progressively as the chunks of the images arrive.

To provide meaningful representation of a document at the earliest moment, it is
important for a HTTP server to prioritize the HTTP responses, or the chunks of
those HTTP responses, that it sends.

HTTP/2 ({{RFC7540}}) provides such prioritization scheme. A client sends a
series of PRIORITY frames to communicate to the server a “priority tree;” a tree
that represents the client's preferred ordering and weighted distribution of the
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
independent, end-to-end representation.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

# The Priority HTTP Header Field

The Priority HTTP header field is used by a client as part of an HTTP request
to specify the priority of the response.  A server can also use the header field
as part of an HTTP request to communicate the correct or amended precedence to
an intermediary.

The systax of this header field is defined as follows.

~~~ abnf
Priority           = 1#priority-directive
priority-directive = token [ "=" ( token / quoted-string ) ]
~~~

## Priority Directives

### urgency

Argument syntax:

~~~ abnf
urgency = "blocking" / "document" / "non-blocking"
~~~

The `urgency` directive indicates how a HTTP response affects the processing of
other responses.

* `blocking` indicates that the response blocks the processing of others.
* `document` indicates that the response contains the document that is being
  processed.
* `non-blocking` indicates that the response does not block the processing of
  the document even though the response is being used or referred by the
  document.

When the Priority header field or the `urgency` directive do not appear in the
request, the server SHOULD act as if urgency level of `document` was specified.

A server SHOULD transmit HTTP responses that have the `blocking` attribute, then
those with the `document` attribute, and finaly the ones with the `non-blocking`
attribute.

The following example shows a request for a CSS file with the urgency set to
`blocking`:

~~~ example
GET /style.css HTTP/1.1
Priority: urgency=blocking

~~~

### progressive

Argument syntax:

~~~ example
progressive = "yes" / "no"
~~~

This boolean directive indicates if a response can be processed progressively,
i.e. provide some meaningful output as chunks of the response arrive.

When the Priority header field or the `progressive` directive do not appear in
the request, the server SHOULD act as if a `progress` directive with an argument
of `no` has specified.

A server SHOULD distribute the bandwidth of a connection between the responses
deemed progressive sharing the same urgency.

The following examples shows a request for a JPEG file with the urgency set to
`non-blocking`, progressive set to `yes`.

~~~ example
GET /image.jpg HTTP/1.1
Priority: urgency=non-blocking, progressive=yes

~~~

## Merging Client- and Server-Driven Directives

It is not always the case that the client has the best view of how the HTTP
responses should be prioritized.  For example, whether a JPEG image should be
served progressively by the server depends on the structure of that image file
- a property only known to the server.

Therefore, a server is permitted to send a "Priority" response header field.
When used, the directives found in this response header field overrides those
specified by the client.

For example, when the client sends a HTTP request with

~~~ example
Priority: urgency=non-blocking; progressive=yes
~~~

and the origin responds with

~~~ example
Priority: progressive=no
~~~

the intermediary's view of the progressiveness of the response becomes negative,
because the server-provided value overrides that provided by the client.  The
urgency is deemed as `non-blocking`, because the server did not specify the
directive.

# Coexistence with HTTP/2 Priorities

When connecting to a HTTP/2 ({{RFC7540}}) server, a client that uses this
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

# Security Considerations

TBD

# IANA Considerations

TBD

--- back
