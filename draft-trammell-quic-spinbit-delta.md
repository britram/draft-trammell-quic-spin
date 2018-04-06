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

This document specifies the experimental addition of a latency spin bit to the
QUIC transport protocol and describes how to use it to measure end-to-end
latency. It is intended as a delta to the current specification of QUIC. An
appendix describes the Valid Edge Counter mechanism, which increases the
utility of the spin bit in less than ideal network and traffic conditions
without using the QUIC packet number.

--- middle

# Introduction

The QUIC transport protocol {{?QUIC-TRANS=I-D.ietf-quic-transport}} is a
UDP-encapsulated protocol integrated with Transport Layer Security (TLS)
{{?TLS=I-D.ietf-tls-tls13}} to encrypt most of its protocol internals, beyond
those handshake packets needed to establish or resume a TLS session, and
information required to decrypt QUIC packets and to route QUIC packets to the
correct machine in a load-balancing situation (the connection ID). In contrast
to TCP, QUIC's wire image (see {{?WIRE-IMAGE=I-D.trammell-wire-image}})
exposes much less information about transport protocol state than TCP's wire
image. Specifically, the fact that sequence and acknowledgement numbers and
timestamps (available in TCP) cannot be seen by on-path observers in QUIC
means that passive TCP loss and latency measurement techniques that rely on
this information (e.g. {{CACM-TCP}}, {{TMA-QOF}}) cannot be easily ported to
work with QUIC.

This document proposes a solution to this problem by adding an explicit signal
for passive latency measurability to the QUIC short header: a "spin bit".
Observation of the spin bit provides one RTT sample per RTT to passive
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

# The Spin Bit Mechanism {#mechanism}

The latency spin bit enables latency monitoring from observation points on the
network path. Each endpoint, client and server, maintains a spin value, 0 or
1, for each QUIC connection, and sets the spin bit on packets it sends for
that connection to the appropriate value (below). It also maintains the
highest packet number seen from its peer on the connection. The value is then
determined at each endpoint as follows:

* The server initializes its spin value to 0. When it receives a packet from
  the client, if that packet has a short header and if it increments the
  highest packet number seen by the server from the client, it sets the spin
  value to the spin bit in the received packet.

* The client initializes its spin value to 0. When it receives a packet from
  the server, if the packet has a short header and if it increments the
  highest packet number seen by the client from the server, it sets the spin
  value to the opposite of the spin bit in the received packet.

This procedure will cause the spin bit to change value in each direction once
per round trip. Observation points can estimate the network latency by
observing these changes in the latency spin bit, as described in {{usage}}.
See {{illustration}} for an illustration of this mechanism in action.

## Proposed Short Header Format Including Spin Bit

Since it is possible to measure handshake RTT without a spin bit, it is
sufficient to include the spin bit in the short packet header. The spin bit
therefore appears only after version negotiation and connection establishment
are completed. As of version -10 of {{QUIC-TRANS}}, this proposal specifies
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

This change limits the number of available short packet types to 4. The short
packet type definitions are unchanged.

# Using the Spin Bit for Passive RTT Measurement {#usage}

When a QUIC flow is sending at full rate (i.e., neither application nor flow
control limited), the latency spin bit in each direction changes value once
per round-trip time (RTT). An on-path observer can observe the time difference
between edges in the spin bit signal in a single direction to measure one
sample of end-to-end RTT. Note that this measurement, as with passive RTT
measurement for TCP, includes any transport protocol delay (e.g., delayed
sending of acknowledgements) and/or application layer delay (e.g., waiting for
a request to complete). It therefore provides devices on path a good
instantaneous estimate of the RTT as experienced by the application. A simple
linear smoothing or moving minimum filter can be applied to the stream of RTT
information to get a more stable estimate.

An on-path observer that can see traffic in both directions (from client to
server and from server to client) can also use the spin bit to measure
"upstream" and "downstream" component RTT; i.e, the component of the
end-to-end RTT attributable to the paths between the observer and the server
and the observer and the client, respectively. It does this by measuring the
delay between a spin edge observed in the upstream direction and that observed
in the downstream direction, and vice versa.

## Limitations

Application-limited and flow-control-limited senders can have application and
transport layer delay, respectively, that are much greater than network RTT.
Therefore, the spin bit provides network latency information only when the
sender is neither application nor flow control limited. When the sender is
application-limited by periodic application traffic, where that period is
longer than the RTT, measuring the spin bit provides information about the
application period, not the RTT. Simple heuristics based on the observed data
rate per flow or changes in the RTT series can be used to reject bad RTT
samples due to application or flow control limitation.

Since the spin bit logic at each endpoint considers only samples on packets
that advance the largest packet number seen, signal generation itself is
resistant to reordering and loss. However, reordering can cause problems at an
observer by causing spurious edge detection and therefore low RTT estimates,
if reordering occurs across a spin bit flip in the stream. Similarly, should
an edge be lost in a burst of lost packets, causing a delayed edge detection
and therefore high RTT estimates. When the packet number is observable,
observation of the packet number can be used to reject lost and reordered
edges.

