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

informative:

--- abstract

This document describes the Priority HTTP header field.  This header field can
be used by endpoints to specify the absolute precedence of an HTTP response
in an HTTP-version-independent way.

--- middle

# Introduction

It is common for an HTTP ({{!RFC7230}}) resource representation to have
relationships to one or more other resources.  Clients will often discover these
relationships while processing a retrieved representation, leading to further
retrieval requests.  Meanwhile, the nature of the relationship determines
whether the client is blocked from continuing to process locally available
resources.  For example, visual rendering of an HTML document could be blocked
by the retrieval of a CSS file that the document refers to.  In contrast, inline
images do not block rendering and get drawn progressively as the chunks of the
images arrive.

To provide meaningful representation of a document at the earliest moment, it is
important for an HTTP server to prioritize the HTTP responses, or the chunks of
those HTTP responses, that it sends.

HTTP/2 ({{?RFC7540}}) provides such a prioritization scheme. A client sends a
series of PRIORITY frames to communicate to the server a “priority tree”; this
represents the client's preferred ordering and weighted distribution of the
bandwidth among the HTTP responses.  However, the design has shortcomings:

* Its complexity has led to varying levels of support by HTTP/2 clients and
  servers.
* It is hard to coordinate with server-driven prioritization.  For example, a
  server, with knowledge of the document structure, might want to prioritize
  the delivery of images that are critical to user experience above other
  images, but below the CSS files.  But with the HTTP/2 prioritization scheme,
  it is impossible for the server to determine how such images should be
  prioritized against other responses that use the client-driven prioritization
  tree, because every client builds the HTTP/2 prioritization tree in a
  different way.
* It does not define a method that can be used by a server to express the
  priority of a response.  Without such a method, intermediaries cannot
  coordinate client-driven and server-driven priorities.
* The design cannot be ported cleanly to HTTP/3 ({{?I-D.ietf-quic-http}}).  One
  of the primary goals of HTTP/3 is to minimize head-of-line blocking.
  Transmitting the evolving representation of a "prioritization tree" from the
  client to the server requires head-of-line blocking.

Based on these observations, this document defines the Priority HTTP header
field that can be used by both the client and the server to specify the
precedence of HTTP responses in a standardized, extensible, protocol-version-
independent, end-to-end format. This header-based prioritization scheme can act
as a substitute for the HTTP/2 frame-based prioritization scheme (see
{{coexistence}}).

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.

The terms sh-token and sh-boolean are imported from
{{!I-D.ietf-httpbis-header-structure}}.

Example HTTP requests and responses use the HTTP/2-style formatting from
{{?RFC7540}}.

# The Priority HTTP Header Field

The Priority HTTP header field can appear in requests and responses. A client
uses it to specify the priority of the response. A server uses it to inform
the client that the priority was overwritten. An intermediary can use the
Priority information from client requests and server responses to correct or
amend the precedence to suit it (see {{collaboration}}).

The value of the Priority header field is a Structured Headers
{{!I-D.ietf-httpbis-header-structure}} Dictionary.  Each dictionary member
represents a parameter of the Priority header field.  This document defines the
`urgency` and `progressive` parameters. Values of these parameters MUST always
be present.  When any of the defined parameters are omitted, or if the Priority
header field is not used, their default values SHOULD be applied.

Unknown parameters MUST be ignored.

## urgency

The `urgency` parameter takes an integer between -1 and 6 as shown below:

| Urgency         | Definition                        |
|----------------:|:----------------------------------|
|              -1 | prerequisite ({{prerequisite}})   |
|               0 | default ({{default}})             |
| between 1 and 5 | supplementary ({{supplementary}}) |
|               6 | background ({{background}})       |
{: #urgencies title="Urgencies"}

The value is encoded as an sh-integer.  The default value is zero.

A server SHOULD transmit HTTP responses in the order of their urgency values.
The lower the value, the higher the precedence.

The following example shows a request for a CSS file with the urgency set to
`-1`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = urgency=-1
~~~

The definition of the urgencies and their expected use-case are described below.
Endpoints SHOULD respect the definition of the values when assigning urgencies.

### prerequisite

The prerequisite urgency (value -1) indicates that the response prevents other
responses with an urgency of prerequisite or default from being used.

For example, use of an external stylesheet can block a web browser from
rendering the HTML. In such case, the stylesheet is given the prerequisite
urgency.

### default

The default urgency (value 0) indicates a response that is to be used as it is
delivered to the client, but one that does not block other responses from being
used.

For example, when a user using a web browser navigates to a new HTML document,
the request for that HTML is given the default urgency.  When that HTML document
uses a custom font, the request for that custom font SHOULD also be given the
default urgency.  This is because the availablity of the custom font is likely
a precondition for the user to use that portion of the HTML document, which is
to be rendered by that font.

### supplementary

The supplementary urgency indicates a response that is helpful to the client
using a composition of responses, even though the response itself is not
mandatory for using those responses.

For example, inline images (i.e., images being fetched and displayed as part of
the document) are visually important elements of an HTML document.  As such,
users will typically not be prevented from using the document, at least to some
degree, before any or all of these images are loaded.  Display of those images
are thus considered to be an improvement for visual clients rather than a
prerequisite for all user agents.  Therefore, such images will be given the
supplementary urgency.

Values between 1 and 5 are used to represent this urgency, to provide
flexibility to the endpoints for giving some responses more or less precedence
than others that belong to the supplementary group. {{collaboration}} explains how
these values might be used.

Clients SHOULD NOT use values 1 and 5.  Servers MAY use these values to
prioritize a response above or below other supplementary responses.

Clients MAY use values 2 to indicate that a request is given relatively high
priority, or 4 to indicate relatively low priority, within the supplementary
urgency group.

For example, an image certain to be visible at the top of the page, might be
assigned a value of 2 instead of 3, as it will have a high visual impact for the
user.  Conversely, an asynchronously loaded JavaScript file might be assigned an
urgency value of 4, as it is less likely to have a visual impact.

When none of the considerations above is applicable, the value of 3 SHOULD be
used.

### background

The background urgency (value 6) is used for responses of which the delivery can
be postponed without having an impact on using other responses.

As an example, the download of a large file in a web browser would be assigned
the background urgency so it would not impact further page loads on the same
connection.

## progressive

The `progressive` parameter takes an sh-boolean as the value that indicates if
a response can be processed progressively, i.e. provide some meaningful output
as chunks of the response arrive.

The default value of the `progressive` parameter is `0`.

A server SHOULD distribute the bandwidth of a connection between progressive
responses that share the same urgency.

A server SHOULD transmit non-progressive responses one by one, preferably in the
order the requests were generated.  Doing so maximizes the chance of the client
making progress in using the composition of the HTTP responses at the earliest
moment.

The following example shows a request for a JPEG file with the urgency parameter
set to `3` and the progressive parameter set to `1`.

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /image.jpg
priority = urgency=3, progressive=?1
~~~

# Merging Client- and Server-Driven Parameters {#merging}

The "Priority" header field is an end-to-end signal ({{end-to-end}}) that allows
endpoints to collaborate on priorities ({{collaboration}}).

Client-driven parameters are sent in requests and server-driven parameters are
sent in responses. When an intermediary is present, the client's suggested
priority is passed to the origin server, which gives it the opportunity to
improve the priority information. The intermediary can then use the origin
server response in order to decide how to best prioritize the response that it
sends.

For example, a client using an HTML document discovers several inline images and
requests them with a low urgency via an intermediary. The origin server has
knowledge that one of the images heavily affects the usage of the HTML document
and it is therefore beneficial to promote the urgency.

To illustrate, when a client sends a request with

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /menu.png
priority = urgency=3, progressive=?1
~~~

and the origin responds with

~~~ example
:status = 200
content-type = image/png
priority = urgency=1
~~~

the intermediary's understanding of the urgency is promoted from `3` to `1`. The
origin omitted the `progressive` parameter and therefore the value `1` specified
by the client continues to be used.

# Coexistence with HTTP/2 Priorities {#coexistence}

Standard HTTP/2 ({{!RFC7540}}) endpoints use frame-based prioritization, whereby
a client sends priority information in dedicated fields present in HEADERS and
PRIORITY frames. A client might instead choose to use header-based
prioritization as specified in this document.

## The SETTINGS_HEADER_BASED_PRIORITY SETTINGS Parameter

To improve communication of the client's intended prioritization scheme, this
document specifies a new HTTP/2 SETTINGS parameter with the name
`SETTINGS_HEADER_BASED_PRIORITY`. The value of the parameter MUST be 0 or 1; the
initial value is 0. Frame-based prioritization is respected when the value is 0,
or when the server does not recognize the setting.

An HTTP/2 client that uses header-based priority SHOULD send a
`SETTINGS_HEADER_BASED_PRIORITY` parameter with a value of 1 when connecting to
a server.

An intermediary SHOULD send a `SETTINGS_HEADER_BASED_PRIORITY` parameter with a
value of 1 for a connection it establishes when, and only when, all the requests
to be sent over that connection originate from a client that utilizes this
header-based prioritization scheme. Otherwise this settings parameter SHOULD be
set to 0.

A client or intermediary MUST NOT send a `SETTINGS_HEADER_BASED_PRIORITY`
parameter with the value of 0 after previously sending a value of 1.

A server MUST NOT send a `SETTINGS_HEADER_BASED_PRIORITY` parameter. Upon
receipt, a client that supports header-based prioritization MUST close the
connection with a protocol error. Non-supporting clients will ignore this
extension element (see {{!RFC7540}}, Section 5.5).

# Considerations

## The Case for a Collaborative Prioritization Process {#collaboration}

HTTP/2 prioritization places a lot of responsibility on the client. It provides
a very flexible dependency-priority scheme that has the potential to model rich
and diverse use cases. However, experience shows that HTTP/2 implementations
have not used this for much innovation, instead preferring simpler structures.

The client is well placed to decide request priority based on information such
as the resource type and the context it will be used in. The goal and how
information is acted on is described lightly in {{?RFC7540}} Section 5.3:

>  Most importantly, priority can be used to select streams for transmitting
   frames when there is limited capacity for sending.

> Explicitly setting the priority for a stream is input to a
  prioritization process.  It does not guarantee any particular
  processing or transmission order for the stream relative to any other
  stream.  An endpoint cannot force a peer to process concurrent
  streams in a particular order using priority.  Expressing priority is
  therefore only a suggestion.

The client is not alone in understanding the best way to prioritize a request. A
server's prioritization decision is driven by a number of factors such as the
selected representation ({{?RFC7231}}, Section 3), data locality, resource
contention, load balancing across multiple clients, application-specific
annotations etc. When it generates responses it might determine that the
client's predicted prioritization was incorrect for how the selected
representation will be used. Rigidly following the client's suggestion could
lead to a sub-optimal user experience, and so to avoid this, the server can send
the response at a different effective priority than the client suggestion.

The prioritization process is a collaborative activity but there are no explicit
means by which a server can present its information. Attempting to infer reasons
about prioritization from implicit signals is impractical and error prone due to
the number of factors involved.

This document makes a case for a revised prioritization process that has is more
collaborative by allowing server-driven inputs to be more fully considered.
Collaboration is foreseen to be especially beneficial when multiple endpoints
are involved i.e. a request through an intermediary. It is important to
highlight that the goal of the process remains the same, responses are
prioritized in order make best use of a limited sending capacity.

Resources have an initial priority that begins with a neutral value. Requests
and responses permit collaborative changes to the resource's initial priority
information. A client attaches suggested priority information to the request,
omission activates a default priority. A server can (if supported) attach
priority information to the response, omission might be an acceptance of the
client offer or an implicit ignore. Once a request or response is in flight, an
endpoint cannot modify the initial priority, any further change is classed as a
reprioritization that requires a non-request or non-response carriage mechanism
(e.g. a frame).

Today's HTTP/2 priority information is exchanged as frames and restricted to
clients. In order to support the new process, changes to frames and their
handling are required. Extending behavior at that this level has proven hard
achieve in Internet deployments with challenges including availability of APIs,
configuration specification and storage, coordination across endpoints, ease of
migration, portability across HTTP versions, and extensibility of the extension
itself. Extending HTTP behaviours with header fields has generally been more
succesful because this set of challenges have already been solved or are moot.
Therefore the Priority header field is proposed by this document as the most
appropriate way to exchange initial priority information.

## Why use an End-to-End Header Field? {#end-to-end}

Contrary to the prioritization scheme of HTTP/2 that uses a hop-by-hop frame,
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

## Why do Urgencies Have Meanings?

One of the aims of this specification is to define a mechanism for merging
client- and server-provided hints for prioritizing the responses.  For that to
work, each urgency level needs to have a well-defined meaning.  As an example, a
server can assign the highest precedence among the supplementary responses to an
HTTP response carrying an icon, because the meaning of `urgency=1` is shared
among the endpoints.

This specification restricts itself to defining a minimum set of urgency levels
in order to provide sufficient granularity for prioritizing responses for
ordinary web browsing, at minimal complexity.

However, that does not mean that the prioritization scheme would forever be
stuck to the eight levels.  The design provides extensibility.  If deemed
necessary, it would be possible to subdivide any of the eight urgency levels
that are currently defined.  Or, a graphical user-agent could send a `visible`
parameter to indicate if the resource being requested is within the viewport.

A server can combine the hints provided in the Priority header field with other
information in order to improve the prioritization of responses.  For example, a
server that receives requests for a font {{?RFC8081}} and images with the same
urgency might give higher precedence to the font, so that a visual client can
render textual information at an early moment.

# Security Considerations

TBD

# IANA Considerations

This specification registers the following entry in the Permanent Message Header
Field Names registry established by {{?RFC3864}}:

Header field name:
: Priority

Applicable protocol:
: http

Status:
: standard

Author/change controller:
: IETF

Specification document(s):
: This document

Related information:
: n/a

This specification registers the following entry in the HTTP/2 Settings registry
established by {{?RFC7540}}:

Name:
: SETTINGS_HEADER_BASED_PRIORITY:

Code:
: 0xTBD

Initial value:
: 0

Specification:
: This document

--- back

# Acknowledgements

Many thanks to Robin Marx, Patrick Meenan and Ian Swett for their feedback.

# Change Log

## Since draft-kazuho-httpbis-priority-00

* None yet.
