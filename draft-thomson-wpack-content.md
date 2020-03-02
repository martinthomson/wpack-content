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
  HTML:
    title: "HTML"
    seriesinfo:
      WhatWG: Living Standard
    author:
      -
        ins: WhatWG
        org: WhatWG


--- abstract

Making content available to clients that are unable or unwilling to contact a
web origin enables new means of acquiring content.  This document describes a
method for taking application state accumulated by an offline user agent in
relation to a piece of content and making that state available in a fully online
context.  This enables continuous use of content, starting from a state where
the user agent does not contact an origin and ending with 

This document proposes an update to the definition of Origin in RFC 6454.


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


# Content-Based Origin Definition

A content-based origin ascribes an identity to content based on the content
itself.  For instance, a web bundle
{{!BUNDLE=I-D.yasskin-wpack-bundled-exchanges}} is attributed a URI based on its
content alone.

The sequence of bytes that comprises the content or bundled content is hashed
using a hash function that is strongly resistant to collision attack, such as
SHA-256 {{?SHA-2=RFC6234}}.  The resulting hash is encoded using the Base 64
encoding with an URL and filename safe alphabet {{!BASE64=RFC4648}}.

This can be formed into the ASCII or Unicode serialization of an origin based on
the Named Information URI scheme {{!NI=RFC6920}}.  This URI is formed from the
string "ni:///", the identifier for the hash algorithm (see Section 9.4 of
{{!NI}}); a semi-colon (";"), and the base64url encoding of the hash function
output.

For instance, the origin of content comprising the single ASCII character "a" is
represented as `ni:///sha-256;ypeBEsobvcr6wjGzmiPcTaeG7_gUfE5yuYB3ha_uSLs`.

In tuple form, the origin is comprised of the scheme ("ni") and a host equal to
the concatenation of hash algorithm identifier, semi-colon, and
base64url-encoded hash function output.  The port number is always absent for an
origin in this form.


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
instance, the
window.postMessage API {{HTML}} allows content to target a specific origin and to
identify a source origin 

Of the hash algorithms defined in {{?NI}}, only "sha-256" is permitted for use
with content-based origins.


# Transfer to HTTPS Origin

In order to transfer state to a 

## Commitment to a Target Origin

Content that makes a commitment to transfer to a particular origin enables
transfer of state to that origin.

To enable transfer, a bundle MUST include an attribute (encoding TBD) that
indicates the target origin.  Only that origin can be the ultimate recipient of
the content.  

Note:
: Altering the target origin will cause the content hash to assume a new value,
  thereby generating a new content-based origin.

A commitment in this form allows user agents to present the target origin in
interfaces.  While content


## Transfer of Content From Mixed Origins


## Transfer of Content 

## Communication Between Origins


# Security Considerations

Transfer of state to an origin with an "http://" origin is not possible.  This
mechanism depends on 


# IANA Considerations

This document has no IANA actions.



--- back

# AMP Use Case

One major driver for 

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
