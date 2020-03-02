---
title: "Content-Based Origins for the Web"
abbrev: "Content-Based Origins"
docname: draft-thomson-wpack-content-latest
category: std
updates: 6454

ipr: trust200902
area: General
workgroup: wpack
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Thomson
    name: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net

normative:

informative:
  CLEAR-DATA:
    title: "Clear Site Data"
    seriesinfo:
      W3C: ED
    date: 2017-11-13
    author:
      -
        name: Mike West
        organization: Google

  HTML:
    title: "HTML"
    seriesinfo:
      WhatWG: Living Standard
    date: 2020
    author:
      -
        ins: WhatWG
        org: WhatWG

  INDEXEDDB:
    title: "Indexed Database API 3.0"
    seriesinfo:
      W3C: ED
    date: 2020-02-29
    author:
      -
        name: Ali Alabbas
        organization: Microsoft
      -
        name: Joshua Bell
        organization: Google

  PERMISSIONS:
    title: "Permissions"
    seriesinfo:
      W3C: ED
    date: 2019-10-30
    author:
      -
        name: Mounir Lamouri
        organization: Google
      -
        name: Marcos Cáceres
        organization: Mozilla
      -
        name: Jeffrey Yasskin
        organization: Google

  SRI:
    title: "Subresource Integrity"
    seriesinfo:
      W3C: ED
    date: 2020-01-04
    author:
      -
        name: Devdatta Akhawe
        organization: Dropbox
      -
        name: Frederik Braun
        organization: Mozilla
      -
        name: François Marier
        organization: Mozilla
      -
        name: Joel Weinberger
        organization: Google

  SRI-ADDRESSABLE:
    title: "Subresource Integrity Addressable Caching"
    date: 2016-10-15
    author:
      -
        name: Brad Hill
        organization: Facebook


--- abstract

Making content available to clients that are unable or unwilling to contact a
web origin enables new means of acquiring content.  This document describes a
method for taking application state accumulated by an offline user agent in
relation to a piece of content and making that state available in a fully online
context.  This enables continuous use of content, starting from a state where
the user agent does not contact an origin and ending with

This document proposes an update to the definition of Origin in RFC 6454.  It
also proposes changes that would affect HTML, which is outside of the remit of
the IETF.


--- middle

# Introduction

The web relies on the concept of origins {{!ORIGIN=RFC6454}} as the primary
means of identification for resources.  A strongly authenticated HTTPS origin is
the basis for many security decisions.  However, establishing this identity
relies on having an active TLS {{?TLS=RFC8446}} connection to a server that is
considered authoritative for the origin; see Section 11 of
{{!HTTP=I-D.ietf-httpbis-semantics}}.

A user agent that is offline when content is received cannot establish this
authority. Similarly, a user agent might be unwilling to contact a server at the
time that content is received.  One reason for not contacting a server might be
to avoid leaking information about the context in which content was referenced.
In either case, content cannot be attributed to the origin at the time it is
received.

In operation, content might be delivered by any means other than a TLS
connection to the intended server.  Then there might be a period during which
the content is used.  For instance, the content might be a web page that is
capable of functioning offline, though certain functions might be unavailable.

At some after content is received, the user agent might want to establish a
connection to the server and continue use.  True continuity of use could depend
on state established during the period that the user agent did not contact the
server.

This document proposes a design whereby content can be ascribed an identity that
is based solely on that content.  Together with a means of transferring control
over state established by one origin to another, this allows content to be
delivered offline and used with the ability to transition to a full online
interaction with a web server.

Alternative designs attempt to provide a means to ascribe an HTTPS origin to
content through the use of signatures
{{!SXG=I-D.yasskin-http-origin-signed-responses}}.  Those designs depend on
finding a way to modify the core concept of web origins that allows for
object-based authority.  This avoids the problems associated with transfer of
state between origins.  However, a fundamental change to the way in which
authority is determined requires the use of a number of safeguards.  These
safeguards contain both technical mechanisms and usage constraints.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document uses the term "user agent" consistent with the usage in both
{{!HTTP}} and {{HTML}}.  Colloquially, "user agent" is a web browser.


