---
title: "Content Origins for Web Packages"
abbrev: "Content Origins"
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
    title: "Permissions API"
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


# Content-Based Origin Definition {#content-origin}

A content-based origin ascribes an identity to content based on the content
itself.  For instance, a web bundle
{{!BUNDLE=I-D.yasskin-wpack-bundled-exchanges}} is attributed a URI based on its
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

For instance, the origin of content comprising the single ASCII character "a" is
represented as `ni:///sha-256;ypeBEsobvcr6wjGzmiPcTaeG7_gUfE5yuYB3ha_uSLs`.

In tuple form, the origin is comprised of the scheme ("ni") and a host equal to
the concatenation of hash algorithm identifier, semi-colon, and
base64url-encoded hash function output.  The port number is always absent for an
origin in this form.

Design Note:
: This design assumes that there won't be a hash-based URI scheme developed for
  bundles.  There is some advantage in having a URI scheme for bundled content,
  especially if that could support this use case.  For instance, state transfer
  could be initiated *toward* another bundle.


## Content Type Malleability

The hash used to generate this origin does not include any indication of content
type or the source of information.  If it were possible for the same content to
be interpreted as valid instances of multiple different content types, that
might be exploited by an attacker.

A user agent SHOULD enact a policy of only attributing a content-based origin to
content that is unambiguously interpreted in a single form.  If it were possible
for content to be interpreted as valid in two different forms that are
attributed to a content-based origin, that might be exploited by an attacker.
For instance, user agents might only allow web bundles and HTML files to be
assigned a content-based origin and ensure that the content of each cannot be
mistaken for the other.


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
to be regarded as equal to `sha-384;VKWbnyKwuAiA2EJ-VIt8I6vYc0huHwNd
zpzWl-hRdQM8qojm1XvDXvrgta_TFF8x` (note space added to meet formatting
constraints).  This requires that this equivalence is known to the user agent.

Of the hash algorithms defined in {{?NI}}, only "sha-256" is permitted for use
with content-based origins.


# Transfer to HTTPS Origin

In order to transfer state to a target origin, the server for that origin needs
to be contacted.  Initiating transfer likely requires that an API be defined for
use by the content of the bundle.

A user agent MAY request user permission to make a transfer.  This might be
conditioned on what state is being transferred.  For instance, a user agent
might prompt before transferring permissions {{PERMISSIONS}}; see
{{transfer-permissions}}.

After a state transfer is initiated, the user agent retrieves a well-known
resource on the target origin.  The content of the resource is hashed (as in
{{content-origin}} and the resulting value is hashed again.  This is formed into
a well-known URL and retrieved (for instance, using HTTP GET).  If the resulting
resource contains the hash of the content, the server indicates willingness to
receive the transfer and also demonstrates knowledge of the contents of the
content.

A state transfer URL is formed into a URL by concatenating the .well-known
prefix for the target origin (e.g., `https://example.com/.well-known/`), the
well-known identifier `state-transfer/`, the hash algorithm identifier (from
{{!NI}}), a semi-colon (';'), and the base64url-encoded value of the
doubly-hashed content.

Note:
: The response to a state transfer can be served from cache.

If the transfer is successful, state associated with the content-based origin
will be transferred to the target origin.  The user agent MAY either remember
the identity of the content-based origin and consider any content-based origin
to be equal to the transferred origin.

A transfer can be unsuccessful in two ways.  If the target origin cannot be
contacted successfully, the content-based origin continues to exist.  Any API
might indicate that the attempt failed.  Failure to transfer state is expected
when the user agent is offline.  After failing to contact the target origin, a
transfer can be re-attempted.

A valid, final HTTP response that indicates anything other than a server error
(that is, 5xx status codes) without including the hash pre-image in the response
body MUST be treated as a failed migration.  After a failed migration,
information about the target origin SHOULD be removed from interfaces related to
the content-based origin, except for diagnostic purposes.  The content-based
origin can continue to exist but further attempts to transfer state MUST
immediately fail.

Any valid HTTP response, successful or not, MUST cause data associated with the
content-based origin to be cleared as though clear-site-data were invoked
{{CLEAR-DATA}}.


## Transfer Target

Content that makes a commitment to transfer to a particular origin enables
transfer of state to that origin.  It does so by identifying the URI of a target
resource to load.  After a transfer is successfully completed, the user agent
loads the transfer target URI.

To enable transfer, a bundle MUST include an attribute (encoding TBD) that
indicates the transfer target.  The origin of the transfer target is the target
origin.  Only the target origin can be the ultimate recipient of the content.

Note:
: Altering the target origin will cause the content hash to assume a new value,
  thereby generating a new content-based origin.  Two different origins cannot
  share state.

A commitment in this form allows user agents to present the target origin in
interfaces.  While no strong assurances can be made about the attribution of
content to this origin, this might make it easier to generate user interfaces.


## State Transfer

Only limited state information can be transferred between origins.  This is
limited to those items that are identified here, or those that are added in
later extensions.  This document describes how content ({{transfer-content}}),
stored data {{transfer-storage}}), and permissions {{transfer-permissions}} are
transferred.

Upon successful completion of transfer, if loading the transfer target resource
produces an HTML document, an event is delivered to content of that resource
that describes what information was transferred.


### Transfer of Content {#transfer-content}

A bundle contains multiple resources that might be usable by the origin to which
state is transferred.  Rather than require that content

For content on the origin to which state was transferred, content MAY be loaded
from the bundle if the Subresource Integrity hash {{SRI}} matches.  This avoids
the negative effects described in {{SRI-ADDRESSABLE}} as the target origin is
required to demonstrate knowledge of the contents of the bundle.

Though a bundle count contain content that is ordinarily available from origins
other than the target origin, content from other origins MUST NOT be loaded from
the bundle in place of making the request directly.


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


## Communication Between Origins

Without knowledge of the content of a resource, or bundle of resources, a
content-based origin will be impossible to guess.  This means that communication
is only possible if the frame in which the content is loaded is known to an
origin that attempts communication.


# Security Considerations

This design avoids questions about having to attribute authority to a static
bundle of content.  Instead, it creates a new origin type and enables the
transfer of state from one origin to another.

Transfer of state to an origin with an "http://" origin is not possible.  This
mechanism depends on the target origin being strongly authenticated.


# IANA Considerations

TODO: well-known prefix for state-transfer.



--- back

# AMP Use Case

One major driver for

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
