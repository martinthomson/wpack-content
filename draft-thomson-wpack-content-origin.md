---
title: "Content-Based Origins for the Web"
abbrev: "Content-Based Origins"
docname: draft-thomson-wpack-content-origin-latest
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
    target: "https://w3c.github.io/webappsec-clear-site-data/"
    author:
      -
        name: Mike West
        organization: Google

  HTML:
    title: "HTML"
    seriesinfo:
      WhatWG: Living Standard
    date: 2020-03-04
    target: "https://html.spec.whatwg.org/"
    author:
      -
        ins: WhatWG
        org: WhatWG

  IF-DIGEST:
    title: "Conditional HTTP Requests Using Digests"
    seriesinfo:
      Internet-Draft: draft-thomson-http-if-digest-00
    date: 2020-03-09
    target: "https://tools.ietf.org/html/draft-thomson-http-if-digest-00"
    author:
      -
        name: Martin Thomson
        organization: Mozilla

  INDEXEDDB:
    title: "Indexed Database API 3.0"
    seriesinfo:
      W3C: ED
    date: 2020-02-29
    target: "https://w3c.github.io/IndexedDB/"
    author:
      -
        name: Ali Alabbas
        organization: Microsoft
      -
        name: Joshua Bell
        organization: Google

  LOADING:
    title: "Loading Signed Exchanges"
    date: 2020-02-21
    target: "https://wicg.github.io/webpackage/loading.html"
    author:
      -
        name: Jeffrey Yasskin
        organization: Google

  PERMISSIONS:
    title: "Permissions"
    seriesinfo:
      W3C: ED
    date: 2019-10-30
    target: "https://w3c.github.io/permissions/"
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

  SERVICE-WORKERS:
    title: "Service Workers 1"
    seriesinfo:
      W3C: ED
    date: 2020-02-24
    target: "https://w3c.github.io/ServiceWorker/v1/"
    author:
      -
        name: Alex Russell
        organization: Google
      -
        name: Jungkee Song
        organization: Microsoft
      -
        name: Jake Archibald
        organization: Google
      -
        name: Marijn Kruisselbrink
        organization: Google

  SRI:
    title: "Subresource Integrity"
    seriesinfo:
      W3C: ED
    date: 2020-01-04
    target: "https://w3c.github.io/webappsec-subresource-integrity/"
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
    target: "https://hillbrad.github.io/sri-addressable-caching/sri-addressable-caching.html"
    author:
      -
        name: Brad Hill
        organization: Facebook

  UNSIGNED:
    title: "Navigation to Unsigned Web Bundles"
    date: 2020-02-20
    target: "https://github.com/WICG/webpackage/blob/master/explainers/navigation-to-unsigned-bundles.md"
    author:
      -
        name: Jeffrey Yasskin
        organization: Google



--- abstract

Making content available to clients that are unable or unwilling to contact a
web origin enables new means of acquiring content.  This document describes a
method for taking application state accumulated by an offline user agent in
relation to a piece of content and making that state available in a fully online
context.  This enables continuous use of content, starting from a state where
the user agent does not contact an origin and ending with online
interactions with an HTTPS origin.

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

Content-based origins are proposed as an alternative to signed exchanges
{{!SXG=I-D.yasskin-http-origin-signed-responses}}, which ascribe an HTTPS
origin to content through the use of signatures. Signed exchanges depends on
finding a way to modify the core concept of web origins that allows for
object-based authority. Signed exchanges avoid any problems arising from
transfer of state between origins. However, a fundamental change to the way in
which authority is determined requires the use of a number of safeguards. These
safeguards contain both technical mechanisms and usage constraints. These
constraints could be operationally challenging to meet, but violating them
could have consequences for security.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document uses the term "user agent" consistent with the usage in both
{{!HTTP}} and {{HTML}}.  Colloquially, "user agent" is a web browser.


# Overview and Comparisons

This section provides an overview of how content-based origins might be used
and compares that to designs based on signed exchanges.

The most interesting scenario involves the transition from an state where the
target origin has not been contacted (that is, the user agent is either offline
or effectively so), to one where the user agent contacts the target origin (the
user agent is online).


## Offline-to-Online Transition for Content-Based Origins

For a content-based origin, content is loaded from a bundle. After loading the
bundle, the browser treats content in the bundle as existing in an origin
unique to the content of the bundle; see {{content-origin}}.

If the bundle contains a transfer target URL (see {{transfer-target}}), the
browser then attempts to load that resource, providing an indication to the
target origin that a transfer from the content-based origin is possible. If the
origin accepts the transfer of state, state is transferred from the
content-based origin to the target origin. This sequence is shown in
{{ex-cbo}}.