# Content-Based Origin Definition {#content-origin}

A content-based origin ascribes an identity to content based on the content
itself.  For instance, a web bundle
{{!BUNDLE=I-D.yasskin-wpack-bundled-exchanges}} is assigned a URI based on its
content alone.

The sequence of bytes that comprises the content or bundled content is hashed
using a hash function that is strongly resistant to collision and pre-image
attack, such as SHA-256 {{?SHA-2=RFC6234}}.  The resulting hash is encoded using
the Base 64 encoding with an URL and filename safe alphabet {{!BASE64=RFC4648}}.

This can be formed into the ASCII or Unicode serialization of an origin based on
the Named Information URI scheme {{!NI=RFC6920}}.  This URI is formed from the
string "ni:///", the identifier for the hash algorithm (see Section 9.4 of
{{!NI}}); a semi-colon (";"), and the base64url encoding of the hash function
output.  Though this uses the ni URL form, the authority and query strings are
omitted from this serialization.

For instance, the origin of content comprising the single ASCII character 'a' is
represented as `ni:///sha-256;ypeBEsobvcr6wjGzmiPcTaeG7_gUfE5yuYB3ha_uSLs`.

In tuple form, the origin is comprised of the scheme ("ni") and a host equal to
the concatenation of hash algorithm identifier, semi-colon, and
base64url-encoded hash function output.  The port number is always absent for an
origin in this form.

Design Note:
: This design currently assumes that there won't be a hash-based URI scheme
  developed for bundles.  There is some advantage in having a URI scheme for
  bundled content, especially if that could support this use case.  For
  instance, state transfer could be initiated *toward* another bundle.

This definition of origin for named information (`ni://`) URIs extends the
definition of origin in Section 4 of {{!ORIGIN}}.


## Content Type Malleability

The hash used to generate this origin does not include any indication of content
type or the source of information.  If it were possible for the same content to
be interpreted as valid instances of multiple different content types, that
might be exploited by an attacker.

A user agent SHOULD enact a policy of only attributing a content-based origin to
content that is unambiguously interpreted in a single form.  If it were possible
for content to be interpreted as valid for multiple content types and those
could be attributed to the same content-based origin, that might be exploited by
an attacker.  A sample policy might only allow web bundles and HTML files to be
assigned a content-based origin; as the first bytes of these formats are
unambiguously different, this ensures that the same content to be interpreted as
valid for both content types.


## Hash Agility

This design depends to some extent on the hash algorithm remaining constant
during the time that the content-based origin is used.  A change in hash
algorithm changes the identity of the resource.

It is possible to silently transfer state from a content-based origin derived
using one hash algorithm to another without affecting the content of the origin
itself.  It is not often that content depends on knowing its own identity.
However, the identity of the origin might be made visible to other origins.  For
instance, the window.postMessage API {{HTML}} allows content to target a
specific origin for sending messages and to identify the source origin of
incoming messages.

For the purposes of determining equality, user agents might consider hashes of
the same content with different hash algorithms to be equal.  For instance, a
user agent might consider `sha-256:ypeBEsobvcr6wjGzmiPcTaeG7_gUfE5yuYB3ha_uSLs`
to be equal to `sha-384;VKWbnyKwuAiA2EJ-VIt8I6vYc0huHwNd
zpzWl-hRdQM8qojm1XvDXvrgta_TFF8x` (note space added to meet formatting
constraints).  This requires that this equivalence is known to the user agent.

Of the hash algorithms defined in {{?NI}}, only "sha-256" is permitted for use
with content-based origins.  This usage has no need for a truncated hash as the
value does not usually need to be manually entered.  Furthermore, the use of
multiple hash algorithms introduces complexity.


# Transfer to HTTPS Origin {#transfer}

In order to transfer state to a target origin, the server for that origin needs
to be contacted.  Initiating transfer likely requires that an API be defined for
use by the content of the bundle.

A transfer is only possible where a transfer target URL is known for content;
see {{transfer-target}}.

A user agent MAY request user permission to transfer some state.  This might be
conditioned on what state is being transferred.  For instance, a user agent
might prompt before transferring permissions {{PERMISSIONS}}; see
{{transfer-permissions}}.

