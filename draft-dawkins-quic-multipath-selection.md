---
title: Path Selection for Multiple Paths In QUIC
abbrev: QUIC Multipath Path Selection
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

  RFC9000:
  I-D.ietf-quic-datagram:
  
  I-D.an-multipath-quic:
  I-D.an-multipath-quic-application-policy:
  I-D.an-multipath-quic-traffic-distribution:
  I-D.chan-quic-owl:
  I-D.deconinck-quic-multipath:
  I-D.huitema-quic-mpath-option:
  I-D.huitema-quic-mpath-req:
  I-D.liu-multipath-quic:
  I-D.piraux-intarea-quic-tunnel-session:
  
  I-D.bonaventure-iccrg-schedulers:
  
  I-D.dawkins-quic-what-to-do-with-multipath:

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

  S2-2104599:
    author:
      - ins: Lenovo, Motorola Mobility
    title: Study on Access Traffic Steering, Switching and Splitting support in the 5G system architecture; Phase 3
    date: 2021
    target: https://www.3gpp.org/ftp/tsg_sa/WG2_Arch/TSGS2_145E_Electronic_2021-05/Docs/S2-2104599.zip

  RFC8684:

  ICCRG-charter:
    title: IRTF Internet Congestion Control Research Group Charter
    target: https://datatracker.ietf.org/rg/iccrg/about/

  QUIC-charter:
    title: IETF QUIC Working Group Charter
    target: https://datatracker.ietf.org/wg/quic/about/
  QUIC-interim-20-10:
    title: IETF QUIC Working Group Virtual Interim Meeting - October 2020
    target: https://github.com/quicwg/wg-materials/tree/master/interim-20-10
    date: October 2020
   
--- abstract

In QUIC working group discussions about proposals to use multiple paths, an obvious question came up - given multiple paths, how does QUIC select paths to send packets over?

The answer to that question may inform decisions in the QUIC working group about the scope of any multipath extensions considered for experimentation and adoption. 

This document is intended to summarize aspects of path selection from those contributions and conversations.

It is recognized that path selection is not the only important open question about QUIC Multipath, but other open questions are out of scope for this document. 

--- middle

# Introduction {#intro}

In QUIC working group {{QUIC-charter}} discussions about proposals to use multiple paths, an obvious question came up - given multiple paths, how does QUIC select paths to send packets over?

The answer to that question may inform decisions in the QUIC working group about the scope of any multipath extensions considered for experimentation and adoption. 

This document is intended to summarize aspects of path selection from those contributions and conversations.

It is recognized that path selection is not the only important open question about QUIC Multipath, but other open questions are out of scope for this document. 

##Why We Should Look at Path Selection Strategies Now {#atsss}

One of the first questions that's come up in discussions about multiple paths for QUIC has been about path selection. As soon as an implementation has multiple paths available, it must decide how to use these paths. The {{RFC9000}} answer, relying on connection migration, is "if you have multiple paths available, you can validate more than one connection at a time, but you only send on one connection at a time, and you migrate to another connection when you decide sending on the current connection is no longer appropriate. How you decide to migrate to another connection is up to you".

That has been a fine answer for many of the implementers who have worked on the first version of QUIC, and have deployed it in their networks. For other implementers, targeting other use cases and other networking environments, it may not be sufficient. 

