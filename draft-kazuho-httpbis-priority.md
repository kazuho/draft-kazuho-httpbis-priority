---
title: Extensible Prioritization Scheme for HTTP
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

This document describes a scheme for prioritizing HTTP responses. This scheme
expresses the priority of each HTTP response using absolute values, rather than
as a relative relationship between a group of HTTP responses.

This document defines a new priority scheme, a Priority header field for
communicating the initial priority in an HTTP version-independent manner,
as well as HTTP/2 and HTTP/3 frames for reprioritizing the responses. These
share a common format structure that is designed to provide future extensibility.

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
bandwidth among the HTTP responses. However, the design and implementation of
this scheme has been observed to have shortcomings, explained in {{motivation}}.

This document defines the Priority HTTP header field that can be used by both
client and server to specify the precedence of HTTP responses in a standardized,
extensible, protocol-version-independent, end-to-end format. Along with the
protocol-version-specific frame for reprioritization, this prioritization scheme
acts as a substitute for the original prioritization scheme of HTTP/2.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.

The terms sh-token and sh-boolean are imported from
{{!STRUCTURED-HEADERS=I-D.ietf-httpbis-header-structure}}.

Example HTTP requests and responses use the HTTP/2-style formatting from
{{?RFC7540}}.

This document uses the variable-length integer encoding from
{{!I-D.ietf-quic-transport}}.


# Motivation for Replacing HTTP/2 Priorities {#motivation}

An important feature of any implementation of a protocol that provides
multiplexing is the ability to prioritize the sending of information. This was
an important realization in the design of HTTP/2. Prioritization is a
difficult problem, so it will always be suboptimal, particularly if one endpoint
operates in ignorance of the needs of its peer.

HTTP/2 introduced a complex prioritization signaling scheme that used a
combination of dependencies and weights, formed into an unbalanced tree. This
scheme has suffered from poor deployment and interoperability.