After a state transfer is initiated, the user agent fetches the transfer target
URL.  The source content is hashed twice; once as described in
{{content-origin}} with the resulting value is hashed again.  The doubly-hashed
value is encoded into a Sec-Content-Origin header field and added to the request
for the transfer target URL.  If a Sec-Content-Origin header field in the
response contains the hash of the content (that is, the pre-image of the value
from the request), that indicates that the server is both willing to receive the
transfer and that the server knows the content.

The value of the Sec-Content-Origin header field is expressed as a structured
header {{!SH=I-D.ietf-httpbis-header-structure}} dictionary with keys being the
hash algorithm identifier (from {{!NI}}) and byte sequence values of the hash.

Multiple values can be provided with different hash algorithm identifiers.
The values from the response correspond to the values in the request with the
same hash algorithm identifier.  For the transfer to be successful, clients MUST
validate that at least one value in the response hashes to the corresponding
value in the request; clients SHOULD validate all provided values.  If the
content does not hash to the any provided value, the transfer is unsuccessful.

Note:
: The response to a state transfer can be served from cache.


## Successful Transfer

If the transfer is successful, state associated with the content-based origin
will be transferred to the target origin.  The user agent MAY either remember
the identity of the content-based origin and consider any content-based origin
to be equal to the transferred origin.

For example, assuming that a single 'a' character is valid content, then the
client would include the following in a request (line breaks are added to
examples for formatting reasons):

~~~
Sec-Content-Origin:
  sha-256=:v106/7c+/S7Gw2rTES3ZM+/tY8Thy//PqI4nWcFE8tg=:
~~~

A successful state transfer would occur if the server indicated:

~~~
Sec-Content-Origin:
  sha-256=:ypeBEsobvcr6wjGzmiPcTaeG7/gUfE5yuYB3ha/uSLs=:
~~~

A user agent that automatically follows redirections (3xx status codes) MUST
allow the server to redirect to a resource that provides the response.


## Unsuccessful Transfer

A transfer can be unsuccessful in two ways.  If the target origin cannot be
contacted successfully, the content-based origin continues to exist.  Any API
might indicate that the attempt failed.  Failure to transfer state is expected
when the user agent is offline.  After failing to contact the target origin, a
transfer can be attempted at a later time.  This causes navigation to fail and
the user agent MAY display a URL from the content-based origin (TODO: this
requires that we define a means of identification for content inside bundles).

A valid, final HTTP response that indicates anything other than a server error
(that is, 5xx status codes) without including the hash pre-image in the response
MUST be treated as a failed migration.  After a failed migration, information
about the target origin SHOULD be removed from interfaces related to the
content-based origin, except for diagnostic purposes.  The content-based origin
can continue to exist but further attempts to transfer state MUST immediately
fail.

After a valid HTTP response, the user agent navigates to the transfer target URL
or any resource that was included in any redirections.

Any valid HTTP response, successful or not, MAY cause data associated with the
content-based origin to be cleared as though clear-site-data were invoked
{{CLEAR-DATA}}.


## Transfer Target {#transfer-target}

To enable transfer, content MUST include an attribute that indicates the
transfer target URL.  The origin of the transfer target URL is the target
origin.  Only the target origin can be the target of a state transfer.  After a
successful transfer, the user agent loads the transfer target URL.

Note:
: Including the transfer target URL in content means that altering the value
  will cause the content hash to assume a new value.  This generates a new
  content-based origin.  Two different origins cannot share state.

A commitment in this form allows user agents to present the target origin in
interfaces.  While no strong assurances can be made about the attribution of
content to this origin, this might make it easier to generate user interfaces.

Different content types might provide a way of encoding the transfer target URL.
The URL could be provided external to the content, but this requires some
unspecified means of ensuring that the content-based origin depends on the value
of the URL; this document only supports content that can encode the transfer
target URL.


## State Transfer

Only limited state information can be transferred between origins.  This is
limited to those items that are identified here, or those that are added in
later extensions.  This document describes how content ({{transfer-content}}),
stored data {{transfer-storage}}), and permissions {{transfer-permissions}} are
transferred.

