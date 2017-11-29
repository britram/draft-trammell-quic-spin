---
title: The Addition of a Spin Bit to the QUIC Transport Protocol
abbrev: Spin Bit
docname: draft-trammell-quic-spin-latest
date:
category: info

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
    ins: P. De Vaere
    name: Piet De Vaere
    org: ETH Zurich
    email: piet@devae.re
  -
    ins: R. Even
    name: Roni Even
    org: Huawei
    email: roni.even@huawei.com
  -
    ins: G. Fioccola
    name: Giuseppe Fioccola
    org: Telecom Italia
    email: giuseppe.fioccola@telecomitalia.it
  -
    ins: T. Fossati
    name: Thomas Fossati
    org: Nokia
    email: thomas.fossati@nokia.com
  -
    ins: M. Ihlar
    name: Marcus Ihlar
    org: Ericsson
    email: marcus.ihlar@ericsson.com
  -
    ins: A. Morton
    name: Al Morton
    org: AT&T Labs
    email: acmorton@att.com
  -
    ins: E. Stephan
    name: Emile Stephan
    org: Orange
    email: emile.stephan@orange.com

normative:

informative:
  TRILAT:
    title: On the Suitability of RTT Measurements for Geolocation (https://github.com/britram/trilateration/blob/paper-rev-1/paper.ipynb)
    author:
      -
        ins: B. Trammell
    date: 2017-08-30
  TOKYO-PING:
    title: From Paris to Tokyo - On the Suitability of ping to Measure Latency (ACM IMC 2014)
    author:
      -
        ins: C. Pelsser
      -
        ins: L. Cittadini
      -
        ins: S. Vissicchio
      -
        ins: R. Bush
    date: 2014-10-23
  CONUS:
    title: Comparison of Backbone Node RTT and Great Circle Distances (https://github.com/acmacm/FIXME-TBD)
    author:
      -
        ins: A. Morton
    date: 2017-09-01
  SPINBIT-REPORT:
    title: Latency Spinbit Implementation Experience
    author:
      -
        ins: P. De Vaere
    date: 2017-11-28
  MINQ:
    title: MINQ, a simple Go implementation of QUIC (https://github.com/ekr/minq)
    author:
      -
        ins: E. Rescorla
    date: 2017-11-28
  MOKUMOKUREN:
    title: Mokumokuren, a lightweight flow meter using gopacket (https://github.com/britram/mokumokuren)
    author:
      -
        ins: B. Trammell
    date: 2017-11-12
  IMC-CONGESTION:
    title: Challenges in Inferring Internet Interdomain Congestion (in Proc. ACM IMC 2014)
    author:
      -
        ins: M. Luckie
      -
        ins: A. Dhamdhere
      -
        ins: D. Clark
      -
        ins: B. Huffaker
      -
        ins: k. claffy
    date: 2014-11

--- abstract

This document summarizes work to date on the addition of a "spin bit",
intended for explicit measurability of end-to-end RTT on QUIC flows. It
proposes a detailed mechanism for the spin bit, describes how to use it to
measure end-to-end latency, discusses corner cases and workarounds therefor in
the measurement, describes experimental evaluation of the mechanism done to
date, and examines the utility and privacy implications of the spin bit. As
the overhead and risk associated with the spin bit are negligible, and the
utility of a passive RTT measurement signal at higher resolution than once per
flow is clear, this document advocates for the addition of the spin bit to the
protocol.

--- middle

# Introduction

\[EDITOR'S NOTE: Brian to write frontmatter. Why we care, why this draft exists.]

# The Spin Bit Mechanism {#mechanism}

The latency spin bit enables latency monitoring from observation points on
the network path. The bit is set by the endpoints in the following way:

* The server sets the spin bit value to the value of the
  spin bit in the packet received from the client with
  the largest packet number.

* The client sets the spin bit value to the opposite
  of the value set in the packet received from the server with the
  largest packet number, or to 0 if no packet as been received yet.

If packets are delivered in order, this procedure will cause the spin bit
to change value in each direction once per round trip. Observation points can
estimate the network latency by observing these changes in the latency spin
bit, as described in {{usage}}.

## Proposed Short Header Format Including Spin Bit

Since it is possible to measure handshake RTT without a spin bit (see
{{other-bad-ideas}}), it is sufficient to include the spin bit in the short
packet header. This proposal suggests to ues the second most significant bit
(0x40) of the first octet in the short header for the spin bit.

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|S|C|K|Type(4)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     [Connection ID (64)]                      +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: #fig-short-header title="Short Header Format including proposed Spin Bit"}

This will shift the Connection ID flag and the Key Phase Bit to 0x20 and 0x10
respectively, and will limit the number of available short packet types to 16.

# Using the Spin Bit for Passive RTT Measurement {#usage}

When a QUIC flow is sending at full rate (i.e., neither application nor flow
control limited), the latency spin bit in each direction changes value once
per round-trip time (RTT). An on-path observer can observe the time difference
between edges in the spin bit signal to measure one sample of end-to-end RTT.
Note that this measurement, as with passive RTT measurement for TCP, includes
any transport protocol delay (e.g., delayed sending of acknowledgements)
and/or application layer delay (e.g., waiting for a request to complete). It
therefore provides devices on path a good instantaneous estimate of the RTT as
experienced by the application. A simple linear smoothing or moving minimum
filter can be applied to the stream of RTT information to get a more stable
estimate.

We note that the Latency Spin Bit, and the measurements that can be done with
it, can be seen as an end-to-end extension of a special case of the alternate
marking method described in {{?ALT-MARK=I-D.ietf-ippm-alt-mark}}.

## Limitations and Workarounds {#limitations}

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
resistent to reordering. However, reordering can cause problems at an observer
by causing spurious edge detection and therefore low RTT estimates. This can
be probabilistically mitigated by the observer tracking the low-order bits of
the packet number, and rejecting edges that appear out-of-order.

## Alternate RTT Measurement Approaches for Diagnosing QUIC flows {#other-bad-ideas}

There are two broad alternatives to explicit signaling for passive RTT
measurement for measuring the RTT experienced by QUIC flows.

The first of these is handshake RTT measurement. As described in
{{?QUIC-MGT=I-D.ietf-quic-manageability}}, the packets of the QUIC handshake are
distinguishable on the wire in such a way that they can be used for one RTT
measurement sample per flow: the delay between the client initial and the
server cleartext packet can be used to measure "upstream" RTT (between the
observer and the server), and the delay between the server cleartext packet
and the next client cleartext packet can be used to measure "downstream" RTT
(between the client and the observer). When RTT measurements are used in large
aggregates (all flows traversing a large link, for example), a methodology
based on handshake RTT could be used to generate sufficient samples for some
use cases (e.g. \[EDITOR'S NOTE: cite use cases here]) without the spin bit.
However, this methodology would rely on the assumption that the difference
between handshake RTT and nominal in-flow RTT is negligible. Specifically, (1)
any additional delay required to compute any cryptographic parameters must be
negligible with respect to network RTT; (2) any additional delay required to
establish state along the path must be negligible with respect to network RTT;
and (3) network treatment of initial packets in a flow must identical to that
of later packets in the flow. When these assumptions cannot be shown to hold,
spin-bit based RTT measurement is preferable to handshake RTT measurement,
even for applications for which handshake RTT measurement would otherwise be
suitable.

The second alternative is parallel active measurement: using ICMP Echo Request
and Reply {{?RFC0792}} {{?RFC4433}}, a dedicated measurement protocol like
TWAMP {{?RFC5357}}, or a separate diagnostic QUIC flow to measure RTT.
Regardless of protocol, the active measurement must be initiated by a client
on the same network as the client of the QUIC flow(s) of interest, or a
network close by in the Internet topology, toward the server. Note that there
is no guarantee that ICMP flows will receive the same network treatment as the
flows under study, both due to differential treatment of ICMP traffic and due
to ECMP routing (see e.g. {{TOKYO-PING}}). TWAMP and QUIC diagnostic flows,
though both use UDP, have similar issues regarding ECMP. However, in
situations where the entity doing the measurement can guarantee that the
active measurement traffic will traverse the subpaths of interest (e.g.
residential access network measurement under a network architecture and
business model where the network operator owns the CPE), active measurement
can be used to generate RTT samples at the cost of at least two non-productive
packets sent though the network per sample.

## Experimental Evaluation

We have evaluated the effectiveness of the spin bit in an emulated network
environment. The spin bit was added to a fork of {{MINQ}}, using the mechanism
described in {{mechanism}}, but with the spin bit appearing in a measurement
byte added to the header for passive measurability experiments. Spin bit
measurement support was added to {{MOKUMOKUREN}}. Full results of these
ongoing experiments are available online in {{SPINBIT-REPORT}}, but we
summarize our findings here.

First, we confirm that the spin bit works as advertised: it provides one
useful RTT sample per RTT to any passive observer of the flow. This sample
tracks each sender's local instantaneous estimate of RTT as well as the
expected RTT (i.e., defined by the emulation) fairly well. One surprising
implication of this is that the spin bit provides _more_ information than is
available by local estimation to an endpoint which is mostly receiving data
frames and sending mainly ACKs, and as such can also be useful in purely
endpoint-local observations of the RTT evolution during the flow. The spin bit also works correctly under moderate to heavy packet loss and jitter.

Second, we confirm that the spin bit can be easily implemented without
requiring deep integration into a QUIC implementation. Indeed, it could be
implemented completely independently, as a shim, aside from the requirement
that the spin bit value be integrity-protected along with the rest of the QUIC
header.

Third, we performed experiments focused on the intermittent-sender problem
described in {{limitations}}. We confirm that the spinbit does not provide
useful RTT samples after the handshake when packets are only sent
intermittently. Simple heuristics can be used to recognize this situation,
however, and to reject these RTT samples. We also find that a simple
sender-side heuristic can be used to determine whether a sample will be
useful. If a sender sends a packet more than a specified delay (e.g. 1ms)
after the last packet received by the client, it knows that any latency spin
observation of that packet will be invalid. If a second "spin valid" bit were
available, the sender could then mark that packet "spin invalid". Our
experiments show that this simple heuristic and spin validity bit are
succesful in marking all packets whose RTT samples should be rejected.

Fourth, we performed experiments focused on the reordering problem described
in {{limitations}}. We find that while reordering can cause spurious samples
at a naive observer, two simple approaches can be used to reject spurious RTT
samples due to reordering. First, a two-bit spin signal that always advances
in a single direction (e.g. 00 -> 01 -> 10 -> 11) successfully rejects all
reordered samples, including under amounts of reordering that render the
transport itself mostly useless. However, adding a bit is not necessary:
having the observer keep the least significant bits of the packet number, and
rejecting samples from packets that do not advance by one, as suggested in
{{limitations}}, is essentially as successful as a two-bit spin signal in
mitigating the effects of reordering on RTT measurement.

Fifth, we performed parallel active measurements using ping, as described in
{{other-bad-ideas}}. In our emulated network, the ICMP packets and the QUIC
packets traverse the same links with the same treatment, and share queues at
each link, which mitigates most of the issues with ping. We find that while
ping works as expected in measuring end-to-end RTT, it does not track the
sender's estimate of RTT, and as such does not measure the RTT experienced by
the application layer as well as the spin bit does.

In summary, our experiments show that the spin bit is suitable for purpose,
can be implemented with minimal disruption, and that most of the problems
identified with it in specific corner cases can be easily mitigated. See {{SPINBIT-REPORT}} for more.

# Use Cases for Passive RTT Measurement

This section describes current use cases for passive RTT measurement with TCP,
i.e., the matching of packets based on sequence and acknowledgment numbers, or
timestamps and timestamp echoes, in order to generate upstream and downstream
RTT samples which can be added to get end-to-end RTT, as with handshake RTT in
{{other-bad-ideas}}. These current use cases would be consumers of RTT samples
measured from the spin bit as in {{usage}}.

In all cases, the basic measurement methodology follows one of a few basic
variants. The RTT evolution of a flow or a set of flows can be compared to
baseline or expected RTT measurements for flows with the same
characterisitcs... \[EDITOR'S NOTE: other variants go here]

## Interdomain Troubleshooting

\[EDITOR'S NOTE: Emile, anything to add here?]

\[EDITOR'S NOTE: this is rewritten from Roni's text on video; Roni, please
check.]

Network access providers are often the first point of contact by their
customers when network problems impact the performance of bandwidth-intensive
and latency-sensitive applications such as video, regardless of whether the
root cause lies within the access provider's network, the service provider's
network, on the Internet paths between them, or within the customer's own
network.

Many residential networks use WiFi (802.11) on the last segment, and WiFi
signal strength degradation manifests in high first-hop delay, due to the fact
that the MAC layer will retransmit loss packets in order to trade latency off
for loss. Measuring the RTT between endpoints on the customer network and
parts of the service provider's own infrastructure (which have predictable
delay characteristics) can be used to isolate this cause of performance
problems.

Comparing the evolution of passively-measured RTTs between a customer network
and selected other networks on the Internet to short- and medium-term baseline
measurements can similarly be used to isolate high latency to specific
networks or network segments. For example, if the RTTs of all flows to a given
service provider increase at the same time, the problem likely exists between
the access network and the service provider, or in the service provider's
network itself. On the other hand, if the RTTs of all flows for a set of
customers sharing some give access provider infrastructure increase, then the
problem is likely attributable to that infrastructure.

These measurements are particularly useful for traffic which is latency
sensitive, such as interactive video applications. However, since high latency
is often correlated with other network-layer issues such as chronic
interconnect congestion {{IMC-CONGESTION}}, it is useful for general
troubleshooting of network layer issues in an interdomain setting.

In all of these cases, handshake RTT as in {{other-bad-ideas}} would provide
limited information, presuming that its assumptions hold. Intraflow
measurements are necessary in this case to increase the baseline and
measurement data available, to increase the chance that enough samples are
available for arbitrarily small aggregates. Given that issues may be caused by
misconfigured differential treatment of traffic, intraflow measurement also
allows targeted troubleshooting of specific flows.

## Latency as input to Active Queue Management

\[EDITOR'S NOTE: Marcus?]

## Other latency measurement tasks

\[EDITOR'S NOTE: Others? Independent verification of network neutrality?]

# Privacy and Security Considerations

The privacy considerations for the latency spin bit are essentially the same
as those for passive RTT measurement in general.

A concern was raised during the discussion of this feature within the QUIC
working group and the QUIC RTT Design Team that high-resolution RTT
information might be usable for geolocation. However, an evaluation based on
RTT samples taken over 13,780 paths in the Internet from RIPE Atlas anchoring
measurements {{TRILAT}} shows that the magnitude and uncertainty of RTT data
render the resolution of geolocation information that can be derived from
Internet RTT is limited to national- or continental-scale; i.e., less
resolution than is generally available from free, open IP geolocation
databases. One reason for the inaccuracy of geolocation from network RTT
is that Internet backbone transmission facilities do not follow the great-circle
path between major nodes. Instead, major geographic features and the efficiency
of connecting adjacent major cities influence the facility routing. An evaluation
of ~3500 measurements on a mesh of 25 backbone nodes in the continental United States
shows that 85% had RTT to great-circle error of 3ms or more, making
location within US State boundaries ambigous {{CONUS}}.
Therefore, in the general case, when an endpoint's IP address is
known, RTT information provides negligible additional information.

RTT information may be used to infer the occupancy of queues along a path;
indeed, this is part of its utility for performance measurement and
diagnostics. When a link on given path has excessive buffering (on the order
of hundreds of milliseconds or more; a situation colloquially referred to as
"bufferbloat"), such that the difference in delay between an empty queue and a
full queue dwarfs normal variance and RTT along the path, RTT variance during
the lifetime of a flow can be used to infer the presence of traffic on the
bottleneck link. In practice, however, this is not a concern for passive
measurement of congestion-controlled traffic, since any observer in a
situation to observe RTT passively need not infer the presence of the traffic,
as it can observe it directly.

In addition, since RTT information contains application as well as network
delay, patterns in RTT variance from minimum, and therefore application delay,
can be used to infer or fingerprint application-layer behavior. However, as
with the case above, this is not a concern with passive measurement, since the
packet size and interarrival time sequence, which is also directly observable,
carries more information than RTT variance sequence.

We therefore conclude that the high-resolution, per-flow exposure of RTT for
passive measurement as provided by the spin bit poses negligible marginal risk
to privacy.

As shown in {{mechanism}}, the spin bit can be implemented separately from the
rest of the mechanisms of the QUIC transport protocol, as it requires no
access to any state other than that observable in the QUIC packet header
itself. We recommend that implementations take advantage of this property, to
reduce the risk that a errors in the implementation could leak private
transport protocol state through the spin bit.


# Acknowledgments

Many thanks to Christian Huitema, who originally proposed the spin bit as pull
request 609 on {{?QUIC-TRANSPORT=I-D.ietf-quic-transport}}. Thanks to the QUIC
RTT Design Team for discussions leading especially to the measurement
limitations and privacy and security considerations sections.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
