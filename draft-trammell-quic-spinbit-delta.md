---
title: The Latency Spin Bit for the QUIC Transport Protocol
abbrev: QUIC Spin Bit
docname: draft-trammell-quic-spinbit-delta-latest
date:
category: exp

ipr: trust200902
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: B. Trammell
    role: editor
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
  -
    ins: P. De Vaere
    name: Piet De Vaere
    org: ETH Zurich
    email: piet@devae.re
normative:

informative:
  CACM-TCP:
    title: Passively Measuring TCP Round-Trip Times (in Communications of the ACM)
    author:
      -
        ins: S. Strowes
    date: 2013-10
  TMA-QOF:
    title: Inline Data Integrity Signals for Passive Measurement (in Proc. TMA 2014)
    author:
      -
        ins: B. Trammell
      -
        ins: D. Gugelmann
      -
        ins: N. Brownlee
    date: 2014-04


--- abstract

This document specifies the addition of a latency spin bit to the
QUIC transport protocol and describes how to use it to measure end-to-end
latency. It is intended as a delta to the current specification of QUIC. An
appendix describes the Valid Edge Counter mechanism, which can be used
to determine the validity of an RTT sample in case of loss and reodering 
without using the QUIC packet number.

--- middle

# Introduction

The QUIC transport protocol {{?QUIC-TRANS=I-D.ietf-quic-transport}} uses
Transport Layer Security (TLS) {{?TLS=I-D.ietf-tls-tls13}} to encrypt most of 
its protocol internals. In contrast to TCP where the sequence and 
acknowledgement numbers and timestamps (if the respective option is in use) 
can be seen by on-path observers and used to estimate end-to-end latency, 
QUIC's wire image (see {{?WIRE-IMAGE=I-D.trammell-wire-image}})
exposes currently not expose any information that can be used for passive 
latency measurement techniques that rely on
this information (e.g. {{CACM-TCP}}, {{TMA-QOF}}).

This document adds an explicit signal
for passive latency measurability to the QUIC short header: a "spin bit".
Passive observation of the spin bit provides one RTT sample per RTT to passive
observers of QUIC traffic. This document describes the mechanism, how it can
be added to QUIC, and how it can be used by passive measurement facilities to
generate RTT samples.

{{vec}} specifies an additional Valid Edge Counter, which can be used to
improve the fidelity of spin bit measurements under less than ideal network
and/or traffic conditions.

## About This Document

This document is maintained in the GitHub repository
https://github.com/britram/draft-trammell-quic-spin, and the editor's copy is
available online at https://britram.github.io/draft-trammell-quic-spin.
Current open issues on the document can be seen at
https://github.com/britram/draft-trammell-quic-spin/issues. Comments and
suggestions on this document can be made by filing an issue there, or by
contacting the editor.

# The Spin Bit Mechanism

The latency spin bit enables latency monitoring from observation points on the
network path. Since it is possible to measure handshake RTT without a spin bit, it is
sufficient to include the spin bit in the short packet header. The spin bit
therefore appears only after version negotiation and connection establishment
are completed. 

## Proposed Short Header Format Including Spin Bit

As of version -10 of {{QUIC-TRANS}}, this proposal specifies
using the fifth most significant bit (0x08) of the first octet in the short
header for the spin bit.

