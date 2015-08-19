---
title: Architectural Considerations for Transport Evolution with Explicit Path Cooperation
abbrev: Explicit Cooperation
docname: draft-trammell-stackevo-explicit-coop-00
date: 2015-7-30
category: info
ipr: trust200902


author:
  -
    ins: B. Trammell
    name: Brian Trammell
    role: editor
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
informative:
informative:
  RFC0791:
  RFC2460:
  RFC5245:
  RFC6347:
  RFC7258:
  RFC7435:
  I-D.ietf-rtcweb-overview:
  I-D.ietf-taps-transports:
  I-D.hildebrand-spud-prototype:
  I-D.huitema-tls-dtls-as-subtransport:
  I-D.blanchet-iab-internetoverport443:
  I-D.baker-6man-hbh-header-handling:
  Saltzer84:
    title: End-to-End Arguments in System Design (ACM Trans. Comp. Sys.)
    author:
      -
        ins: J. H. Saltzer
      -
        ins: D. P. Reed
      -
        ins: D. D. Clark
    date: 1984  
  RFC0792:
  RFC2827:
  RFC4821:
  RFC6347:
  RFC7510:
  I-D.hildebrand-spud-prototype:
  I-D.huitema-tls-dtls-as-subtransport:
  I-D.trammell-stackevo-newtea:
  I-D.iab-semi-report:
  I-D.ietf-dart-dscp-rtp:

--- abstract

The IAB Stack Evolution in a Middlebox Internet (SEMI) workshop in Zurich in
January 2015 and the follow-up Substrate Protocol for User Datagrams (SPUD)
BoF session at the IETF 92 meeting in Dallas in March 2015 identified the
potential need for a UDP-based encapsulation to allow explicit cooperation
with middleboxes while using encryption at the transport layer and above to
protect user payload and metadata from inspection and interference. This
document proposes a set of architectural considerations for such approaches.

--- middle

# Introduction and Motivation

The current work of the IAB IP Stack Evolution Program is to support the
evolution of the Internet's transport layer and its interfaces to other layers
in the Internet Protocol stack. The need for this work is driven by two
trends. First is the development and increased deployment of cryptography in
Internet protocols to protect against pervasive monitoring {{RFC7258}}, which
will break many middleboxes used in the operation and management of
Internet-connected networks and which assume access to plaintext content. An
additional encapsulation layer to allow selective, explicit metadata exchange
between the endpoints and devices on path to replace ad-hoc packet inspection
is one approach to retain network manageability in an encrypted Internet.

Second is the increased deployment of new applications (e.g. interactive media
as in RTCWEB {{I-D.ietf-rtcweb-overview}}) for which the abstractions provided
by today's transport APIs (i.e., either a single reliable stream as in
SOCK_STREAM over TCP, or an unreliable, unordered packet sequence as in
SOCK_DGRAM over UDP) are inadequate. This evolution is constrained by the
presence of middleboxes which interfere with connectivity or packet
invariability in the presence of new transport protocols or transport protocol
extensions.

The core issue is one of layer violation. The boundary between the network and
transport layers was originally defined to be the boundary between information
used (and potentially modified) hop-by-hop, and that only used end-to-end. The
widespread deployment of network address and port translation (NAPT) in the
Internet has eroded this boundary. The first four bytes after the IP header or
header chain -- the source and destination ports -- are now the de facto
boundary between the layers. This erosion has continued into the transport and
application layer headers and down into content, as the capabilities of
deployed middleboxes have increased over time. Evolution above the network
layer is only possible if this layer boundary is reinforced. Asking on-path
devices nicely not to muck about in the transport layer and below -- stating
in an RFC that devices on path MUST NOT use or modify some header field -- has
not proven to be of much use here, so we need a new approach.

This boundary can be reinforced by encapsulating new transport protocols in
UDP and encrypting everything above the UDP header. Indeed, this will be a
side effect of the increased deployment of cryptography in Internet protocols
to protect against pervasive monitoring in any event. However, this brings
with it other problems. First, middleboxes which maintain state must use
timers to expire that state for UDP flows, since there is no exposure of flow
lifetime and bidirectional establishment as with TCP's SYN, ACK, FIN, and RST
flags. These timers are often set fast enough to require a relatively high
rate of heartbeat traffic to maintain this state. A limited facility to expose
basic semantics of the underlying transport protocol would allow these devices
to keep state as they do with TCP, with no worse characteristics with respect
to state management than those of TCP.

