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
    name: Markus Ihlar
    org: Ericsson
    email: marcus.ihlar@ericsson.com
  -
    ins: A. Morton
    name: Al Morton
    org: AT&T
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

\[EDITOR'S NOTE: Marcus to write. Modify slightly from PR609 design, spin only
on short header packets. Coordinate with work on reserved bits in the short
header in the transport draft?]

# Using the Spin Bit for Passive RTT Measurement

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

## Limitations and Workarounds

The spin bit provides latency information only when the sender is neither
application nor flow control limited. When the sender is application-limited
by periodic application traffic, where that period is longer than the RTT,
measuring the spin bit provides information about the application period, not
the RTT. Simple heuristics based on the observed data rate per flow or changes
in the RTT series can be used to reject bad RTT samples due to application or
flow control limitation. \[EDITOR'S NOTE: more here?]

Since the spin bit logic at each endpoint considers only samples on packets
that advance the highest packet number seen, signal generation itself is
resistent to reordering. However, reordering can cause problems at an observer
by causing spurious edge detection and therefore low RTT estimates. This can
be probabilistically mitigated by the observer considering the low-order bits
of the packet number, and rejecting edges that appear out-of-order.

## Alternate RTT Measurement Approaches for Diagnosing QUIC flows

There are two alternatives to explicit signaling for passive RTT measurement
for measuring the RTT experienced by QUIC flows.

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

\[EDITOR'S NOTE: Summary of Piet's work to date goes here.]

# Use Cases for Passive RTT Measurement

\[EDITOR'S NOTE: Roni on video]

\[EDITOR'S NOTE: Emile on interdomain]

\[EDITOR'S NOTE: Markus on AQM?]

\[EDITOR'S NOTE: Others?]

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
databases. Therefore, in the general case, when an endpoint's IP address is
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
request 609 on draft-ietf-quic-transport. Thanks to the QUIC RTT Design Team
for discussions leading especially to the measurement limitations and privacy
and security considerations sections.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
