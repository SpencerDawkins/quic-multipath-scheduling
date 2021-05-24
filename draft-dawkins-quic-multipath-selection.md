---
title: Path Selection for Multiple Paths In QUIC
abbrev: QUIC Multipath Scheduling
docname: draft-dawkins-quic-multipath-selection-latest
date:
category: info

ipr: trust200902
area: transport
workgroup: QUIC Working Group
keyword: Internet-Draft

coding: us-ascii
stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
  ins: S. Dawkins
  name: Spencer Dawkins
  organization: Tencent America LLC
  email: spencerdawkins.ietf@gmail.com

informative:

  I-D.ietf-quic-transport:
  I-D.ietf-quic-datagram:
  I-D.ietf-quic-http:
  I-D.deconinck-quic-multipath:
  I-D.an-multipath-quic:
  I-D.an-multipath-quic-application-policy:
  I-D.liu-multipath-quic:
  I-D.huitema-quic-mpath-option:
  I-D.bonaventure-iccrg-schedulers:
  I-D.amend-iccrg-multipath-reordering:
  I-D.dawkins-quic-what-to-do-with-multipath:
  I-D.dawkins-quic-multipath-questions:

  TS23501:
    author:
      - ins: 3GPP (3rd Generation Partnership Project)
    title: Technical Specification Group Services and System Aspects; System Architecture for the 5G System; Stage 2 (Release 16)
    date: 2019
    target: https://www.3gpp.org/ftp/Specs/archive/23_series/23.501/
  TR23700-93:
    author:
      - ins: 3GPP (3rd Generation Partnership Project)
    title: Technical Specification Group Services and System Aspects; Study on access traffic steering, switch and splitting support in the 5G System (5GS) architecture; Phase 2 (Release 17)
    date: 2021
    target: https://www.3gpp.org/ftp/Specs/archive/23_series/23.700-93/

  S2-2104582:
    author:
      - ins: Lenovo, Motorola Mobility
    title: Discussion on ATSSS Enhancements
    date: 2021
    target: https://www.3gpp.org/ftp/tsg_sa/WG2_Arch/TSGS2_145E_Electronic_2021-05/Docs/S2-2104582.zip

  RFC0793:
  RFC4960:
  RFC5061:
  RFC7540:
  RFC8684:

  ICCRG-charter:
    title: IRTF Internet Congestion Control Research Group Charter
    target: https://datatracker.ietf.org/rg/iccrg/about/

  QUIC-charter:
    title: IETF QUIC Working Group Charter
    target: https://datatracker.ietf.org/wg/quic/about/
  QUIC-IETF-109-minutes:
    title: IETF QUIC Working Group IETF 109 Meeting - November 2020 - Minutes
    target: https://datatracker.ietf.org/doc/minutes-109-quic/
  QUIC-interim-20-10:
    title: IETF QUIC Working Group Virtual Interim Meeting - October 2020
    target: https://github.com/quicwg/wg-materials/tree/master/interim-20-10
    date: October 2020
  QUIC-interim-20-10-minutes:
    title: IETF QUIC Working Group Virtual Interim Meeting - October 2020 - Minutes
    target: https://github.com/quicwg/wg-materials/tree/master/interim-20-10
    
--- abstract

In QUIC working group discussions about proposals to use multiple paths, one obvious question came up - how does QUIC select paths for packet scheduling over multiple paths?

This document is intended to summarize aspects of path selection from those contributions and conversations.

--- middle

# Introduction {#intro}

In QUIC working group discussions about proposals to use multiple paths, one obvious question came up - how does QUIC select paths for packet scheduling over multiple paths?

This document is intended to summarize aspects of path selection from those contributions and conversations.

## Notes for Readers {#readernotes}

This document is an informational Internet-Draft, not adopted by any IETF working group, and does not carry any special status within the IETF.

Please note well that this document reflects the author's current understanding of past working group discussions and proposals.  Contributions that add or improve considerations are welcomed, as described in {{contrib}}. 

## Contribution and Discussion Venues for this draft. {#contrib}

(Note to RFC Editor - if this document ever reaches you, please remove this section)

This document is under development in the Github repository at https://github.com/SpencerDawkins/draft-dawkins-quic-multipath-selection.

Readers are invited to open issues and send pull requests with contributed text for this document, but since the document is intended to guide discussion for the QUIC working group, substantial discussion of this document should take place on the QUIC working group mailing list (quic@ietf.org). Mailing list subscription and archive details are at https://www.ietf.org/mailman/listinfo/quic.

## Minimal Terminology {#min-term}

This document adopts three terms, stolen from {{TS23501}}, that seemed helpful in discussions about multipath in the QUIC working group. 

* Traffic Steering - selecting a path (in {{I-D.ietf-quic-transport}}, this would be "validating a connection"). 
* Traffic Switching - selecting a different path (in {{I-D.ietf-quic-transport}}, this would be "migrating a connection"). 
* Traffic Splitting - using multiple paths simultaneously (this may require a QUIC extension).

#Background for this document {#background}

In this document, "QUIC multipath" is only used as shorthand for "QUIC using multiple paths". It does not refer to a specific proposal. 

It cannot be emphasized enough that multiple proposals for "QUIC multipath" have been submitted to the the quic working group (at a minimum, {{I-D.deconinck-quic-multipath}}, {{I-D.an-multipath-quic}}, {{I-D.an-multipath-quic-application-policy}}, 
{{I-D.liu-multipath-quic}}, and {{I-D.huitema-quic-mpath-option}}).