This is a specific case of a more general issue: some of the inspection of
traffic done by middleboxes above the network-transport boundary is
operationally useful. However, the use of transport layer and higher layer
headers is an implicit feature of the architecture: middleboxes are exploiting
the fact that data is often transmitted in cleartext. There is no explicit
cooperation here: the endpoints have no control over the information exposed
to the path, and the middleboxes no information about the intentions of the
endpoint application other than that inferred from the inspected traffic. We
propose a change to the architecture to remedy this condition.

# Explicit Cooperation as Architectural Principle

The principle behind this change is one of explicit cooperation.

The present Internet architecture is rife with implicit cooperation between
endpoints and devices on the path between them. For example, network address
translators (NATs) rewrite addresses and ports in order to increase the size
of the Internet at the expense of universal reachability, but this translation
is not explicitly invoked by either endpoint. As another example, traffic
classification is often required for network management purposes, and often
uses deep packet inspection to determine the traffic class of a particular
packet or flow. This classification is done without endpoint consent or
knowledge, and without any explicit cooperation of the designers and
implementors of the protocols being inspected. Indeed, the entire knowledge
available at the midpoint in this arrangement is based on assumptions about
the protocol implementations in use at the endpoints.

It is this implicit cooperation which has led to the ossification of the
transport layer in the Internet. Implicit cooperation requires devices along
the path to make assumptions about the format of the packets and the nature of
the traffic they are forwarding, which in turn leads to problems using
protocols which don't meet these assumptions. It also forces application and
transport protocol developers to build protocols that operate in this
presumed, least-common-denominator network environment.

We take the position that this situation can be improved by replacing implicit
cooperation with explicit cooperation. We first explore the properties of an
ideal architecture for explicit cooperation, then consider the constraints
imposed by the present configuration of the Internet which would make
transition to this ideal architecture infeasible. From this we derive a set of
architectural principles and protocol design requirements which will support
an incrementally deployable approach to explicit cooperation between
applications on endpoints and middleboxes in the Internet.

## What does good look like?

We can take some guidance for the future from the original Internet architecture.

The original Internet architecture defined the split between TCP and IP by
defining IP to contain those functions the gateways need to handle (and
possibly de- and re-encapsulate, including fragmentation), while defining TCP
to contain functions that can be handled by hosts end-to-end {{RFC0791}}.
Gateways were essentially trusted not to meddle in TCP.

As a first principle, a strict division between hop-to-hop and end-to-end
functions is desirable to restore and maintain end-to-end service in the
Internet.

In the original architecture, there was no provision for "in-network
functionality" beyond forwarding, fragmentation, and basic diagnostics. This
was as much a function of adherence to the end-to-end-principle {{Saltzer84}}
as a desire to limit computational complexity and state requirements on the
gateways.

In-network functions in the Internet Protocol as presently defined are
explicit. Forwarding is inherently explicit: placing an address in the
destination address field, the endpoint (and by extension, the application)
indicates that a packet should be sent to a given address. The contract for
fragmentation was implicit in IPv4, but in-network fragmentation was removed
in IPv6 {{RFC2460}}.

We note that layer boundaries can be enforced using sufficiently strong
cryptography.

As a second principle, the presence of in-network functionality along a path
which results in the modification of packet streams received at the other end
of a connection should be explicit.

For optional functionality, the applications at either end of a connection
should have appropriate explicit control over the presence of those functions
on path, even if they are present by default. For those functions which are
necessary, without which there is no end-to-end connectivity (e.g. NATs in
many environments), or which are otherwise non-optional for operational
reasons (e.g. firewalls), the functionality should be explicitly discoverable
by the applications on either end.

This explicitness extends into the transport stack in the endpoint. When
applications can clearly define transport requirements, instead of implicitly
lensing them through known implementations of each socket type, these
transport requirements can be exposed to and/or matched with properties of
devices along the path, where that is useful.

## What keeps us from getting there?