To take only one example, one of several presentations at {{QUIC-interim-20-10}} described aspects of 3GPP Access Traffic Steering, Switch and Splitting support (ATSSS), which contained four "Steering Modes" as part of Rel-16 in 2019 {{TS23501}}, each of which corresponding roughly to a path selection strategy described in {{strategies}} of this document. A study on "ATSSS Phase 2" {{TR23700-93}} included four more Steering Modes for Rel-17, expected to be finalized in mid-2021, and none of these eight (so far) Steering Modes are based on QUIC - they are based on Multipath TCP ({{RFC8684}} or simple IP packet forwarding. And if that were not enough, a proposal for a study on "ATSSS Phase 3" {{S2-2104599}} was provided to the SA2 145-e meeting in May 2021. Some of the ATSSS strategies rely in 5G network internals and don't translate to the broader Internet, but most do translate, and 3GPP participants certainly aren't the only people thinking about path selection strategies. 

Since the various proposals presented at {{QUIC-interim-20-10}} were developed in isolation from each other, the path selection strategies that they reflect may be similar between proposals, but not quite the same, and none of the proposals presented had more than two strategies in common with any other proposal. 

Given the number of path selection strategies being considered, implemented, and even deployed in any number of venues, and the potential for combinatorial explosion as described in {{combo}}, it seems that identifying common aspects of path selection strategies, sooner rather than later, is important. 

## Notes for Readers {#readernotes}

This document is an informational Internet-Draft, not adopted by any IETF working group, and does not carry any special status within the IETF.

Please note well that this document reflects the author's current understanding of past working group discussions and proposals.  Contributions that add or improve considerations are welcomed, as described in {{contrib}}. 

## Minimal Terminology {#min-term}

In this document, "QUIC multipath" is only used as shorthand for "QUIC using multiple paths". It does not refer to a specific proposal.

In this document, "path selection strategy" means the policy that a QUIC sender uses to guide its choice between multiple interfaces of a QUIC connection for "the next packet". 

This document adopts three terms, stolen from {{TS23501}}, that seemed helpful in previous discussions about multipath in the QUIC working group. 

* Traffic Steering - selecting an initial path (in {{RFC9000}}, this would be "validating a connection, and then using it". Although an {{RFC9000}} client can validate multiple connections, the client will only use one validated connection at a time. 
* Traffic Switching - selecting a different validated path (in {{RFC9000}}, this is something like "migrating to a new validated connection", although whether connection migration as defined in {{RFC9000}}) would be sufficient is discussed in {{implic}}). 
* Traffic Splitting - using multiple validated paths simultaneously (this would almost certainly require an extension beyond connection migration as defined in {{RFC9000}}).

"Traffic Steering" does not require any extension to {{RFC9000}}, and is not discussed further in this document. The focus will be on "Traffic Switching" and "Traffic Splitting". 

## Contribution and Discussion Venues for this draft. {#contrib}

(Note to RFC Editor - if this document ever reaches you, please remove this section)

This document is under development in the Github repository at https://github.com/SpencerDawkins/quic-multipath-selection.

Readers are invited to open issues and send pull requests with contributed text for this document, but since the document is intended to guide discussion for the QUIC working group, substantial discussion of this document should take place on the QUIC working group mailing list (quic@ietf.org). Mailing list subscription and archive details are at https://www.ietf.org/mailman/listinfo/quic.

#Background for this document {#background}

A number of individual draft proposals for "QUIC over multiple paths" have been submitted to the IETF QUIC and INTAREA working groups, dating back as far as 2017. The author thinks that the complete list is as follows (and reminders for proposals he missed are welcomed): 

* {{I-D.an-multipath-quic}}
* {{I-D.an-multipath-quic-application-policy}}
* {{I-D.an-multipath-quic-traffic-distribution}}
* {{I-D.chan-quic-owl}}
* {{I-D.deconinck-quic-multipath}}
* {{I-D.deconinck-quic-multipath}}, 
* {{I-D.huitema-quic-mpath-req}}
* {{I-D.huitema-quic-mpath-option}})
* {{I-D.liu-multipath-quic}}
* {{I-D.piraux-intarea-quic-tunnel-session}}

{{I-D.bonaventure-iccrg-schedulers}} has also been submitted to the Internet Congestion Control Research Group {{ICCRG-charter}} in the Internet Research Task Force. It contains specific proposals for implementing some multipath schedulers, and includes some discussion of path selection relevant to this document. 

One point of confusion in QUIC working group discussions was that the various proposals (also using Multipath TCP {{RFC8684}}, so not all proposals were QUIC-specific) discussed in working group meetings and on the QUIC mailing list were from various proponents who weren't solving the same problem. This meant that no two of the use cases presented at the QUIC working group virtual interim on QUIC Multipath {{QUIC-interim-20-10}} were relying on the same strategies.

It seemed useful to collect the path selection strategies described in those proposals, to look for common elements, and to write them down in one place, to allow more focused discussion. {{I-D.dawkins-quic-what-to-do-with-multipath}} was intended to summarize, at a high level, various proposals for the use of multipath capabilities in QUIC, both inside the IETF and outside the IETF, in order to identify elements that were common across proposals. This draft tries to describe the impact of these various strategies on potential QUIC Multipath extensions. 

One element that is certainly worth considering is whether the path selection strategies that have been proposed can be satisfied using a small number of "building block" strategies. 

#Overview of Proposed Path Selection Strategies {#strategies}

The following strategies were discussed at {{QUIC-interim-20-10}}, and afterwards on the QUIC mailing list. These are summarized in this section, are described in more detail in {{I-D.dawkins-quic-what-to-do-with-multipath}}, and are attributed to various proposals in that document.

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

##Active-Standby {#act-stand}

The traffic associated with a specific flow will be sent via a specific path (the 'active path') and switched to another path (called 'standby path') when the active access is unavailable.

##Latency Versus Bandwidth {#lat-band}

Some traffic might be sent over a network path with lower latency and other traffic might be sent over a different network path with higher bandwidth.

##Bandwidth Aggregation/Load Balancing {#load-bal}

Traffic is sent using all available paths simultaneously, so that all available bandwidth is utilized, likely based on something like weighted round-robin path selection. This strategy is often used for bulk transfers. 

##Minimum RTT Difference {#min-rtt}

Traffic is sent over the path with the lowest smoothed RTT among all available paths, in order to minimize latency for latency-sensitive flows. 

##Round-Trip-Time Thresholds {#rtt-thresh}

Traffic is sent over the first path with a smoothed RTT that meets a certain threshold.

##RTT Equivalence {#rtt-sam}

When multiple paths each have sufficiently similar smoothed RTTs, traffic is sent over all of these paths. This is similar to "Bandwidth Aggregation/Load Balancing", with the additional qualification that not all available paths are used for this traffic. 

