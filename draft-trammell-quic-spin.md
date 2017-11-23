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
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: P. De Vaere
    name: Piet De Vaere
    org: ETH Zurich
    email: piet@devae.re
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
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
    ins: E. Stephan
    name: Emile Stephan
    org: Orange
    email: emile.stephan@orange.com

normative:

informative:


--- abstract

This document summarizes work to date on the addition of a Spin Bit, intended
for explicit measurability of end-to-end RTT on QUIC flows. It proposes a
detailed mechanism for the spin bit, describes how to use it to measure
end-to-end latency, discusses corner cases and workarounds therefor in the
measurement, and examines the utility and privacy implications of the spin
bit.

--- middle

# Introduction

\[EDITOR'S NOTE: Brian to write frontmatter. Why we care, why this draft exists.]

# The Spin Bit Mechanism

\[EDITOR'S NOTE: Marcus to write. Modify slightly from PR609 design, spin only
on short header packets. Coordinate with work on reserved bits in the short
header in the transport draft?]

# Using the Spin Bit for Passive RTT Measurement

When a QUIC flow is sending at full rate (i.e., neither application nor flow
control limited), the latency spin bit in each direction changes value once
per round-trip time (RTT). An on-path observer can observe the duration
between observing the edges in the spin bit signal to determine end-to-end
RTT. Note that this measurement, as with passive RTT measurement for TCP,
includes any transport protocol delay (e.g., delayed sending of
acknowledgement) and/or application layer delay (e.g., waiting for a request
to complete). It therefore provides devices on path a good instantaneous
estimate of the RTT experienced by the application. A simple linear smoothing
or moving minimum filter can be applied to the stream of RTT information to
get a more stable estimate.

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

## Experimental Evaluation

\[EDITOR'S NOTE: Summary of Piet's work to date goes here.]

# Use Cases for Passive RTT Measurement

\[EDITOR'S NOTE: Roni on video]

\[EDITOR'S NOTE: Emile on interdomain]

\[EDITOR'S NOTE: Markus on AQM?]

\[EDITOR'S NOTE: Others?]

# Privacy and Security Considerations

\[EDITOR'S NOTE: Brian on trilateration experiments.]

# Acknowledgments

Many thanks to Christian Huitema, who originally proposed the spin bit as pull
request 609 on draft-ietf-quic-transport. Thanks to the QUIC RTT DT for
discussions leading especially to the measurement limitations and privacy and
security considerations sections.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