The clear separation of network and transport layer has been steadily eroded
over the past twenty years or so. Network address and port translation (NAPT)
have effectively made the first four bytes of the transport header a de-facto
part of the network layer, and have made it difficult to deploy protocols
where NAPT devices don't know that the ports are safe to touch: anything other
than UDP and TCP. Protocols to support with NAT traversal (e.g. Interactive
Connectivity Establishment {{RFC5245}}) do not address this fundamental
problem.

Mechanisms that could be used to support explicit cooperation between
applications and middleboxes could be supported within the network layer. The
IPv6 Hop-by-Hop Options Header is intended for this purpose, and a new
hop-by-hop option could be defined. However, there are some limitations to
using this header: it is only supported by IPv6, it may itself cause packets
to be dropped, it may not be handled efficiently (or indeed at all) by
currently deployed routers and middleboxes
{{I-D.baker-6man-hbh-header-handling}}, and it requires changes to operating
system stacks at the endpoints to allow applications to access these headers.

One of the effects of the fact that cryptography enforces layer boundaries is
that applications and transports run over HTTPS de facto
{{I-D.blanchet-iab-internetoverport443}}, since HTTPS is the most widely
implemented, accessible, and deployable way for application developers to get
this enforcement.

However, the greatest barriers to explicit cooperation between applications
and devices along the path is the lack of explicit trust among them. While it
is possible to assign trust within the "first hop" administrative domains,
especially when the endpoint and network operator are the same entity,
building and operating an infrastructure for assigning and maintaining these
trust relationships within an Internet context is currently impractical.

Finally, the erosion of the end-to-end principle has not occurred in a vacuum.
There are incentives to deploy in-network functions, and services that are
impaired by them have already worked around these impairments. For example,
the present trend toward service recentralization ("cloud computing") can be
seen in part as the market's response to the end of end-to-end. If every
application-layer transaction is mediated by services owned by the
application's operator, two-end NAT traversal is no longer important. This new
architecture for services has additional implications for the types of
interactions supported, and for the types of business models encouraged, which
may in turn make some of the concerns about limited deployability of new
transport protocols moot.

## What can we do?

First we turn to the problem of re-separation of the network layer from the
transport layer. NAPT, as noted, has effectively made the ports part of the
network layer, and this change is not easy to undo, so we can make this
explicit. In many NAPT environments only UDP and TCP traffic will be
forwarded, and a packet with a TCP header may be assumed by middleboxes to
have TCP semantics; therefore, the solution space is most probably constrained
to putting the "new" separation between the network and transport layers
within a UDP encapsulation.

Since the relative delay in getting new transport protocols into widely
deployed kernel implementations has historically been another impediment to
deploying these new protocols in the Internet, a UDP encapsulation based
approach has a further implication for incremental deployability: it is
possible to implement UDP-based encapsulations in userspace. While userspace
implementations may lack some of the interfaces to lower layers available to
kernelspace implementations necessary to build high-performance transport
implementations, the ability to deploy widely and quickly may help break the
chicken-and-egg problem of getting a transport protocol adopted into
kernelspace.

Some issues inherent in this approach are addressed in {{encapsulation-considerations}}.

Next, we turn to the problem of explicit cooperation. Here we presume that the
encapsulation we use to reinforce the boundary can be extended to do
path-endpoint and endpoint-path signaling below the "new", encrypted transport
layer boundary. We explore the issues in building such an explicit signaling
approach without having to manage identity and trust relationships among every
endpoint and middlebox in the Internet in
{{trust-in-endpoint-path-signaling}}.

# Encapsulation considerations

As noted above, if we assume that only TCP and UDP traffic can be easily transported over the current Internet, we are left with considering UDP-based encapsulations. Encapsulation based approaches have issues we must consider.

## MTU issues

A good deal of experience with tunnels has shown that the per-stream overhead of a given encapsulation is generally less important than its impact on MTU. For instance, the SPUD prototype as presently defined {{I-D.hildebrand-spud-prototype}} needs at least 20 additional bytes in the header per packet: 2 each for source and destination UDP port, 2 for UDP length, 2 for UDP checksum, 8 to identify tubes, 1 for control bits for SPUD itself, and 3 for the smallest possible CBOR map containing a single opaque higher layer datagram. For 1500-byte Ethernet frames, the marginal cost of SPUD before is therefore 1.33% in byte terms, but it does imply that 1450 byte application datagrams will no longer fit in a single SPUD-over-UDP-over-IPv4-over Ethernet packet.

