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
  Saltzer84:
    title: End-to-End Arguments in System Design (ACM Trans. Comp. Sys.)
    author:
      -
        ins: J. H. Saltzer
      -
        ins: D. P. Reed
      -
        ins: D. D. Clark
    date: 1984  RFC0792:
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

The IAB Stack Evolution in a Middlebox Internet (SEMI) workshop in Zurich in January 2015 and the follow-up Substrate Protocol for User Datagrams (SPUD) BoF session at the IETF 92 meeting in Dallas in March 2015 identified the potential need for a UDP-based encapsulation to allow explicit cooperation with middleboxes while using new, encrypted transport protocols. This document proposes a set of architectural considerations for such approaches.

--- middle

# Introduction and Motivation

[EDITOR'S NOTE: this whole document is a horrendous mishmash of spudreq and newtea at the moment. needs edit badly.]


The current work of the IAB IP Stack Evolution Program is to support the evolution of the Internet's transport layer and its interfaces to other layers in the Internet Protocol stack. The need for this work is driven by two trends. First is the development and increased deployment of cryptography in Internet protocols to protect against pervasive monitoring {{RFC7258}}, which will break many middleboxes used in the operation and management of Internet-connected networks and which assume access to plaintext content. An additional encapsulation layer to allow selective, explicit metadata exchange between the endpoints and devices on path to replace ad-hoc packet inspection is one approach to retain network manageability in an encrypted Internet.

Second is the increased deployment of new applications (e.g. interactive media as in RTCWEB {{I-D.ietf-rtcweb-overview}}) for which the abstractions provided by today's transport APIs (i.e., either a single reliable stream as in SOCK_STREAM over TCP, or an unreliable, unordered packet sequence as in SOCK_DGRAM over UDP) are inadequate. This evolution is constrained by the presence of middleboxes which interfere with connectivity or packet invariability in the presence of new transport protocols or transport protocol extensions.

Parts of this problem are presently being addressed in various ways by the IETF. The Transport Services (TAPS) Working Group is defining a new abstract interface for specifying transport requirements to the transport layer, with a vocabulary based upon existing transport protocol service features. This will allow future transport layers (implemented in userspace libraries, in operating system kernels, or some combination of the two) to select a wire protocol based upon these requirements and the properties of the path between the endpoints, including the impairments of middleboxes along that path.

The Substrate Protocol for User Datagrams (SPUD) Birds of a Feather (BoF) session at IETF 92 in Dallas in March 2015 discussed use cases and a prototype protocol {{I-D.hildebrand-spud-prototype}} for encapsulating opaque content in UDP, with a facility for signaling limited transport semantics and binding metadata to packets and flows in a flexible way. This encapsulation is designed to provide explicit cooperation between endpoints and middleboxes where this makes sense, while allowing new transport protocol development to happen both in the kernel -- to which it has largely been restricted due to the history of the development of TCP/IP -- as well as in userspace. The outcome of the BoF session was to continue the discussion about the architecture, transport semantics and metadata vocabulary, and experimental implementation of this approach on the mailing list established for the BoF (spud@ietf.org)

SPUD is not the only protocol-level work to address explicit communication between endpoints and devices along the path: work in the Transport Layer Security (TLS) working group {{I-D.huitema-tls-dtls-as-subtransport}} discusses the possibility and provides a gap analysis for running a "minimal common subtransport" exposing common transport semantics as in SPUD directly over the Datagram Transport Layer Security (DTLS) protocol {{RFC6347}}.

These efforts aim at building flexible mechanisms to solve the problem of expanding the interface between the transport layer and the applications above it as well as the problem of making explicit the contract between the transport layer and devices on path which should, in an end-to-end Internet, limit themselves to lower-layer interactions, but practically speaking have not done so for the past two decades.

This document aims to provide an architectural basis for these efforts, enumerating a set of architectural assumptions for transport evolution based upon new encapsulations, and discussing limitations on the vocabulary used in each of these new interfaces necessary to achieve deployment.

The boundary between the network and transport layers was originally defined to be the boundary between information used (and potentially modified) hop-by-hop, and that only used end-to-end. The widespread deployment of network address and port translation (NAPT) in the Internet has eroded this boundary. The first four bytes after the IP header or header chain -- the source and destination ports -- are now the de facto boundary between the layers. This erosion has continued into the transport and application layer headers and down into content, as the capabilities of deployed middleboxes have increased over time. Evolution above the network layer is only possible if this layer boundary is reinforced. Asking on-path devices nicely not to muck about in the transport layer and below -- stating in an RFC that devices on path MUST NOT use or modify some header field -- has not proven to be of much use here, so we need a new approach.

This boundary can be reinforced by encapsulating new transport protocols in UDP and encrypting everything above the port numbers. Indeed, this will be a side effect of the increased deployment of cryptography in Internet protocols to protect against pervasive monitoring {{RFC7258}} in any event. However, this brings with it other problems. First, middleboxes which maintain state must use timers to expire that state for UDP flows, since there is no exposure of flow lifetime and bidirectional establishment as with TCP's SYN, ACK, FIN, and RST flags. These timers are often set fast enough to require a relatively high rate of heartbeat traffic to maintain this state. A limited facility to expose basic semantics of the underlying transport protocol would allow these devices to keep state as they do with TCP, with no worse characteristics with respect to state management than those of TCP.

This is a specific case of a more general issue: some of the inspection of traffic done by middleboxes above the network-transport boundary is operationally useful. However, the use of transport layer and higher layer headers is an implicit feature of the architecture: middleboxes are exploiting the fact are transmitted in cleartext. There is no explicit cooperation here: the endpoints have no control over the information exposed to the path, and the middleboxes no information about the intentions of the endpoint application other than that inferred from the inspected traffic. We propose a change to the architecture to remedy this condition.



# Terminology

This document borrows terminology from {{I-D.ietf-taps-transports}}, specifically Transport Service, Transport Service Feature, Transport Protocol, and Transport Protocol Component, for discussing the composition of transport services.

[EDITOR'S NOTE: define Application Endpoint, Endhost, and Routable Endpoint, as well as Midpoint, Middlebox, etc., using existing terminology where applicable. A defined terminology here will help avoid imprecision in this conversation.]

# An Architecture for Explicit Path-Endpoint Cooperation

The present Internet architecture is rife with implicit cooperation between endpoints and devices on the path between them. For example, network address translators (NATs) rewrite addresses and ports in order to increase the size of the Internet at the expense of universal reachability, but this translation is not explicitly invoked by either endpoint. Traffic classification is often required for network management purposes, and often uses deep packet inspection to determine the traffic class of a particular packet or flow.

It is this implicit cooperation which has led to the ossification of the transport layer in the Internet. Implicit cooperation requires devices along the path to make assumptions about the format of the packets and the nature of the traffic they are forwarding, which in turn leads to problems using protocols which don't meet these assumptions. It also forces application and transport protocol developers to build protocols that operate in this presumed, least-common-denominator network environment.

We take the position that this situation can be improved by replacing implicit cooperation with explicit cooperation. We first explore the properties of an ideal architecture for explicit cooperation, then consider the constraints imposed by the present configuration of the Internet which would make transition to this ideal architecture infeasible. From this we derive a set of architectural principles and protocol design requirements which will support an incrementally deployable approach to explicit cooperation between applications on endpoints and middleboxes in the Internet.

## Principles: What does good look like?

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

We note that layer boundaries can be enforced using sufficiently strong cryptography.

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

[EDITOR'S NOTE: this is perhaps a bit further than we want to actually go, but this would seem to be the logical conclusion of "make path interaction explicit"]

## Impairments: What keeps us from getting there?

The clear separation of network and transport layer has been steadily eroded over the past twenty years or so. Network address and port translation (NAPT) have effectively made the first four bytes of the transport header a de-facto part of the network layer, and have made it difficult to deploy protocols where NAPT devices don't know that the ports are safe to touch: anything other than UDP and TCP. Protocols to support with NAT traversal (e.g. Interactive Connectivity Establishment {{RFC5245}}) do not address this fundamental problem.

Mechanisms that could be used to support explicit cooperation between applications and middleboxes could be supported within the network layer. The IPv6 Hop-by-Hop Options Header is explicitly intended for this purpose, and a new hop-by-hop option could be defined. However, there are some limitations to using this header: it is only supported by IPv6, it may itself cause packets to be dropped, it may not be handled efficiently (or indeed at all) by currently deployed routers and middleboxes, and it requires changes to operating system stacks at the endpoints to allow applications to access these headers.

One of the effects of the fact that cryptography enforces layer boundaries is
that applications and transports run over HTTPS de facto
{{I-D.blanchet-iab-internetoverport443}}, since HTTPS is the most widely
implemented, accessible, and deployable way for application developers to get
this enforcement.

However, the greatest barriers to explicit cooperation between applications and devices along the path is the lack of explicit trust among them. While it is possible to assign trust within the "first hop" administrative domains,  especially when the endpoint and network operator are the same entity, building and operating an infrastructure for assigning and maintaining these trust relationships within an Internet context is currently impractical.

Finally, the erosion of the end-to-end principle has not occurred in a vacuum.  There are incentives to deploy in-network functions, and services that are impaired by them have already worked around these impairments. For example, the present trend toward service recentralization ("cloud computing") can be seen in part as the market's response to the end of end-to-end. Tf every application-layer transaction is mediated by services owned by the application's operator, two-end NAT traversal is no longer important. This new architecture for services has additional implications for the types of interactions supported, and for the types of business models encouraged, which may in turn make some of the concerns about limited deployability of new transport protocols moot.

## What can we do?

First we turn to the problem of re-separation of the network layer from the transport layer. NAPT, as noted, has effectively made the ports part of the network layer, and this change is not easy to undo, so we can make this explicit. In many NAPT environments only UDP and TCP traffic will be forewarded, and a packet with a TCP header may be assumed by middleboxes to have TCP semantics; therefore, the solution space is constrained to putting the "new" separation between the network and transport layers within a UDP encapsulation. This has a further positive implication for incremental deployability: it is possible to implement UDP-based encapsulations in userspace

# Encapsulation and signaling mechanisms

[EDITOR'S NOTE: frontmatter on encaps]

## Encapsulations are narrow

A good deal of experience with tunnels has shown that the per-stream overhead of a given encapsulation is generally less important than its impact on MTU. For instance, the SPUD prototype as presently defined needs at least 20 additional bytes in the header per packet: 2 each for source and destination UDP port, 2 for UDP length, 2 for UDP checksum, 8 to identify tubes, 1 for control bits for SPUD itself, and 3 for the smallest possible CBOR map containing a single opaque higher layer datagram. For 1500-byte Ethernet frames, the marginal cost of SPUD before is therefore 1.33% in byte terms, but it does imply that 1450 byte application datagrams will no longer fit in a single SPUD-over-UDP-over-IPv4-over Ethernet packet.

This fact has two implications for encapsulation design: First, maximum payload size per packet should be communicated up to the higher layer, as an explicit feature of the transport layer's interface. Second, link-layer MTU is a fundamental property of each link along a path, so any signaling protocol allowing path elements to communicate to the endpoint should treat MTU as one of the most important properties along the path to explicitly signal.

# Implicit trust in endpoint-path signaling

In a perfect world, the trust relationships among endpoints and elements on path would be precisely and explicitly defined: an endpoint would explicitly delegate some processing to a network element on its behalf, network elements would be able to trust any command from any endpoint, and the integrity and authenticity of signaling in both directions would be cryptographically protected.

However, both the economic reality that the users at the endpoints and the operators of the network may not always have aligned interests, as well as the difficulty of universal key exchange and trust distribution among widely heterogeneous devices across administrative domain boundaries, require us to take a different approach toward building deployable, useful metadata signaling.

We observe that imperative signaling approaches in the past have often failed in that they give each actor incentives to lie. Markings to ask the network to explicitly treat some packets as more important than others will see their value inflate away -- i.e., most packets from most sources will be so marked -- unless some mechanism is built to police those markings. Reservation protocols suffer from the same problem: for example, if an endpoint really needs 1Mbps, but there is no penalty for reserving 1.5Mbps "just in case", a conservative strategy on the part of the endpoint leads to over-reservation.

## Declarative marking

An alternate approach is to treat these markings as declarative and advisory, and to treat all markings on packets and flows as relative to other markings on packets and flows from the same sender. In this case, where neither endpoints nor path elements can reliably predict the actions other elements in the network will take with respect to the marking, and where no endpoint can ask for special treatment at the expense of other endpoints, the incentives to marking inflation are greatly diminished.

## Verifiable marking

Second, markings and declarations should be defined in such a way that they are verifiable, and verification built end to endpoints and middleboxes wherever practical. Suppose for example an endpoint declares that it will send constant-bitrate, loss-insensitive traffic at 192kbps. The average data rate for the given flow is trivially verifiable at any endpoint. A firewall which uses this data for traffic classification and differential quality of service may spot-check the data rate for such flows, and penalize those flows and sources which are clearly mismarking their traffic.

It is probably not possible, especially in an environment with ubiquitous opportunistic encryption {{RFC7435}}, to define a useful marking vocabulary such that every marking will be so easily verifiable. However, in an environment in which markings are variably-trusted and verified, trustworthiness can be dynamically assigned by each device, as well as

the trustworthiness of each endpoint and path can be independently assessed by any node involved in a communication, and reputation-tracking approaches can be used to signal how believable a declaration is to transport protocols which use those declarations, as well as to provide an additional incentive to mark honestly.

# Privacy and Incentives

There is a tradeoff between extensibility and privacy preservation.
Scoping the tube ID to a five-tuple will reduce or eliminate its
usefulness for tracking (as well as fixing problems with ID collision in NAT
environments). The information that are exposed by the extensibility mechanism
must further be minimize to only expose the information that are actually needed
to implement a certain network service while information on top of SPUD should be encrypted.
This allows the endpoint to control more fine-grained which information to expose.

Further information exposed must provide benefits to both the endpoint and middlebox by e.g. providing additional information to use a certain network service that enables a better service experience to the application as well as helps the network to make operational decision that simplifies network management.

# IANA Considerations

This document has no considerations for IANA.

# Security Considerations

This revision of this document presents no security considerations. A more rigorous definition of the limits of declarative and verifiable marking would need to be evaluated against a specified threat model, but we leave this to future work.

# Acknowledgments

Many thanks to the attendees of the IAB Workshop on Stack Evolution in a Middlebox Internet (SEMI) in Zurich, 26-27 January 2015; most of the thoughts in this document follow directly from discussions at SEMI. This work is partially supported by the European Commission under Grant Agrement FP7-318627 mPlane; support does not imply endorsement by the Commission of the content of this work.


# History

An outcome of the IAB workshop on Stack Evolution in a Middlebox Internet
(SEMI) {{I-D.iab-semi-report}}, held in Zurich in January 2015, was a
discussion on the creation of a substrate protocol to support the deployment
of new transport protocols in the Internet. Assuming that a way forward for
transport evolution in user space would involve encapsulation in UDP
datagrams, the workshop noted that it may be useful to have a facility built
atop UDP to provide minimal signaling of the semantics of a flow that would
otherwise be available in TCP. At the very least, indications of first and
last packets in a flow may assist firewalls and NATs in policy decision and
state maintenance. Further transport semantics would be used by the protocol
running atop this facility, but would only be visible to the endpoints, as the
transport protocol headers themselves would be encrypted, along with the
payload, to prevent inspection or modification.  This encryption might be
accomplished by using DTLS {{RFC6347}} as a subtransport
{{I-D.huitema-tls-dtls-as-subtransport}} or by other suitable methods. This
facility could also provide minimal application-to-path and
path-to-application signaling, though there was less agreement about what
should or could be signaled here.

The Substrate Protocol for User Datagrams (SPUD) BoF was held at IETF 92 in
Dallas in March 2015 to develop this concept further. It is clear from
discussion before and during the SPUD BoF that any selective exposure of
traffic metadata outside a relatively restricted trust domain must be
advisory, non-negotiated, and declarative rather than imperative. This
conclusion matches experience with previous endpoint-to-middle and
middle-to-endpoint signaling approaches. As with other metadata systems,
exposure of specific elements must be carefully assessed for privacy risks and
the total of exposed elements must be so assessed.  Each exposed parameter
should also be independently verifiable, so that each entity can assign its
own trust to other entities. Basic transport over the substrate must continue
working even if signaling is ignored or stripped, to support incremental
deployment. These restrictions on vocabulary are discussed further in
{{I-D.trammell-stackevo-newtea}}. This discussion includes privacy and trust
concerns as well as the need for strong incentives for middlebox cooperation
based on the information that are exposed.

# Terminology

This document uses the following terms

- Overlying transport: : A transport protocol that uses SPUD for middlebox signaling and traversal.

- Endpoint: : A node that sends or receives a packet. In this document, this term may refer to either the SPUD implementation on this node, the overlying transport implementation on this node, or the applications running over that overlying transport.

- Path: : The set of Internet Protocol nodes and links that a given packet traverses from endpoint to endpoint.

- Middlebox: : A device on the path that makes decisions about forwarding behavior based on other than IP or sub-IP addressing information, and/or that modifies the packet before forwarding.

# Use Cases

The primary use case for endpoint to path signaling, making use of packet
grouping, is the binding of limited related semantics (start-tube and
stop-tube) to a group of packets which are semantically related in terms of
the application or overlying transport. By explicitly signaling start and stop
semantics, a flow allows middleboxes to use those signals for setting up and
tearing down their relevant state (NAT bindings, firewall pinholes), rather
than requiring the middlebox to infer this state from continued traffic. At
best, this would allow the application to refrain from sending heartbeat
traffic, which might result in reduced radio utilization (and thus greater
battery life) on mobile platforms.

SPUD may also provide some facility for SPUD-aware nodes on the path to signal
some property of the path relative to a tube to the endpoints and other
SPUD-aware nodes on the path. The primary use case for path to application
signaling is parallel to the use of ICMP [RFC0792], in that it describes a set
of conditions (including errors) that applies to the datagrams as they
traverse the path. This usage is, however, not a pure replacement for ICMP
but a "5-tuple ICMP" which would traverse NATs in the same way as the traffic
related to it, and be deliverable to the application with appropriate tube
information.

[EDITOR'S NOTE: specific applications we think need this go here? reference draft-kuehlewind-spud-use-cases.]

# Functional Requirements

The following requirements detail the services that SPUD must provide to overlying transports, endpoints, and middleboxes using SPUD.

## Grouping of Packets

Transport semantics and many properties of communication that endpoints may want to expose to middleboxes are bound to flows or groups of flows (five-tuples). SPUD must therefore provide a basic facility for associating packets together (into what we call a "tube" ,for lack of a better term) and associate information to these groups of packets. The simplest mechanisms for association would involve the addition of an identifier to each packet in a tube. The tube ID must be bi-directional to support state establishment and is scoped to the forward and backward five-tuple due to privacy concern. Current thoughts on the tradeoffs on requirements and constraints on this identifier space are given in {{tradeoffs-in-tube-identifiers}}.

## Endpoint to Path Signaling

SPUD must be able to provide information from the end-point(s) to all
SPUD-aware nodes on the path. To be able to potentially communicate with all
SPUD-aware middleboxes on the path SPUD must either be designed as an in-band
signaling protocol, or there must be a pre-existing relationship between the
endpoint and the SPUD-aware middleboxes along the path. Since it is
implausible that an endpoint has these relationships to all SPUD-aware
middleboxes on a certain path in the context of the Internet, SPUD must
provide in-band signaling. SPUD may in addition also offer mechanisms for
out-of-band signaling when it is appropriate to use. See
{{in-band-out-of-band-piggybacked-and-interleaved-signaling}} for more
discussion.


## Path to Endpoint Signaling

SPUD must be able to provide information from a SPUD-aware middlebox to the endpoint.
See {{in-band-out-of-band-piggybacked-and-interleaved-signaling}}
for more discussion on tradeoffs here.

## Extensibility

SPUD must enable multiple new transport semantics and application/path declarations without requiring updates to SPUD implementations in middleboxes.

## Authentication

The basic SPUD protocol must not require any authentication or a priori trust relationship between endpoints and middleboxes to function.  However, SPUD should support the presentation/exchange of authentication information in environments where a trust relationship already exists, or can be easily established, either in-band or out-of-band.

## Integrity

SPUD must provide integrity protection of SPUD-encapsulated packets, though the details of this integrity protection are still open; see {{tradeoffs-in-integrity-protection}}. At the very least, endpoints should be able to:

1. detect simple transmission errors over at least whatever headers SPUD uses its own signaling.

2. detect packet modifications by non-SPUD-aware middleboxes along the path

3. detect the injection of packets into a SPUD flow (defined by 5-tuple) or tube by nodes other than the remote endpoint.

The decision of how to handle integrity check failures other than case (1) may be left up to the overlying transport.

## Privacy

SPUD must allow endpoints to control the amount of information exposed to middleboxes, with the default being the minimum necessary for correct functioning.

# Non-Functional Requirements

The following requirements detail the constraints on how the SPUD facility must meet its functional requirements.

## Middlebox Traversal

SPUD must be able to traverse middleboxes that are not SPUD-aware. Therefore SPUD must be encapsulated in a transport protocol that is known to be accepted on a large fraction of paths in the Internet, or implement some form of probing to determine in advance which transport protocols will be accepted on a certain path.

## Low Overhead in Network Processing

SPUD must be low-overhead, specifically requiring very little effort to recognize that a packet is a SPUD packet and to determine the tube it is associated with.

## Implementability in User-Space

To enable fast deployment SPUD and transports above SPUD must be implementable without requiring kernel replacements or modules on the endpoints, and without having special privilege (root or "jailbreak") on the endpoints. Usually all operating systems will allow a user to open a UDP socket. Therefore SPUD must provide an UDP-based encapsulation, either exclusively or as a mandatory-to-implement feature.

## Incremental Deployability in an Untrusted, Unreliable Environment

SPUD must operate in the present Internet. In order to maximize deployment, it should also be useful as an encapsulation between endpoints even before the deployment of middleboxes that understand it. The information exposed over SPUD must provide incentives for adoption by both endpoints and middleboxes, and must maximize privacy (by minimizing information exposed). Further, SPUD must be robust to packet loss, duplication and reordering by the underlying network service.  SPUD must work in multipath, multicast, and endpoint multi-homing environments.

Incremental deployability likely requires limitations of the vocabulary used in signaling, to ensure that each actor in a nontrusted environment has incentives to participate in the signaling protocol honestly; see {{I-D.trammell-stackevo-newtea}} for more.

## Minimum restriction on the overlying transport

SPUD must impose minimal restrictions on the transport protocols it encapsulates. However, to serve as a substrate, it is necessary to factor out the information that middleboxes commonly rely on and endpoints are commonly willing to expose. This information should be included in SPUD, and might itself impose additional restrictions to the overlying transport. One example is that SPUD is likely to impose at least return routability and the presence of a feedback channel between endpoints, this being necessary for protection against reflection, amplification, and trivial state exhaustion attacks; see {{return-routability-and-feedback}} for more.

## Minimum Header Overhead

To avoid reducing network performance, the information and coding used in SPUD should be designed to use the minimum necessary amount of additional space in encapsulation headers.

## No additional start-up latency

SPUD should not introduce additional start-up latency for overlying transports.

## Reliability and Duplication

As any information provided by SPUD is anyway opportunistic, we assume for now that SPUD does not have to provide reliability and any SPUD mechanism using SPUD information must handle duplication of information. However, this decision also depends on the signal type used by SPUD, as further discussed in {{in-band-out-of-band-piggybacked-and-interleaved-signaling}}, and currently assumes that there are no SPUD information that would need to be split over multiple packets.

# Open questions and discussion

The preceding requirements reflect the present best understanding of the authors of the functional and technical requirements on an encapsulation-based protocol for common middlebox-endpoint cooperation for overlying transports. There remain a few large open questions and points for discussion, detailed in the subsections below.

## Tradeoffs in tube identifiers

Grouping packets into tubes requires some sort of notional tube identifier;
for purposes of this discussion we will assume this identifier to be a simple
vector of N bits. The properties of the tube identifier are subject to
tradeoffs on the requirements for privacy, security, ease of implementation,
and header overhead efficiency.

We first assume that the 5-tuple of source and destination IP address, UDP (or
other transport protocol) port, and IP protocol identifier (17 for UDP) is
used in the Internet as an existing flow identifier, due to the widespread
deployment of network address and port translation. The question then arises
whether tube identifiers should be scoped to 5-tuples (i.e., a tube is
identified by a 6-tuple including the tube identifier) or should be separate,
and presumed to be globally unique.

If globally unique, N must be sufficiently large to reduce to negligibility
the probability of collision among multiple tubes having the same identifier
along the same path during some period of time. An advantage of globally
unique tube identifiers is they allow migration of per-tube state across
multiple five-tuples for mobility support in multipath protocols. However,
globally unique tube identifiers would also introduce new possibilities for
user and node tracking, with a serious negative impact on privacy.

In the case of 5-tuple-scoped identifiers, mobility must be supported
separately by each overlying transport. N must still be sufficiently large,
and the bits in the identifier sufficiently random, that possession of a valid
tube ID implies that a node can observe packets belonging to the tube. This
reduces the chances of success of blind packet injection attacks of packets
with guessed valid tube IDs.

Further using multiple tube identifiers within one 5-tuple also raises some
protocol design questions: Can one packet belong to multiple tubes? Do all packets
in a five-tuple flow have to belong to one tube? Can/Must the backward flow have the
same tube ID or a different one? Especially at connection start-up, depending on
the semantics of the overlying transport protocol, there likely might be only one
packet to start multiple streams/tubes. Can this start message signal multiple
tube IDs at once or do we need an own start message for each tube? Or is it in
this case not possible to have multiple tubes within one five-tuple? These
questions have to be further investigated based on the semantic of existing
transport protocols.  The discussion in {{I-D.ietf-dart-dscp-rtp}} concerning
use of different QoS treatments within a single 5-tuple is related, e.g.,
WebRTC multiplexing of multiple application layer flows onto a single transport
layer (5-tuple) flow.

## Property binding

Related to identifier scope is the scope of properties bound to SPUD packets
by endpoints. SPUD may support both per-tube properties as well as per-packet
properties. Properties signaled per packet reduce state requirements at
middleboxes, but also increase per-packet overhead. It is likely that both
types of property binding are necessary, but the selection of which properties
to bind how must be undertaken carefully.

## Tradeoffs in integrity protection

In order to protect the integrity of information carried by SPUD against
trivial forging by malicious devices along the path, it is necessary to be
able to authenticate the originator of that information. We presume that the
authentication of endpoints is a generally desirable property, and to be
handled by the overlying transport; in this case, SPUD can borrow that
authentication to protect the integrity of endpoint-originated information.

However, in the Internet, it is not in the general case possible for the
endpoint to authenticate every middlebox that might see packets it sends and
receives. In this case information produced by middleboxes may enjoy less
integrity protection than that produced by endpoints.  In addition, endpoint
authentication of middleboxes and vice-versa may be better conducted out-of-
band (treating the middlebox as an endpoint for the authentication protocol)
than in-band (treating the middlebox as a participant in a 3+ party communication).

## Return routability and feedback

We have identified a requirement to support as wide a range of overlying
transports as possible and feasible, in order to maximize SPUD's potential for
improving the evolvability of the transport stack.

The ease of forging source addresses in UDP together with the only limited
deployment of network egress filtering {{RFC2827}} means that UDP traffic
presently lacks a return routability guarantee. This has led in part to the
present situation wherein UDP traffic may be blocked by firewalls when not
explicitly needed by an organization as part of its Internet connectivity. In
addition, to defend against state exhaustion attacks on middleboxes, SPUD may
need to see a first packet in a reverse direction on a tube to consider that
tube acknowledged and valid.

Return routability is therefore a minimal property of any transport that can
be responsibly deployed at scale in the Internet. Therefore SPUD should
enforce bidirectional communication at start-up, whether the overlying
transport is bidirectional or not.  This excludes use of the UDP source
port as an entropy input that does not accept traffic (i.e., for one-way
communication, as is commonly done for unidirectional UDP tunnels, e.g.,
MPLS in UDP {{RFC7510}}).

## In-band, out-of-band, piggybacked, and interleaved signaling

Discussions about SPUD to date have focused on the possibility of in-band
signaling from endpoints to middleboxes and back -- the signaling channel
happens on the same 5-tuple as the data carried by the overlying transport.
This arrangement would have the advantage that it does not require
foreknowledge of the identity and addresses of devices along the path by
endpoints and vice versa, but does add complexity to the signaling protocol.

In-band signaling can be either piggybacked on the overlying transport or
interleaved with it. Piggybacked signaling uses some number of bits in each
packet generated by the overlying transport to achieve signaling. It requires
either reducing the MTU available to the overlying transport (and telling the
overlying transport about this MTU reduction, or relying on the overlying
transport to use PLPMTUD {{RFC4821}}), or opportunistically using bits
between the network-layer MTU and the bits actually used by the transport.

This is even more complicated in the case of middleboxes that wish to add
information to piggybacked-signaling packets, and may require the endpoints to
introduce "scratch space" in the packets for potential middlebox signaling
use, further increasing complexity and overhead. In any case, a SPUD sender
would effectively request SPUD information from a middlebox that respectively
the middlebox would be able to insert the requested information into this place holder.

However, a SPUD using
piggybacked signaling is at the mercy of the overlying transport's
transmission scheduler to actually decide when to send packets. At the same time,
piggybacked signaling has the benefit that SPUD can use the overlying
transport's reliability mechanisms to meet any reliability requirements it has
for its own use (e.g. in key exchange). This would require SPUD to understand the
semantics of the overlying protocol but can reduce overhead. Piggyback signaling is also the only
way to achieve per-packet signaling as in {{property-binding}}.

Interleaved signaling uses SPUD-only packets on the same 5-tuple with the same
tube identifier to achieve signaling. This reduces complexity and sidesteps
MTU problems, but is only applicable to per-tube signaling. It also would
require SPUD to provide its own reliability mechanisms if per-tube signaling
requires reliability, and still needs to interact well with the overlying
transport's transmission scheduler.

Out-of-band signaling uses direct connections between endpoints and
middleboxes, separate from the overlying transport -- connections that are
perhaps negotiated by in-band signaling. A key disadvantage here is that
out-of-band signaling packets may not take the same path as the packets in the
overlying transport and therefore connectivity cannot be guaranteed.

Signaling of path-to-endpoint information, in the case that a middlebox wants
to signal something to the sender of the packet, raises the added problem of
either (1) requiring the middlebox to send the information to the receiver for
later reflection back to the sender, which has the disadvantage of complexity,
or (2) requiring out-of-band direct signaling back to the sender, which in
turn either requires the middlebox to spoof the source address and port of the
receiver to ensure equivalent NAT treatment, or some other NAT-traversal
approach.

The tradeoffs here must be carefully weighed, and the final approach may use a
mix of all these communication patterns where SPUD provides different signaling
patterns for different use case. E.g., a middlebox might need to generate out-of-band signals
for error messages or can provide requested information in-band and feedback over the receiver
if a minimum or maximum value from all SPUD-aware middleboxes on path should be discovered.

## Continuum of trust among endpoints and middleboxes

There are different security considerations for different security contexts.
The end-to-end context is one; anything that only needs to be seen by the path
shouldn't be exposed in SPUD, but rather by the overlying transport. There are
multiple different types of end-to-middle context based on levels of trust
between end and middle -- is the middlebox on the same network as the
endpoint, under control of the same owner? Is there some contract between the
application user and the middlebox operator? It may make sense for SPUD to
support different levels of trust than the default ("untrusted, but presumed
honest due to limitations on the signaling vocabulary") and
fully-authenticated; this needs to be explored further.

## Discovery and capability exposure

There are three open issues in discovery and capability
exposure. First, an endpoint needs to discover if the other
communication endpoint understands SPUD. Second, endpoints need to test
whether SPUD is potentially not usable along a path because of
middleboxes that block SPUD packets or strip the SPUD header. If such
impairments exist in the path, a SPUD sender needs to fall back to some other approach to achieve the goals of the overlying transport. Third, endpoints might want to be able to discover SPUD-aware middleboxes along the path, and to discover which parts of the vocabulary that can be spoken by the endpoints are supported by those middleboxes as well as the other communication endpoint, and vice versa.

In addition, endpoints may need to discover and negotiate which overlying transports are available for a given interaction. SPUD could assist here. However, it is explicitly not a goal of SPUD to expose information about the details of the overlying transport to middleboxes.

## Hard state vs. soft state

The initial thinking on signaling envisions "hard state" in middleboxes that is established when the middlbox observes the start of a SPUD tube and is torn down when the middlebox observes the end (stop) of a SPUD tube.  Such state can be abandoned as a result of network topology changes (e.g., routing update in response to link or node failure).  An alternative is a "soft state" approach that requires periodic refresh of state in middleboxes, but cleanly times out and discards abandoned state.  SPUD has the opportunity to use different timeouts than the defaults that are required for current NAT and firewall pinhole maintenance. Of course, applications will still have to detect non-SPUD middleboxes that use shorter timers.

# Security Considerations

The security-relevant requirements for SPUD deal mainly with endpoint authentication and the integrity of exposed information ({{authentication}}, {{integrity}}, {{privacy}}, and {{tradeoffs-in-integrity-protection}}); protection against attacks ({{tradeoffs-in-tube-identifiers}} and {{return-routability-and-feedback}}); and the trust relationships among endpoints and middleboxes {{continuum-of-trust-among-endpoints-and-middleboxes}}. These will be further addressed in protocol definition work following from these requirements.

# IANA Considerations

This document has no actions for IANA.

# Contributors

In addition to the editors, this document is the work of David Black, Ken Calvert, Ted Hardie, Joe Hildebrand, Jana Iyengar, and Eric Rescorla.

[EDITOR'S NOTE: make this a real contributor's section once we figure out how to make kramdown do that...]