Upon successful completion of transfer, if loading the transfer target resource
produces an HTML document, an event is delivered to content of that resource
that describes what information was transferred.  \[\[TBD: The form of the
event.]]


### Transfer of Content {#transfer-content}

A bundle contains multiple resources that might be usable by the origin to which
state is transferred.  Rather than require that content

For content on the origin to which state was transferred, content MAY be loaded
from the bundle if the Subresource Integrity hash {{SRI}} matches.  This avoids
the negative effects described in {{SRI-ADDRESSABLE}} as the target origin is
required to demonstrate knowledge of the contents of the bundle.

Though a bundle could contain content that is ordinarily available from origins
other than the target origin, content from other origins MUST NOT be loaded from
the bundle in place of making the request to those other origins directly.

A server might use the Digest header field
{{?DIGEST=I-D.ietf-httpbis-digest-headers}} to indicate that content matches
content from a bundle. A client might not know what content from the bundle
corresponds to a given resource, but it can include a Want-Digest header field
in requests that might result in content that is loaded from the bundle.  For
instance, if content sourced from a bundle refers to a resource in the bundle
with a relative URL and the same content for the referring content is provided
by a server, then a user agent might infer that it is possible that referenced
content will match what appears in the bundle.

\[\[TODO: is there a case here for a conditional If-Digest header field here?]]


### Transfer of Storage {#transfer-storage}

Upon transfer, any indexedDB stores {{INDEXEDDB}} are transferred to the target
origin.  The names of any databases are prefixed with the origin from which they
came.  If a database with the same name exists at the destination, a new name is
selected.

The event that indicates transfer will list the databases that were transferred
by name.


### Transfer of Permissions {#transfer-permissions}

As part of a state transfer, any persistent permissions that a user granted the
content-based origin might be transferred to the target origin.

Whether any given permission is transferred and what conditions are attached to
the transferral will depend on the policy of the user agent.  In some cases, it
might be considered appropriate to discard permissions and request them anew.
As transferrance is a one-time event, causing prompts to be reinitiated might
not be too much of an imposition.


## Navigation Transfers

Initiating transfer on navigation enables pre-loading of content from bundles.

To address this use case, the user agent navigates to a bundle.  That bundle
might be served from the site providing the link to ensure that information
about the linking page is not revealed to the target origin prior to navigation.

Navigating to the bundle causes an immediate load of the content of the bundle.
Once the transfer target URL is known, the user agent navigates to that URL.
The user agent MUST fetch the transfer target URL and include a
Sec-Content-Origin header field in the request as described in {{transfer}}.
This request MAY also include the ETag header field from the bundled content.

The resource at the transfer target URL might return a 304 (Not Modified) status
code in response to a request that contains a Sec-Content-Origin header field if
the content provided in the bundle known to the server to be sufficient.

\[\[TBD: Maybe it is best not to overload this header field with semantics of a
conditional request and the above If-Digest is the right approach.]]

The result is that the state from the bundle is transferred to the target
origin.  As the navigation is immediate, no state will have been created within
the bundle so the transfer of state can be skipped by the browser if navigation
from a previously unused content-based origin is successful.  It is possible
that the content-based origin might have previously used, in which case state
transferrance might occur.


## Communication Between Origins

Without knowledge of the content of a resource, or bundle of resources, a
content-based origin will be impossible to guess.  This means that communication
is only possible if the frame in which the content is loaded is known to an
origin that attempts communication.


# Security Considerations

This design avoids questions about having to attribute authority to a static
bundle of content.  Instead, it creates a new origin type and enables the
transfer of state from one origin to another.  The target origin can decide
whether to accept a transfer, thereby avoiding any questions of poisoning of
content.  If content is found to be compromised, an origin can subsequently
refuse to accept a transfer from that content.

Transfer of state to an origin with an "http://" origin is not possible.  This
mechanism depends on the target origin being strongly authenticated.


# IANA Considerations

TODO: Register the Sec-Content-Origin header field.



--- back

# Acknowledgments
{:numbered="false"}

This work is hardly original, nor is it an individual effort.  TODO:
acknowledge.