This fact has two implications for encapsulation design: First, maximum payload size per packet should be communicated up to the higher layer, as an explicit feature of the transport layer's interface. Second, link-layer MTU is a fundamental property of each link along a path, so any signaling protocol allowing path elements to communicate with the endpoint should treat MTU as one of the most important properties along the path to explicitly signal. This information must be made available to the transport protocol and application on the endpoint, as well.

## Return routability and feedback

The ease of forging source addresses in UDP together with the only limited
deployment of network egress filtering {{RFC2827}} means that UDP traffic
presently lacks a return routability guarantee. This has led in part to the
present situation wherein UDP traffic may be blocked by firewalls when not
explicitly needed by an organization as part of its Internet connectivity.

Return routability is therefore a minimal property of any transport that can
be responsibly deployed at scale in the Internet. Encapsulation approaches
must provide a return routability guarantee and defense against state
exhaustion attacks.

# Trust in endpoint-path signaling

In a perfect world, the trust relationships among endpoints and elements on
path would be precisely and explicitly defined: an endpoint would explicitly
delegate some processing to a network element on its behalf, network elements
would be able to trust any command from any endpoint, and the integrity and
authenticity of signaling in both directions would be cryptographically
protected.

However, both the economic reality that the users at the endpoints and the
operators of the network may not always have aligned interests, as well as the
difficulty of universal key exchange and trust distribution among widely
heterogeneous devices across administrative domain boundaries, require us to
take a different approach toward building deployable, useful metadata
signaling.

We observe that imperative signaling approaches in the past have often failed
in that they give each actor incentives to lie. Markings to ask the network to
explicitly treat some packets as more important than others will see their
value inflate away -- i.e., most packets from most sources will be so marked --
unless some mechanism is built to police those markings. Reservation protocols
suffer from the same problem: for example, if an endpoint really needs 1Mbps,
but there is no penalty for reserving 1.5Mbps "just in case", a conservative
strategy on the part of the endpoint leads to over-reservation.

## Declarative marking

An alternate approach is to treat these markings as declarative and advisory,
and to treat all markings on packets and flows as relative to other markings
on packets and flows from the same sender. In this case, where neither
endpoints nor path elements can reliably predict the actions other elements in
the network will take with respect to the marking, and where no endpoint can
ask for special treatment at the expense of other endpoints, the incentives to
marking inflation are greatly diminished.

For example, consider the case of an endpoint declaring that a particular
traffic flow is latency- rather than loss- sensitive. In other words, it is
more willing to accept higher packet loss rates than longer delay. These
tolerances may be expressed either with a single boolean signal
("latency-sensitive" as opposed to the default loss-sensitive assumption
implicitly made in the current Internet) or with some richer vocabulary (i.e.
"if this packet doesn't make it in 200ms it's useless"). First, note that in
any case, the preference the endpoint is declaring is a tradeoff, as opposed
to "better" service at the expense of some other traffic. This reduces the
incentive to "mark inflation".

As a second example, consider a sender that knows it has a 200kbps audio
stream, and that it has two minutes of audio to send. It can declare its
intention to send at the known rate and duration, which the path can use for
traffic engineering purposes. However, there in a declarative environment,
there is no expectation on the part of the endpoint that this bandwidth will
be reserved, so there is no incentive to ask for more bandwidth for longer.

## Verifiable marking

In addition, markings and declarations should be defined in such a way that
they are verifiable, and verification built end to endpoints and middleboxes
wherever practical. Going back to the previous example, the average data rate
for the given flow is trivially verifiable at any endpoint. A firewall which
uses this data for traffic classification and differential quality of service
may spot-check the data rate for such flows, and penalize those flows and
sources which are clearly mismarking their traffic.

It is probably not possible, especially in an environment with ubiquitous
opportunistic encryption {{RFC7435}}, to define a useful marking vocabulary
such that every marking will be so easily verifiable. But we note that the
more verifiable a marking is, the more likely it will be useful in a
variably-trusted environment.