See also {{vec}} for a proposed enhancement which addresses all of these
limitations, even when the packet number is encrypted.

## Illustration {#illustration}

\[EDITOR'S NOTE: probably cut this]

To illustrate the operation of the spin bit, we consider a simplified model of
a single path between client and server as a queue with slots for five
packets, and assume that both client and server send packets at a constant
rate. If each packet moves one slot in the queue per clock tick, note that
this bidirectional path has a RTT of 10 ticks.

Initially, during connection establishment, no packets with a spin bit are
in flight, as shown in {{illus0}}.

~~~~~
+--------+   -  -  -  -  -   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   -  -  -  -  -   +--------+
~~~~~
{: #illus0 title="Initial state, no spin bit between client and server"}

Either the server, the client, or both can begin sending packets with short
headers after connection establishment, as shown in {{illus1}}; here, no spin
edges are yet in transit.

~~~~~
+--------+   0  0  -  -  -   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   -  -  0  0  0   +--------+
~~~~~
{: #illus1 title="Client and server begin sending packets with spin 0"}

Once the server's first 0-marked packet arrives at the client, the client sets
its spin value to 1, and begins sending packets with the spin bit set, as
shown in {{illus2}}. The spin edge is now in transit toward the server.

~~~~~
+--------+   1  0  0  0  0   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   0  0  0  0  0   +--------+
~~~~~
{: #illus2 title="The bit begins spinning"}

Five ticks later, this packet arrives at the server, which takes its spin
value from it and reflects that value back on the next packet it sends, as
shown in {{illus3}}. The spin edge is now in transit toward the client.

~~~~~
+--------+   1  1  1  1  1   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   0  0  0  0  1   +--------+
~~~~~
{: #illus3 title="Server reflects the spin edge"}

Five ticks later, the 1-marked packet arrives at the client, which inverts its
spin value and sends the inverted value on the next packet it sends, as shown
in {{illus4}}.

~~~~~
      obs. points  X  Y
+--------+   0  1  1  1  1   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   1  1  1  1  1   +--------+
                      Y
~~~~~
{: #illus4 title="Client inverts the spin edge"}

Here we can also see how measurement works. An observer watching the signal at
single observation point X in {{illus4}} will see an edge every 10 ticks, i.e. once
per RTT. An observer watching the signal at a symmetric observation point Y in
{{illus4}} will see a server-client edge 4 ticks after the client-server edge,
and a client-server edge 6 ticks after the server-client edge, allowing it to
compute component RTT.

{{illus5}} shows how this mechanism works in the presence of reordering. Here,
packet C carries the spin edge, and packet B is reordered on the way to the
client. In this case, the client will begin sending spin 1 after the arrival
of C, and ignore the spin bit flip to 1 on packet B, since B < C; i.e. it does not
increment the highest packet number seen.

~~~~~
+--------+   0  0  0  0  0   +--------+
|        |     -------->     |        |
| Client |                   | Server |
|        |     <--------     |        |
+--------+   1  0  1  0  0   +--------+
    PN=      A  C  B  D  E
~~~~~
{: #illus5 title="Handling reordering"}

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
same as those for passive RTT measurement in general.

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

A one-bit spin signal is resistent to reordering during signal generation,
since the spin value is only updated at each endpoint on a packet that
advances the packet counter. However, it does not allow an observer to correct
for reordered or lost edges, and it requires observers to use heuristics to
determine whether an edge was delayed at the sender.

The Valid Edge Counter (VEC) addresses all these issues with two additional
bits added to each packet, encoding values from 0 to 3. The VEC is set by each
endpoint as follows; unlike the spin bit, note that there is no difference
between client and server handling of the VEC:

- By default, the VEC is set to 0.
- If a packet contains an edge (transition 0->1 or 1->0) in the spin signal,
  and that edge is delayed (sent more than a configured delay since the edge
  was received, defaulting to 1ms), the VEC is set to 1.
- If a packet contains an edge in the spin signal, and that edge is not
  delayed, the VEC is set to the value of the VEC that accompanied the last
  incoming spin bit transition plus one, holding at 3.

The VEC can be used by observers to determine whether an edge in the spin bit
signal is valid or not, as follows:

- A packet containing an no apparent edge in the spin signal, regardless of
  the VEC value, cannot be used for RTT measurement.
- A packet containing an apparent edge in the spin signal, but with a VEC of
  0, cannot be used for RTT measurement.
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

This mechanism allows observers to recognize spurious edges due to reordering
and delayed edges due to loss, since these packets will have been sent with
VEC 0: they were not edges when they were sent. In addition, it allows senders
to signal that a valid edge was delayed because the sender was
application-limited: these edges are sent with the VEC set to 0 by the sender,
prompting the VEC to count back up over the next RTT.