~~~~~

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|K|1|1|0|S|T T|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Destination Connection ID (0..144)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~~
{: #fig-short-header title="Short Header Format including proposed Spin Bit"}

S: The Spin bit is set 0 or 1 depending on the stored spin value that is updated on packet 
reception as explained in sec {[spinbit]}.

## Seeting the Spin Bit  {#spinbit}

Each endpoint, client and server, maintains a spin value, 0 or
1, for each QUIC connection, and sets the spin bit in the short header to the 
currently stored value when a packet with a short header is sent out. 
The spin value is initialized to 0 on both side, at the client as well as the server
at connection start. Each endpoint also remebers the
highest packet number seen from its peer on the connection. The spin value is then
determined at each endpoint as follows:

* When it receives a packet from
the client, if that packet has a short header and if it increments the
highest packet number seen by the server from the client, it sets the spin
value to the value obsovered in the spin bit in the received packet.

* When it receives a packet from
the server, if the packet has a short header and if it increments the
highest packet number seen by the client from the server, it sets the spin
value to the opposite of the spin bit in the received packet.

This procedure will cause the spin bit to change value in each direction once
per round trip. Observation points can estimate the network latency by
observing these changes in the latency spin bit, as described in {{usage}}.
See {{?SPIN-BIT=I-D.trammell-quic-spin-bit}} for further illustration of this mechanism in action.

# Using the Spin Bit for Passive RTT Measurement {#usage}

When a QUIC flow is continuously sending data, the latency spin bit in each direction changes value once
per round-trip time (RTT). An on-path observer can observe the time difference
between edges (changes from 1 to 0 or 0 to 1) in the spin bit signal in a single direction to measure one
sample of end-to-end RTT. 

Further an observer should also remember the largest observed packet number 
and reject edges that do not have a packet number that is larger than the last
largest observed packet number.  This will detect spurious edges that can be 
caused by reordering across a spin bit flip in the stream and would therefore lead 
to too low RTT estimates, if not ignored.

Further, the packet number can be used to filter out invalid samples that 
indicate a too large RTT estimates due to loss of the actual edge in a burst of lost 
packets. If the spin bit edge occurs after a long packet number gap, it should be rejected.

Note that this measurement, as with passive RTT
measurement for TCP, includes any transport protocol delay (e.g., delayed
sending of acknowledgements) and/or application layer delay (e.g., waiting for
a request to complete). It therefore provides devices on path a good
instantaneous estimate of the RTT as experienced by the application. A simple
linear smoothing or moving minimum filter can be applied to the stream of RTT
information to get a more stable estimate.

However, application-limited and flow-control-limited senders can have application and
transport layer delay, respectively, that are much greater than network RTT.
When the sender is application-limited and e.g. only sends small amount of 
periodic application traffic, where that period is
longer than the RTT, measuring the spin bit provides information about the
application period, not the network RTT. Simple heuristics based on the observed data
rate per flow or changes in the RTT series can be used to reject bad RTT
samples due to application or flow control limitation.

An on-path observer that can see traffic in both directions (from client to
server and from server to client) can also use the spin bit to measure
"upstream" and "downstream" component RTT; i.e, the component of the
end-to-end RTT attributable to the paths between the observer and the server
and the observer and the client, respectively. It does this by measuring the
delay between a spin edge observed in the upstream direction and that observed
in the downstream direction, and vice versa.


# Scope of the Experiment

This document specifies an experimental delta to the QUIC transport protocol.
Specifically, this experimentation is intended to determine:

- the impact of the addition of the latency spin bit on implementation and
  specification complexity; and
- the accuracy and value of the information derived from spin bit measurement
  on live network traffic.

The information generated by this experiment will be used by the QUIC working
group as input to a decision about the standardization of the latency spin
bit.

This document describes a one-bit latency spin signal. A three-bit latency
spin signal, which provides reordering, loss, and edge delay resistance even
without cleartext packet numbers in the QUIC header, is described in {{vec}};
experimentation with this approach is also encouraged.

# IANA Considerations

This document has no actions for IANA.

# Security and Privacy Considerations

The spin bit is intended to expose end-to-end RTT to observers along the path,
so the privacy considerations for the latency spin bit are essentially the
same as those for passive RTT measurement in general. However, it has been
shown that these kind of RTT estimates do not provide a sufficiently high 
enough accurancy for geo-locating, therefore the privacy risk of exposing
these information is considered low.

# Acknowledgments

This document is derived from {{?I-D.trammell-quic-spin}}, which was the work
of the following authors in addition to the editor:

- Roni Even, Huawei
- Giuseppe Fioccola, Telecom Italia
- Thomas Fossati, Nokia
- Marcus Ihlar, Ericsson
- Al Morton, AT&T Labs
- Emile Stephan, Orange

The QUIC Spin Bit was originally specified by Christian Huitema, and the Valid
Edge Counter mechanism in {{vec}} was first specified by Piet De Vaere.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.

--- back

# The Valid Edge Counter {#vec}

This mechanism is indented to provide addition information about the
validity of the passively observed spin edges in case the packet number 
can not be used for this purpose anymore.

A one-bit spin signal is resistent to reordering during signal generation,
since the spin value is only updated at each endpoint on a packet that
advances the packet counter. However, wthout the packet number,
an observer cannot easily detect reordering or infer a lost edges.
Without the packet number, an passive observer would need to 
use heuristics to single reject too low or too high RTT samples.
However, such a filter could conceal exactly those effects that 
an observer what to measure for troubleshooting. Further, such
heuristics can not determine whether edges get constantly delayed at the sender,
e.g. due to application limits, and therefore do not provide a 
valid estimate in the first place.

The Valid Edge Counter (VEC) addresses these issues with two additional
bits added to each packet, encoding values from 0 to 3, indicating that
an edge was considered to be valid when send out by the sender, and 
providing a possibility to detect invalid edges due to reodering and edge loss.

## Proposed Short Header Format Including Spin Bit and VEC

As of version -10 of {{QUIC-TRANS}}, this proposal specifies
using the fifth most significant bit (0x08) of the first octet in the short
header for the spin bit.

~~~~~

0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|K|1|1|0|S|VEC|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Destination Connection ID (0..144)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~~
{: #fig-short-header title="Short Header Format including proposed Spin Bit"}

S: The Spin bit is set 0 or 1 depending on the stored spin value that is updated on packet 
reception as explained in sec {[spinbit]}.

VEC: The Valid Edge Counter is rotated 2 and 3 on every valid spin bit edge and set to 1 on
invalid edges as further explained in {{vec-bits}}. If the spin bit is not an edge the VEC is set to 0.

## Setting the Valid Edge Counter (VEC) {#vec-bits}

The VEC is set by each endpoint as follows; unlike the spin bit, note that there is no difference
between client and server handling of the VEC:

- By default, the VEC is set to 0.
- If a packet contains an edge (transition 0->1 or 1->0) in the spin signal,
  and that edge is delayed (sent more than a configured delay since the edge
  was received, defaulting to 1ms), the VEC is set to 1.
- If a packet contains an edge in the spin signal, and that edge is not
  delayed, the VEC is set to the value of the VEC that accompanied the last
  incoming spin bit transition plus one, holding at 3. That means, if the edge 
  transition is trigger by a packet that has a VEC of 0, the VEC in the packet
  that will be sent out it set to 1, indicating an invalid edge as actually received 
  edge that shoud have trigger the transition was probably lost. If the transition is
  trigger by a packet with a VEC of 1, that edge was invalid that the new edge is valid 
 again, indicated by a VEC of 2. A received VEC of 2, will result in a VEC of 3, and 
 when a VEC of 3 is received, the new VEC will stay at 3.
 
 This mechanism allows observers to recognize spurious edges due to reordering
 and delayed edges due to loss, since these packets will have been sent with
 VEC 0: they were not edges when they were sent. In addition, it allows senders
 to signal that a valid edge was delayed because the sender was
 application-limited: these edges are sent with the VEC set to 1 by the sender,
 prompting the VEC to count back up over the next RTT.
 
 ## Use of the VEC by a passive observer

The VEC can be used by observers to determine whether an edge in the spin bit
signal is valid or not, as follows:

- A packet containing an apparent edge in the spin signal with a VEC of
  0 is not a valid edge but may be caused by reordering or loss and therefore 
  should be ignored.
- A packet containing an apparent edge in the spin signal with a VEC of 1 can
  be used as a left edge (i.e., to start measuring an RTT sample), but not as
  a right edge (i.e., to take an RTT sample since the last edge).
- A packet containing an apparent edge in the spin signal with a VEC of 2 can
  be used as a left edge, but not as a right edge. If the observation point is
  symmetric (i.e, it can see both upstream and downstream packets in the
  flow), the packet can also be used to take a component RTT sample on the
  segment of the path between the observation point and the direction in which
  the previous VEC 1 edge was seen.
- A packet containing an apparent edge in the spin signal with a VEC of 3 can
  be used as a left edge or right edge, and can be used to compute component
  RTT in either direction.