# Privacy considerations in endpoint-path signaling

Attaching new information to flows or subflows will require a new identifier
to associate sets of packets with each other. Here there is a tradeoff between
flexibility and privacy preservation. A large identifier space with globally
unique identifiers for packet groups would support multicast, multipath, and
mobility applications, but would also build a new tracking identifier into the
encapsulation, nullifying one of the advantages of encrypting everything above
the networking layer. Recognizing that much of the network recognizes a
five-tuple (source and destination address and port along with protocol
identifier) as a flow ID, scoping the packet group ID to the five-tuple will
reduce or eliminate its additional usefuless for tracking, as well as fixing
problems with ID collision in NAT environments.

Declarations must be designed according to the principle of least access:
information exposed by the signaling mechanism must only expose that
information actually needed to implement a certain service in the network, and
no more. For the example above, the traffic bit rate and duration are
signaled, but there is no explicit declaration of the content type, encoding,
or identity.

# Engineering tradeoffs

## Tradeoffs in integrity protection

In order to protect the integrity of information exposed to the path against
trivial forging by malicious devices along the path, it is necessary to be
able to authenticate the originator of that information. For endpoint
authentication, this can borrow the authentication mechanism used by the
encrypted transport layer.  However, in the Internet, it is not possible for
the endpoint to authenticate every middlebox that might see packets it sends
and receives in the general case. Information produced by middleboxes may
therefore necessarily enjoy less integrity protection than that produced by
endpoints.

## Tradeoffs in signaling patterns

In the simplest case of information exposed by endpoints, in-band signaling is
possible. Here, the signaling channel happens on the same 5-tuple as the data
carried by the transport layer. This arrangement does not require
foreknowledge of the identity and addresses of devices along the path by
endpoints and vice versa -- endpoint declarations are seen by each device
along the path as a side effect of packet forwarding -- but is harder to
realize for path-to-endpoint signaling.

In-band signaling can be either piggybacked or interleaved. Piggybacked
signaling uses some number of bits in each packet generated by the overlying
transport to achieve signaling. It requires either reducing the MTU, which is
already necessary in encapsulation-based approaches to layer reinforcement, or
opportunistically using bits between the suspected network-layer MTU and the
bits actually used by the transport. This is even more complicated in the case
of middleboxes that wish to add information to piggybacked-signaling packets,
and may require the endpoints to introduce "scratch space" in the packets for
potential middlebox signaling use, further increasing complexity and overhead.

Interleaved signaling uses packets on the same 5-tuple to achieve signaling,
but packets are either signaling or content. This reduces complexity and
sidesteps MTU problems, but is only applicable to signaling that can be done
per packet group as opposed to per packet.

Out-of-band signaling uses direct connections between endpoints and
middleboxes, separate from the transport. These connections could be set up
using in-band signaling. A key disadvantage here is that out-of-band signaling
packets may not take the same path as packets with content. Signaling of
path-to-endpoint information, in the case that a middlebox wants to signal
something to the sender of the packet, raises the added problem of either (1)
requiring the middlebox to send the information to the receiver for later
reflection back to the sender, which has the disadvantage of complexity, or
(2) requiring out-of-band direct signaling back to the sender, which in turn
either requires the middlebox to spoof the source address and port of the
receiver to ensure equivalent NAT treatment, or some other NAT-traversal
approach.

# Incentives for deployment

Information exposed must provide benefits to both the endpoint and
middlebox by e.g. providing additional information to use a certain network
service that enables a better service experience to the application as well as
helps the network to make operational decision that simplifies network
management.

# IANA Considerations

This document has no considerations for IANA.

# Security Considerations

This revision of this document presents no security considerations. A more rigorous definition of the limits of declarative and verifiable marking would need to be evaluated against a specified threat model, but we leave this to future work.

# Acknowledgments

Many thanks to the attendees of the IAB Workshop on Stack Evolution in a Middlebox Internet (SEMI) in Zurich, 26-27 January 2015; most of the thoughts in this document follow directly from discussions at SEMI. This work is partially supported by the European Commission under Grant Agrement FP7-318627 mPlane; support does not imply endorsement by the Commission of the content of this work.