##Priority-based {#prior}

Priorities are assigned to each path (often by association with network interfaces). Traffic is sent on a highest-priority path until it becomes congested, and then "overflows" onto a lower-priority path. 

##Redundant {#redundant}

Traffic is replicated over two or more paths. This strategy could be used continuously, but is more commonly used when measured network conditions indicate that redundant sending may be necessary to increase the likelihood that at least one copy of each packet will arrive at the receiver. 

##Control Plane Versus Data Plane {#cp-dp}

An application might stream media over one or more available paths (based on one of the other strategies named in this section), but might send ACK traffic or retransmission over a path specifically chosen for that purpose. This is more likely to be beneficial if the path characteristics differ significantly between available paths - for example, satellite uplink/downlink stations connected by both higher-bandwidth Low Earth Orbit satellite paths and lower-bandwidth cellular or landline paths.

##Combinations of Strategies {#combo}

In addition to the strategies described above, it is also possible to combine these strategies. For example, a scheduler might use load-balancing over three paths, but when one of the paths becomes unavailable, the scheduler might switch to the two paths that are still available, in a way similar to Active-Standby. This is very much an example chosen at random - potentially, there are many combinations that could be useful. 

#Implications for QUIC Multipath {#implic}

This section summarizes potential implications for "Multipath QUIC" of path selection strategies described in {{strategies}}, dividing them between "Traffic Switching" ({{single-active}}) and "Traffic Splitting" ({{mult-active}}).

##Selecting a Single Path Among Multiple Validated Paths ("Traffic Switching") {#single-active}

If a sender using Active-Standby (described in {{act-stand}}) does not perform frequent path switching, it can likely be supported using connection migration as defined in {{RFC9000}} without change. 

* The caveat here is that connection migration can include the also-implicit assumption that an endpoint can free up resources associated with the previously-active path. If connection migration happens often enough, the endpoint may spend considerable time "thrashing" between allocating resources and quickly freeing them. Of course, if a sender is frequently selecting a new path for connection migration, this probably degenerates into one of the other path selection strategies. 

Some path selection strategies could be supported by a mechanism as simple as the one proposed in {{I-D.huitema-quic-mpath-option}}, which replaces "the implicit signaling of path migration through data transmission, by means of a new PATH_OPTION frame" (this isn't intended to imply the proposal is simple, only the explicit signaling), if the receiver uses this option to notify the sender of the preferred path. For example, Minimum RTT Difference (described in {{min-rtt}}) and Round-Trip-Time Thresholds (described in {{rtt-thresh}}) likely fall into this category. 

Some path selection strategies are exploiting a relatively long-lived difference between paths - for example, Latency Versus Bandwidth (described in {{lat-band}}), Priority-based (described in {{prior}}), and Control Plane Versus Data Plane (described in {{cp-dp}}) may fall into this category. One might wonder why these senders would need to use a single "multipath connection", rather than multiple {{RFC9000}} connections, for these cases, but if there is a reason to use a single multipath connection, a mechanism similar to the one proposed in {{I-D.huitema-quic-mpath-option}} could also be used in these cases. 

##Selecting Multiple Active Paths ("Traffic Splitting") {#mult-active}

Some path selection strategies are treating more than one path as a set of active paths, whether the sender is performing "Traffic Splitting" (as defined in {{min-term}})), as is the case for Bandwidth Aggregation/Load Balancing (described in {{load-bal}}) and RTT Equivalence (described in {{rtt-sam}}), or simply transmitting the same packet across multiple paths, as is the case for Redundant (described in {{redundant}}). 

For these cases, a more complex mechanism is likely required. 

##Arbitrary Combinations

Because it is simple enough to imagine various combinations of strategies (as described in {{combo}}), it seems important to understand what basic building blocks are required in order to support the strategies that seem common across a variety of use cases, because interactions between strategies may have significant implications for QUIC Multipath that might not arise when considering strategies in isolation. 

This seems especially important because existing proposals for QUIC Multipath don't use the same vocabulary to describe path selection strategies, so implementations may not behave in the same way, even if they are each using a strategy that seems to be common. 

#Next Steps

If this discussion is useful, it may also be useful to take the next step, and identify potential building blocks that can be used to construct the path selection strategies described in {{single-active}} and {{mult-active}}.

# IANA Considerations

This document does not make any request to IANA.

# Security Considerations

QUIC-specific security considerations are discussed in Section 21 of {{RFC9000}}.

Section 6 of {{I-D.ietf-quic-datagram}} discusses security considerations specific to the use of the Unreliable Datagram Extension to QUIC.

Some "Multipath QUIC"-specific security considerations can be found in the corresponding section of {{I-D.deconinck-quic-multipath}}. 

Having said that, it may be best to repeat the security considerations section from {{I-D.huitema-quic-mpath-option}}: "TBD.  There are probably ways to abuse this.".

# Acknowledgments

Your name could appear here. Please comment and contribute, as per {{contrib}}. 