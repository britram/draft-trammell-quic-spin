---
title: The Addition of a Spin Bit to the QUIC Transport Protocol
abbrev: Spin Bit
docname: draft-trammell-quic-spin
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
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

normative:

informative:


--- abstract

This document summarizes work to date on the addition of a Spin Bit, intended for explicit measurability of end-to-end RTT on QUIC flows. It proposes a detailed mechanism for the spin bit, describes how to use it to measure end-to-end latency, discusses corner cases and workarounds therefor in the measurement, and examines the utility and privacy implications 

--- middle

# Introduction

# The Spin Bit Mechanism

# Using the Spin Bit for Passive RTT Measurement

## Limitations

Susceptibility to Reordering

Receiver- and Sender-Limited Transmission

# Use Cases for Passive RTT Measurement

# Privacy and Security Considerations 

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