{{I-D.bonaventure-iccrg-schedulers}} has also been submitted to the Internet Congestion Control Research Group {{ICCRG-charter}} in the Internet Research Task Force. It contains specific proposals for implementing some multipath schedulers, and includes some discussion of path selection relevant to this document. 

#Path Selection Strategies {#strategies}

One point of confusion in QUIC working group discussions was that various proposals (dating back to the use of Multipath TCP {{RFC8684}}, so not QUIC-specific) discussed in working group meetings and on the QUIC mailing list were from various proponents who weren't solving the same problem, so no two of the use cases presented at the QUIC working group virtual interim on Multipath {{QUIC-interim-20-10}} were relying on the same strategies.

The following strategies were discussed at {{QUIC-interim-20-10}}, and afterwards on the QUIC mailing list. These are summarized in this section, described in more detail in {{I-D.dawkins-quic-what-to-do-with-multipath}}, and are attributed to various proposals in that document.

* Active-Standby - described in {{act-stand}}
* Latency Versus Bandwidth - described in {{lat-band}}
* Bandwidth Aggregation/Load Balancing - described in {{load-bal}}
* Minimum RTT Difference - described in {{min-rtt}} 
* Round-Trip-Time Thresholds - described in {{rtt-thresh}}
* RTT Equivalence - described in {{rtt-sam}}
* Priority-based - described in {{prior}}
* Redundant - described in {{redundant}}
* Control Plane Versus Data Plane - described in {{cp-dp}}
* Combinations of Strategies - described in {{combo}}

The terminology defined in {{min-term}} is used in this section. 

##Active-Standby {#act-stand}

The traffic associated with a specific flow will be sent via a specific path (the 'active path') and switched to another path (called 'standby path') when the active access is unavailable.

##Latency Versus Bandwidth {#lat-band}

Some traffic is sent over a network path with lower latency and other traffic is sent over a different network path with higher bandwidth. 

##Bandwidth Aggregation/Load Balancing {#load-bal}

Traffic is sent over all available paths simultaneously. This strategy is often used for bulk transfers. 

##Minimum RTT Difference {#min-rtt}

Traffic is sent over the path with the lowest smoothed RTT among all available paths, in order to minimize latency for latency-sensitive flows. 

##Round-Trip-Time Thresholds {#rtt-thresh}

Traffic is sent over the first path with a smoothed RTT that meets a certain threshold.

##RTT Equivalence {#rtt-sam}

When multiple paths each have sufficiently similar smoothed RTTs, traffic is sent over all of these paths. This is similar to "Bandwidth Aggregation/Load Balancing", with the additional qualification that not all available paths are used for this traffic. 

##Priority-based {#prior}

Priorities are assigned to each path (often by association with network interfaces). Traffic is sent on a highest-priority path until it becomes congested, and then "overflows" onto a lower-priority path. 

##Redundant {#redundant}

Traffic is replicated over two or more paths to increase the likelihood that at least one copy of each packet will arrive at the receiver. This strategy could be used for all traffic, but is more commonly used when measured network conditions indicate that redundant sending may be beneficial. 

##Control Plane Versus Data Plane {#cp-dp}

An application might stream media over one or more available paths (based on one of the other strategies named in this section), but might send ACK traffic or retransmission over a path specifically chosen for that purpose. This is more likely to be beneficial if the path characteristics differ significantly between available paths - for example, satellite uplink/downlink stations connected by both higher-bandwidth Low Earth Orbit satellite paths and lower-bandwidth cellular or landline paths.

##Combinations of Strategies {#combo}

In addition to the strategies described above, it is also possible to combine these strategies. For example, a scheduler might use load-balancing over three paths, but when one of the paths becomes unavailable, the scheduler might switch to the two paths that are still available.

#Implications for QUIC 

The Active-Standby strategy does not rely on any extension to {{I-D.ietf-quic-transport}}. If more than one QUIC connection is validated, an endpoint can migrate a connection to a new local address by sending packets containing non-probing frames from that address. This strategy provides traffic steering and traffic switching, but does not provide traffic splitting, because only one path is in active use at any time. 



#A Personal Aside on 3GPP Access Traffic Steering, Switch and Splitting support (ATSSS) {#atsss}

Just to provide perspective on the importance of understanding packet scheduling strategies, 3GPP Access Traffic Steering, Switch and Splitting support (ATSSS) contained four scheduling strategies in Rel-16 in 2019 {{TS23501}}, all of which corresponded roughly to packet scheduling strategies described in {{strategies}}, but a study on "ATSSS Phase 2" {{TR23700-93}} included four more strategies, and a discussion paper for an "ATSSS Phase 3" was recently submitted to SA2#145-e {{S2-2104582}}. 

Some of the ATSSS packet scheduling strategies rely in 5G network internals and don't translate to the broader Internet, but others do translate, and they certainly aren't the only people thinking about multipath QUIC packet scheduling strategies. 

Given the number of strategies under discussion, or even actually implemented and deployed, and the potential for combinatorial explosion as described in {{combo}}, it seems that identifying common aspects of packet scheduling strategies, sooner rather than later, is important. 

# IANA Considerations

This document does not make any request to IANA.

# Security Considerations

QUIC-specific security considerations are discussed in Section 21 of {{I-D.ietf-quic-transport}}.

Section 6 of {{I-D.ietf-quic-datagram}} discusses security considerations specific to the use of the Unreliable Datagram Extension to QUIC.

Some multipath QUIC-specific security considerations can be found in the corresponding section of {{I-D.deconinck-quic-multipath}}. 

# Acknowledgments

Your name could appear here. Please comment and contribute, as per {{contrib}}. 