~~~
               +---------------+
               |  Load Bundle  |
               +---------------+
                       |
                       v
               +---------------+       +---------+
 Aborted +-----| Content-Based |=======|  State  |
Transfer +---->|    Origin     |       +---------+
               +---------------+            :
                       |  State Transfer    :
                       | IFF Origin Accepts :
                       v                    v
               +---------------+       +---------+
               |   Online at   |=======|  State  |
               | Target Origin |       +---------+
               +---------------+
~~~
{: #ex-cbo title="State Transfer for Content-Based Origins"}

Failure to connect to the target origin - such as when the browser has no
active Internet connection - causes the redirect to be aborted. The browser
continues to display the bundle content.

If the origin does not accept the transfer, the browser shows content from the
target origin and any state from the bundle is destroyed.


## Offline-to-Online Transition for Signed Exchanges

With {{LOADING}} and {{!SXG=I-D.yasskin-http-origin-signed-responses}}, content
is loaded from a signed bundle. After loading the bundle, if the signature is
considered valid, the browser stores bundled content in a cache that is
specific to the target origin. The browser then redirects to the request (or
target) URL from the bundle. The browser uses content from the bundle in place
of fetching from the origin.  This is illustrated in {{ex-sxg}}.

~~~
    +-----------------+
    |   Load Bundle   |
    +-----------------+
            |             ..If signature is trusted...
            v             :                          :
    +-----------------+   :   +-----------------+    :
    | Check Signature +------>| Store Exchanges |    :
    +-----------------+   :   +-----------------+    :
  Redirect  |             :                          :
            v             :                          :
    +-----------------+   :   +-----------------+    :
    |    Online at    |<------|  Use Exchanges  |    :
    |   Request URL   |   :   +-----------------+    :
    +-----------------+   :...........................
~~~
{: #ex-sxg title="Content Use with Signed Exchanges"}

Having signed content from the bundle allows use of that content prior to
connecting to the origin. Importantly, it allows attribution of any operations
to that origin. Optimizations that otherwise might not be possible are enabled
because resources from the bundle can be treated as if they were loaded from
the target origin, but the browser does not need to make a request to the
origin.

Critically, if the browser is fully offline, it can decide to operate with the
bundle without having to connect to the origin. If the bundled content is
signed and trusted, the application to operate offline. Any actions performed
act on state for the target origin.

Note:
: While Service Workers {{SERVICE-WORKERS}} can offer a similar experience,
  they require that the browser be online initially to load the service worker
  script. This design requires no prior interaction with the target origin.

Relative to content-based origins, the use of signatures avoids the need to
connect to the target origin and confirm knowledge of the content. For cases
where the transition to online status is intended to be immediate, signed
exchanges allow a redirect to happen without any requests being made. A signed
exchange will therefore display content that is attributed to the target origin
faster.

Failure to validate a signature causes a redirect to an online location. If the
client is offline, that origin will be inaccessible. If the client is online,
none of the optimizations afforded by the bundle are available and this appears
to be a normal redirection.

\[\[Note that it is unclear whether a browser might be able to fall back to
treating a signed bundle as unsigned if the signature is bad.]]


## Comparison with Signed Exchanges

Signed exchanges attempt to address the problem of attributing content to an
origin.  In effect, they add an object-based security model to the existing
channel-based model used on the web.  Signatures over bundles (or parts
thereof) are used by an origin to attest to the contents of a bundle.

Having two security models operate in the same space potentially creates an
exposure to the worst properties of each model. To reduce the chances that the
drawbacks of the newly-added object-based model affect existing channel-based
usages, signed exchanges include a number of hardening measures. In addition to
signatures, there are required modifications to certificates, constraints on
validity periods, and a range of limitations on the types of content that can
be signed.

In comparison, content-based origins do not require signatures. Questions of
validity only apply at the point that a state transfer is attempted. However,
this has drawbacks also. Content is not attributed to origins and state is not
available to an online origin until a transfer is complete. This avoids the
complexity inherent to merging two different security models, but the process
of state transfer could be quite complicated in practice.

In terms of usability, the identity attributed to content from a content-based
origin is opaque and not particularly relatable. Though state might eventually
be adopted by some origin, communicating the true status of content could be
challenging. Finally, content-based origins aren't prevented from interacting
with HTTP origins, which could lead to surprising outcomes if existing code is
poorly unprepared for this possibility.


## Comparison with Unsigned Bundles

Unsigned bundles {{UNSIGNED}} proposes a model whereby bundles served by a
given origin could be declared to be part of that origin and trusted. Unsigned
bundles would otherwise be considered untrusted and would be isolated in some
fashion. This is similar to the way in which a content-based origin might be
transferred to a target origin.

That proposal divides bundles into trusted and untrusted bundles. Trusted
bundles would be able to access state from the origin that served them.

In comparison, content-based origins would always be used for bundles and the
distinction between trusted and untrusted is unnecessary. Content from the
bundle would be usable to a target origin that accepts a state transfer.


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
  developed for bundles.  It would be vastly preferable to have a URI scheme
  for bundled content.  It would then be possible to reference content in a
  bundle from outside the bundle, and internal references could be
  canonicalized. Importantly, state transfer could also be initiated *toward*
  another bundle.  That could be used to upgrade bundles and other nice
  features.

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


## Hash Agility {#hash-agility}

This design depends to some extent on the hash algorithm remaining constant
during the time that the content-based origin is used.  A change in hash
algorithm changes the identity of the resource.

It is possible to transfer state from a content-based origin derived using one
hash algorithm to another without affecting the content of the origin itself.
It is not often that content depends on knowing its own identity.  However, the
identity of the origin might be made visible to other origins.  For instance,
the window.postMessage API {{HTML}} allows content to target a specific origin
for sending messages and to identify the source origin of incoming messages.

For the purposes of determining equality, user agents might consider hashes of
the same content with different hash algorithms to be equal. For instance, a
user agent might consider `sha-256:ypeBEsobvcr6wjGzmiPcTaeG7_gUfE5yuYB3ha_uSLs`
to be equal to
`sha-384;VKWbnyKwuAiA2EJ-VIt8I6vYc0huHwNdzpzWl-hRdQM8qojm1XvDXvrgta_TFF8x`.
This requires that this equivalence is known to the user agent.

Of the hash algorithms defined in {{?NI}}, only "sha-256" is permitted for use
with content-based origins.  This usage has no need for a truncated hash as the
value does not usually need to be manually entered.  Furthermore, the use of
multiple hash algorithms introduces complexity.


# Transfer to HTTPS Origin {#transfer}

In order to transfer state to a target origin, the server for that origin needs
to be contacted.  Initiating transfer likely requires that an API be defined for
use by the content of the bundle.  Transfer might be automatically initiated when navigating to a URL from a bundle; see {{navigation}}.

A transfer is only possible where a transfer target URL is known for content;
see {{transfer-target}}.

After a state transfer is initiated, the user agent fetches the transfer target
URL.  The source content is hashed twice; once as described in
{{content-origin}}; then the resulting value is hashed again.  The twice-hashed
value is encoded into a Sec-Content-Origin header field and added to the request
for the transfer target URL.  If a Sec-Content-Origin header field in the
response contains the hash of the content (that is, the pre-image of the value
in the request), that indicates that the server is both willing to receive the
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

A successful state transfer would occur if the response from the server
included:

~~~
Sec-Content-Origin:
  sha-256=:ypeBEsobvcr6wjGzmiPcTaeG7/gUfE5yuYB3ha/uSLs=:
~~~

A user agent that automatically follows redirections (3xx status codes) MUST
allow the server to redirect to a resource that provides the response.
Redirection can change the origin that ultimately accepts the transfer. Any
redirection to an origin that is not strongly authenticated MUST cause the
transfer to fail.

After a successful transfer, the user agent MAY treat the content-based origin
as an alias for the origin to which state was transferred. This aliasing might
be particularly useful in addressing hash agility; see {{hash-agility}}.


## Unsuccessful Transfer

A transfer can be unsuccessful in two ways.  If the target origin cannot be
contacted successfully, the content-based origin continues to exist.  Any API
might indicate that the attempt failed.  Failure to transfer state is expected
when the user agent is offline.  After failing to contact the target origin, a
transfer can be attempted at a later time.  This causes navigation to fail and
the user agent MAY display a URL from the content-based origin.

\[\[TODO: Again, this requires that we define a means of identification for
content inside bundles.]]

A valid, final HTTP response that indicates anything other than a server error
(that is, 5xx status codes) or lack of authority (the 421 status code
{{!HTTP2=RFC7540}}) without including the hash pre-image in the response MUST
be treated as a failed migration. After a failed migration, information about
the target origin SHOULD be removed from interfaces related to the
content-based origin, except for diagnostic purposes. The content-based origin
MAY continue to exist but further attempts to transfer state MUST immediately
fail.

After a valid HTTP response, the user agent navigates to the transfer target URL
or any resource that was included in any redirections.

Any valid HTTP response, successful or not, MUST cause data associated with the
content-based origin to be cleared as though clear-site-data were invoked
{{CLEAR-DATA}}.


## Transfer Target {#transfer-target}

To enable transfer, content MUST include an attribute that indicates the
transfer target URL. If a bundle could contain multiple potential entry points,
each entry point for the bundle would separately specify a different transfer
target URL.

\[\[Intuitively, it seems prudent to limit transfer target URLs in the same
bundle to the same origin, but I don't have a concrete reason for doing so
right now.]]

A transfer target URL does not need to be specified if the intent is to never
support the ability to transfer to an online state.

The origin of the transfer target URL is the target origin. Only
the target origin can be the target of a state transfer. After a successful
transfer, the user agent loads the transfer target URL.

Note:
: Including the transfer target URL in content means that the content hash
  is dependent on its value.  This ensures that different content-based origins
  are produced if different transfer target URLs are used.  Two different
  origins cannot share state.

A commitment in this form could allow user agents to present the target origin
in interfaces. While no strong assurances can be made about the attribution of
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
that describes what information was transferred.

A user agent MAY request user permission to transfer some state.  This might be
conditioned on what state is being transferred.  For instance, a user agent
might prompt before transferring permissions {{PERMISSIONS}}; see
{{transfer-permissions}}.


### Transfer of Content {#transfer-content}

How content that is delivered in the bundle is used by the target origin is
probably the most important aspect of any design in this space. This document
proposes that all content in the bundle is adopted by the origin after a
transfer completes.

This design means that bundles need a way to describe how all their resources
map to URLs in the target origin. If it is possible for bundles to contain
content from multiple origins, content from other origins won't be accessible
after transfer without first making a request to the other origin.

Content loaded from a bundle MUST NOT be made available for use by origins
other than the target origin. For example, if the bundle includes a script that
maps to a URL on the target origin and a site at another origin loads that
script, then the user agent fetches the script. The user agent MAY use a
conditional request with If-None-Digest {{IF-DIGEST}} and the Digest header
field {{?DIGEST=I-D.ietf-httpbis-digest-headers}} to reduce the overhead of
these requests at its discretion, noting that this might introduce observable
timing signals.

Restricting content to use by the target origin avoids the negative effects
described in {{SRI-ADDRESSABLE}}. The target origin is required to demonstrate
knowledge of the contents of the bundle, which prevents that origin from being
poisoned by an attacker. For origins other than the target origin, content
cannot be emplaced through the use of a bundle.

Alternative methods for transfer of content might require per-resource
confirmation by the origin. That might be loosened so that resources that use
Subresource Integrity {{SRI}} do not require confirmation. But that increases
the need to provide integrity attributes at the point that resources are
referred to. If content is fetched programmatically, that might be
operationally challenging.


### Transfer of Storage {#transfer-storage}

Upon transfer, any indexedDB databases {{INDEXEDDB}} are transferred to the
target origin.  The names of any databases are prefixed with the origin from
which they came.  If a database with the same name exists at the destination, a
new name is selected.

The event that indicates transfer will list the databases that were
transferred, their names and any name changes.


### Transfer of Permissions {#transfer-permissions}

As part of a state transfer, any persistent permissions that a user granted the
content-based origin might be transferred to the target origin.

Whether any given permission is transferred and what conditions are attached to
the transferral will depend on the policy of the user agent. It might be
considered best to discard permissions and request each anew as necessary. As
transferral is a one-time event, causing prompts to be reinitiated might not be
too much of an imposition.


## Navigation Transfers {#navigation}

Initiating transfer on navigation enables pre-loading of content from bundles.

To address this use case, the user agent navigates to a bundle.  That bundle
might be served from the site providing the link to ensure that information
about the linking page is not revealed to the target origin prior to navigation.

Navigating to the bundle causes an immediate load of the content of the bundle.
This assumes that there is either a URL component that specifies a single entry
point for the bundle or that the bundle itself identifies the entrypoint.

If a transfer target URL is specified for the target resource, the user agent
navigates to that URL. The user agent MUST fetch the transfer target URL and
include a Sec-Content-Origin header field in the request as described in
{{transfer}}. This request MAY also include the ETag header field from the
bundled content.

Using If-None-Digest {{IF-DIGEST}} allows the resource at the transfer target
URL to return a 304 (Not Modified) status code in response to a request if the
content provided in the bundle known to the server to be sufficient.

The result is that the state from the bundle is transferred to the target
origin.  As the navigation is immediate, no state will have been created within
the bundle so the transfer of state can be skipped by the browser if navigation
from a previously unused content-based origin is successful.  It is possible
that the content-based origin might have previously used, in which case state
transferrance might occur.


## Communication Between Origins

Without knowledge of the content of a resource, or bundle of resources, a
content-based origin will be impossible to guess.  This means that communication
is only possible if the frame in which the content is loaded by the origin
attempting communication, or the content is known to that origin.


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

\[\[TODO: Register the Sec-Content-Origin header field.  Probably a lot more.]]



--- back

# Acknowledgments
{:numbered="false"}

This work is hardly original, nor is it an individual effort. \[\[TODO:
acknowledge.]]