The rich flexibility of client-driven HTTP/2 prioritization tree building is
rarely exercised; experience shows that clients either choose a single model
optimized for a web use case (and don't vary it) or do nothing at all. But every
client builds their prioritization tree in a different way, which makes it
difficult for servers to understand their intent and act or intervene
accordingly.

Many HTTP/2 server implementations do not include support for the priority
scheme, some favoring instead bespoke server-driven schemes based on heuristics
and other hints, like the content type of resources and the order in which
requests arrive. For example, a server, with knowledge of the document
structure, might want to prioritize the delivery of images that are critical to
user experience above other images, but below the CSS files. Since client trees
vary, it is impossible for the server to determine how such images should be
prioritized against other responses.

The HTTP/2 scheme allows intermediaries to coalesce multiple client trees into a
single tree that is used for a single upstream HTTP/2 connection. However, most
intermediaries do not support this. The scheme does not define a method that can
be used by a server to express the priority of a response. Without such a
method, intermediaries cannot coordinate client-driven and server-driven
priorities.

HTTP/2 describes denial-of-service considerations for implementations. On
2019-08-13 Netflix issued an advisory notice about the discovery of several
resource exhaustion vectors affecting multiple HTTP/2 implementations. One
attack, CVE-2019-9513 aka "Resource Loop", is based on manipulation of the
priority tree.

The HTTP/2 scheme depends on in-order delivery of signals, leading to challenges
in porting the scheme to protocols that do not provide global ordering. For
example, the scheme cannot be used in HTTP/3 {{?I-D.ietf-quic-http}} without
changing the signal and its processing.

Considering the problems with deployment and adaptability to HTTP/3, retaining
the HTTP/2 priority scheme increases the complexity of the entire system without
any evidence that the value it provides offsets that complexity. In fact,
multiple experiments from independent research have shown that simpler schemes
can reach at least equivalent performance characteristics compared to the more
complex HTTP/2 setups seen in practice, at least for the web use case.

The problems and insights laid out above are motivation for the alternative and
more straightforward prioritization scheme presented in this document. In order
to support deployment of new schemes, a general-purpose negotiation mechanism is
specified in {{negotiating-priorities}}.

# Negotiating Priorities

The document specifies a negotiation mechanism that allows each peer to
communicate which, if any, priority schemes are supported, as well as the
server's ranked preference.

For both HTTP/2 and HTTP/3, either peer's SETTINGS may arrive first,
so any negotiation must be unilateral and not rely upon receiving
the peer's SETTINGS value.

Servers are likely to only use one prioritization scheme at once per each
connection, and may be unable to change the scheme once established, so the
setting MUST be sent prior to the first request if it is ever sent. In HTTP/3,
SETTINGS might arrive after the first request even if they are sent first.
Therefore, future specifications that define alternative prioritization schemes
for HTTP/3 SHOULD define how the server would act when it receives a
stream-level priority signal prior to receiving the SETTINGS frame.

## The SETTINGS_PRIORITIES SETTINGS Parameter

This document adds a new SETTINGS parameter to those defined by
[RFC7540], Section 6.5.2.

The new parameter name is SETTINGS_PRIORITIES, which allows both
peers to indicate which prioritization schemes they support.

A value of 0 indicates no support for priorities. If either side sends the
parameter with a value of 0, clients SHOULD NOT send hop-by-hop prioritiy
signals (e.g., HTTP/2 PRIORITY frame) and servers SHOULD NOT make any
assumptions based on the presence or lack thereof of such signals.

If the value is non-zero, then the least significant 8 bits indicate the
sender's most preferred priority scheme, the second least significant 8 bits
indicate the sender's second choice, and so on. This allows expressing
support for 4 schemes in HTTP/2 and 7 in HTTP/3.

A sender MUST comply with the following restrictions when constructing a
preference list: duplicate 8-bit values (excluding the value 0) MUST NOT be used,
and if any byte is 0 then all more significant bytes MUST also be 0. An endpoint
that receives a setting in violation of these requirements MUST treat it as a
connection error of type PROTOCOL_ERROR for HTTP/2 {{!RFC7540}}, or of type
HTTP_SETTINGS_ERROR for HTTP/3 {{!I-D.ietf-quic-http}}.

In HTTP/2, the setting SHOULD appear in the first SETTINGS frame and peers
MUST NOT process the setting if it's received multiple times in order to
avoid changing the agreed upon prioritization scheme.

If there is a prioritization scheme supported by both the client and server,
then the servers's preference order prevails and both peers SHOULD
only use the agreed upon priority scheme for the remainder of the session.
The server chooses because it is in the best position to know what
information from the client is of the most value.

Once the negotiation is complete, endpoints MAY stop sending hop-by-hop	
prioritization signals that were not negotiated in order to conserve bandwidth.	
However, endpoints SHOULD continue sending end-to-end signals (e.g., the	
Priority header field), as that might have meaningful effect to other nodes that	
handle the HTTP message.

An 8 bit value of 1 in HTTP/2 indicates support for HTTP/2 priorities
as defined in Section 5.3 of [RFC7540] and is an error in HTTP/3 because
there is not a clean mapping to HTTP/3.

## Negotiating the Extensible Priority Scheme {#settings-this-scheme}

The extensible priority scheme is negotiated using the described mechanism. It is
identified by the 8-bit value of 2.

An intermediary connecting to a backend server SHOULD declare support for the
extensible priority scheme when and only when all the requests that are to be
sent on that backend connection originates from one client-side connection that
has negotiated the use of the extensible priority scheme (see {{fairness}}).

# Urgency Prioritization

Urgency based prioritization uses two fields, `urgency` and `progressive` to
indicate the relative urgency of requests and whether responses should be
interleaved with other responses at the same priority.

The fields are encoded as a Structured Headers Dictionary
({{!STRUCTURED-HEADERS}}). Each dictionary member represents a parameter of the
Priority header field. This document defines the `urgency` and `progressive`
parameters. Values of these parameters MUST always be present. When any of the
defined parameters are omitted, or if the Priority header field is not used,
their default values SHOULD be applied.

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
default urgency.  This is because the availability of the custom font is likely
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
than others that belong to the supplementary group. {{merging}} explains how
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


# The Priority HTTP Header Field

The Priority HTTP header field can appear in requests and responses. A client
uses it to specify the priority of the response. A server uses it to inform
the client that the priority was overwritten. An intermediary can use the
Priority information from client requests and server responses to correct or
amend the precedence to suit it (see {{merging}}).

Unknown parameters MUST be ignored.


## urgency

The following example shows a request for a CSS file with the urgency set to
`-1`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = urgency=-1
~~~

## progressive

The following example shows a request for a JPEG file with the urgency parameter
set to `3` and the progressive parameter set to `1`.

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /image.jpg
priority = urgency=3, progressive=?1
~~~

# Reprioritization

Once a client sends a request, circumstances might change and mean that it is
beneficial to change the priority of the response. As an example, a web browser
might issue a prefetch request for a JavaScript file with the urgency parameter
of the Priority request header field set to `urgency=6` (background). Then, when
the user navigates to a page which references the new JavaScript file, while the
prefetch is in progress, the browser would send a reprioritization frame with the
priority field value set to `urgency=-1` (prerequisite).

However, a client cannot reprioritize a response by using the Priority header
field.  This is because an HTTP header field can only be sent as part of an HTTP
message. Therefore, to support reprioritization, it is necessary to define a
HTTP-version-dependent mechanism for transmitting the priority parameters.

This document specifies a new PRIORITY_UPDATE frame type for HTTP/2
({{!RFC7540}}) and HTTP/3 ({{!I-D.ietf-quic-http}}) that is specialized for
reprioritization. It carries updated priority parameters and references the
target of the reprioritization based on a version-specific identifier; in
HTTP/2 this is the Stream ID, in HTTP/3 this is either the Stream ID or Push ID.

In HTTP/2 and HTTP/3 a request message sent on a stream transitions it into a
state that prevents the client from sending additional frames on the stream.
Modifying this behavior requires a semantic change to the protocol, this is
avoided by restricting the stream on which a PRIORITY_UPDATE frame can be sent.
In HTTP/2 the frame is on stream zero and in HTTP/3 it is sent on the control
stream ({{!I-D.ietf-quic-http}}, Section 6.2.1).


## HTTP/2 PRIORITY_UPDATE Frame

The HTTP/2 PRIORITY_UPDATE frame (type=0xF) carries the stream ID of the
response that is being reprioritized, and the updated priority in ASCII text,
using the same representation as that of the Priority header field value.

The Stream Identifier field ({{!RFC7540}}, Section 4.1) in the PRIORITY_UPDATE
frame header MUST be zero (0x0).

~~~ drawing
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +---------------------------------------------------------------+
 |R|                        Stream ID (31)                       |
 +---------------------------------------------------------------+
 |                   Priority Field Value (*)                  ...
 +---------------------------------------------------------------+
~~~
{: #fig-h2-reprioritization-frame title="HTTP/2 PRIORITY_UPDATE Frame Payload"}

TODO: add more description of how to handle things like receiving
PRIORITY_UPDATE on wrong stream, a PRIORITY_UPDATE with an invalid ID, etc.

## HTTP/3 PRIORITY_UPDATE Frame

The HTTP/3 PRIORITY_UPDATE frame (type=0xF) carries the identifer of the element
that is being reprioritized, and the updated priority in ASCII text, using the
same representation as that of the Priority header field value.

The PRIORITY_UPDATE frame MUST be sent on the control stream
({{!I-D.ietf-quic-http}}, Section 6.2.1).

~~~ drawing
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |T|    Empty    |   Prioritized Element ID (i)                ...
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                   Priority Field Value (*)                  ...
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-h3-reprioritization-frame title="HTTP/3 PRIORITY_UPDATE Frame Payload"}

The PRIORITY_UPDATE frame payload has the following fields:

T (Prioritized Element Type):
: A one-bit field indicating the type of element
being prioritized. A value of 0 indicates a reprioritization for a Request
Stream, so the Prioritized Element ID is interpreted as a Stream ID. A
value of 1 indicates a reprioritization for a Push stream, so the Prioritized
Element ID is interpreted as a Push ID.

Empty:
: A seven-bit field that has no semantic value.

TODO: add more description of how to handle things like receiving
PRIORITY_UPDATE on wrong stream, a PRIORITY_UPDATE with an invalid ID, etc.

# Merging Client- and Server-Driven Parameters {#merging}

It is not always the case that the client has the best understanding of how the
HTTP responses deserve to be prioritized.  For example, use of an HTML document
might depend heavily on one of the inline images.  Existence of such
dependencies is typically best known to the server.

By using the "Priority" response header, a server can override the
prioritization hints provided by the client.  When used, the parameters found
in the response header field overrides those specified by the client.

For example, when the client sends an HTTP request with

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

the intermediary's understanding of the urgency is promoted from `3` to `1`,
because the server-provided value overrides the value provided by the client.
The progressiveness continues to be `1`, the value specified by the client, as
the server did not specify the `progressive` parameter.


# Security Considerations

## Fairness and Coalescing Intermediaries {#fairness}

When an intermediary coalesces HTTP requests coming from multiple clients into
one HTTP/2 or HTTP/3 connection going to the backend server, requests that
originate from one client might have higher precedence than those coming from
others.

It is sometimes beneficial for the server running behind an intermediary to obey
to the value of the Priority header field. As an example, a resource-constrained
server might defer the transmission of software update files that would have the
background urgency being associated. However, in the worst case, the asymmetry
between the precedences declared by multiple clients might cause responses going
to one end client to be delayed totally after those going to another.

In order to mitigate this fairness problem, when a server responds to a request
that is known to have come through an intermediary, the server SHOULD prioritize
the response as if it was assigned the priority of  `urgency=0, progressive=?1`
(i.e. round-robin) regardless of the value of the Priority header field being
transmitted, unless the server has the knowledge that no intermediaries are
coalescing requests from multiple clients. That can be determined by the
settings when the intermediaries support this specification (see
{{settings-this-scheme}}), or else through configuration.

A server can determine if a request came from an intermediary through
configuration, or by consulting if that request contains one of the following
header fields:

* CDN-Loop ({{?RFC8586}})
* Forwarded, X-Forwarded-For ({{?RFC7239}})
* Via ({{?RFC7230}}, Section 5.7.1)

Responding to requests coming through an intermediary in a round-robin manner
works well when the network bottleneck exists between the intermediary and the
end client, as the intermediary would be buffering the responses and then be
forwarding the chunks of those buffered responses based on the priorization
scheme it implements. A sophisticated server MAY use a weighted round-robin
reflecting the urgencies expressed in the requests, so that less urgent
responses would receive less bandwidth in case the bottleneck exists between the
server and the intermediary.

# Considerations

## Why use an End-to-End Header Field?

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

## Can an Intermediary Send it's own Signal?

There might be a benefit in recommending a coalescing intermediary to embed its
own prioritization hints into the HTTP request that it forwards to the backend
server, as otherwise the Priority header field would not be as helpful to the
backend (see {{fairness}}).

One way of achieving that, without dropping the original signal, would be to let
the intermediary express its own signal using the Priority header field, at the
same time transplanting the original value to a different header field.

As an example, when a client sends an HTTP request carrying a priority of
`urgency=-1` and the intermediary wants to instead associate
`urgency=0; progressive=?1`, the intermediary would send a HTTP request that
contains to the following two header fields to the backend server:

~~~
priority = urgency=0; progresive=?1
original-priority = urgency=-1
~~~

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
established by {{!RFC7540}}:

Name:
: SETTINGS_HEADER_BASED_PRIORITY:

Code:
: 0xTBD

Initial value:
: 0

Specification:
: This document

This specification registers the following entry in the HTTP/2 Frame Type
registry established by {{?RFC7540}}:

Frame Type:
: PRIORITY_UPDATE

Code:
: 0xF

Specification:
: This document

This specification registers the following entries in the HTTP/3 Frame Type
registry established by {{!I-D.ietf-quic-http}}:

Frame Type:
: PRIORITY_UPDATE

Code:
: 0xF

Specification:
: This document

--- back

# Acknowledgements

Roy Fielding presented the idea of using a header field for representing
priorities in <http://tools.ietf.org/agenda/83/slides/slides-83-httpbis-5.pdf>.
In <https://github.com/pmeenan/http3-prioritization-proposal>, Patrick Meenan
advocates for representing the priorities using a tuple of urgency and
concurrency.

The motivation for defining an alternative to HTTP/2 priorities is drawn from
discussion within the broad HTTP community. Special thanks to Roberto Peon,
Martin Thomson and Netflix for text that was incorporated explicitly in this
document.

Many thanks to Robin Marx, Patrick Meenan and Ian Swett for their feedback.

# Change Log

## Since draft-kazuho-httpbis-priority-02

* Consolidation of the problem statement (#61, #73)
* Define SETTINGS_PRIORITIES for negotiation (#58, #69)
* Define PRIORITY_UPDATE frame for HTTP/2 and HTTP/3 (#51)
* Explain fairness issue and mitigations (#56)

## Since draft-kazuho-httpbis-priority-01

* Explain how reprioritization might be supported.

## Since draft-kazuho-httpbis-priority-00

* Expand urgency levels from 3 to 8.
