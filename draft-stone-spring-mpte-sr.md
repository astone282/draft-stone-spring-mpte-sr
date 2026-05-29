---
title: "Multipath Traffic Engineering for Segment Routing"
abbrev: "spring-mpte-sr"
category: info

docname: draft-stone-spring-mpte-sr-latest
submissiontype: IETF
number:
date: {DATE}
consensus: true
v: 3
area: "Routing"
workgroup: "Source Packet Routing in Networking"

author:
 -
    fullname: Andrew Stone
    organization: Nokia
    email: andrew.stone@nokia.com
 -
    fullname: Vishnu Pavan Beeram
    organization: Juniper Networks
    email: vbeeram@juniper.net
 -
    fullname: Nick Buraglio
    organization: Energy Sciences Network
    email: buraglio@forwardingplane.net
 -
    fullname: Shaofu Peng
    organization: ZTE Corporation
    email: peng.shaofu@zte.com.cn

normative:
  I-D.draft-kompella-teas-mpte:
  RFC8402:
  RFC9256:
  RFC5440:
  RFC8231:
  RFC9552:
  RFC7880:
  RFC8287:
  RFC9259:
  RFC9855:
  I-D.draft-ietf-idr-segment-routing-te-policy:
  I-D.draft-ietf-idr-bgp-ls-sr-policy:

informative:
  RFC9524:



--- abstract

This document describes a mechanism to achieve Multipath Traffic Engineering for Segment Routing based networks.

--- middle

# Introduction

{{I-D.draft-kompella-teas-mpte}} introduces a multipath traffic engineering concept that combines the benefits of both Equal-Cost Multipath (ECMP)
forwarding and traffic-engineered paths. This approach uses a Directed Acyclic Graph (DAG) based forwarding mechanism, with the DAG signaled to participating network nodes.
The concept is to move beyond simple ECMP paths by incorporating both ECMP and non-ECMP paths while still adhering to traffic engineering constraints, to provide
added resiliency while also permitting better usage of link bandwidth.

This document discusses a centralized computation and signaling mechanism for SR-based networks.

This document does not propose new extensions to the Segment Routing protocol constructs.

The document assumes the reader is familiar with {{RFC8402}}, {{RFC9256}}, and {{I-D.draft-kompella-teas-mpte}}.

## Requirements Language

{::boilerplate bcp14-tagged}

The BCP 14 keywords used in this document describe the expected behavior of the centralized controller and the Junction nodes
when realizing an MPTE DAG using SR Policy and Binding SIDs. They are not intended to introduce new requirements on the underlying SR Policy,
PCEP, BGP, BGP-LS, or NETCONF protocols themselves.

# Terminology

- For MPTE terminology such as MPTED, DAG, MC, MID, Junction node and others see {{I-D.draft-kompella-teas-mpte}}.

- For SR terminology see {{RFC8402}} and {{RFC9256}}.

# MPTE vs multiple SID lists

It's worth recognizing the SR Policy information model supports encoding multiple paths on a tunnel at the ingress, by using multiple SID lists within
a Candidate Path. A Directed Acyclic Graph (DAG) can therefore be expressed as a collection of paths, each programmed as a separate
ingress SID list. However, this ingress only encoding has trade offs such as the number of paths can grow significantly with graph size, weighted flow
hashing is performed only at the ingress (not at each downstream node), and the maximum segment depth (MSD) constrains long paths that deviate from the shortest path.

Encoding the DAG's forwarding instructions across the participating Junction nodes reduces the number of ingress SID lists and allows
per-node weight tuning, at the cost of additional state in the network. The choice between ingress-only segment lists and the MPTE
DAG based mechanism depends on the traffic engineering requirements, network design, link/path metrics, and the DAG structure.

# MPTE concepts with Segment Routing

This document proposes the below concepts for applying MPTE in an SR environment.

## MPTED

The MPTED is managed by a centralized controller, such as a Path Computation Element (PCE) acting as the MC. Topology discovery is performed using BGP-LS {{RFC9552}}, while transport control plane signaling
is achieved through controller-oriented protocols such as Path Computation Element Protocol (PCEP) {{RFC5440}}, {{RFC8231}} and BGP/BGP-LS {{I-D.draft-ietf-idr-segment-routing-te-policy}}, {{I-D.draft-ietf-idr-bgp-ls-sr-policy}}.
The MC computes, manages, and distributes all forwarding information to the nodes participating in the MPTE DAG, which form the MPTED.

{{I-D.draft-kompella-teas-mpte}} specifies that a node in the MPTED is identified by its IPv6 loopback address. However, this document allows the use of a 32-bit dotted quad router ID as an
alternative to support IPv4 or dual stack networks. This value represents the headend address of a node participating in the DAG.

As per {{I-D.draft-kompella-teas-mpte}}, the controller acting as the MC is responsible for assigning the MID and incrementing the MPTED unique ID version.

## Junction Segment

The concept of a Junction Segment is introduced to describe the signaling and forwarding behavior of a Junction node in an SR network:

A junction segment is:

- realized using SR Policy construct with a single Candidate Path, a Binding SID {{RFC9256}} and one or more segment lists to one or more downstream nodes.

- similar to a Replication Segment {{RFC9524}}, but performs forwarding based on load balancing rather than replication.

- installed on nodes identified as Junction Nodes, as defined in {{I-D.draft-kompella-teas-mpte}}.

- combined with one or more Junction segments across a segment routed topology domain to form an MPTE DAG tunnel.

Since a Junction Segment may egress to multiple downstream nodes, the endpoint of the corresponding SR Policy can be set to the null value (0.0.0.0 for IPv4, :: for IPv6).
Therefore, a Junction segment is identified by its <headend, color> attribute.

The color value is mapped to the <MID, Version> tuple and is tracked by the controller.

It is RECOMMENDED that a globally consistent color value be used across all Junction Segments that belong to a single DAG instance or that serve a common DAG intent.

The MPTE tunnel is used by one or many SR Policies instantiated on one or many ingress nodes.

A Junction Segment may be used by multiple ingress SR Policies, provided that the subgraph it forwards over is fully shared by all SR Policies using it, including the set of their respective egress nodes.

The ingress SR Policy is responsible for initiating traffic steering into the DAG and is associated with a color value representing service intent. The junction segment, on the other hand,
is a DAG forwarding construct implemented as an SR Policy with a BSID.

To support global optimization with make-before-break (MBB) operations across a set of ingress SR Policies, the color value used by Junction Segments SHOULD differ from the color values of the ingress SR Policies
that are consumers of the DAG. This distinction allows ingress SR Policies to independently manage service steering while enabling consistent forwarding behavior across the DAG.

The controller is responsible for maintaining and tracking the association and semantic meaning of color values across all Junction Segments that participate in the DAG.

Accordingly, an implementation SHOULD treat ingress SR Policies and Junction Segments as decoupled constructs, each with their own versioning and lifecycle.

Since SR-based networks support specifying multiple egress interfaces using adjacency-SID sets and Node SIDs,
the Junction Segment MAY include a SID list entry that identifies multiple outgoing interfaces. In addition to egress interfaces,
{{I-D.draft-kompella-teas-mpte}} describes signaling ingress interfaces. The use of a Junction Segment omits the need for per-interface
ingress signaling as a single Binding Segment attached to an SR Policy is used. All upstream originated traffic sent to a downstream Junction Node
uses the same, single Junction Segment value which is a Binding Segment.

# Operation

## General

A path computation request or tunnel delegation notification is sent to the controller, specifying one or more ingress and egress nodes, along with constraints or service level agreement policy information.
This request may originate from an ingress router in the network or be provisioned directly via an API to the controller.
This tunnel computation request or delegation pertains to an instance of an MPTE Tunnel achieved with the ingress SR Policy.

The controller computes the DAG for ingress and all egress endpoints to determine all Junction nodes in the DAG to be used for the tunnel.

The controller signals the Junction Segment(s) to all downstream nodes, starting with the penultimate egress hop node(s) and
working upwards toward the ingress nodes. The controller may perform deployments in parallel provided there is no interdependency of values between junction segments,
and the controller does not update the ingress until all deployments are deemed successful.

If a Junction Segment cannot be deployed to a node, the controller SHOULD re-attempt the deployment up to a configurably defined maximum number of retries.

Depending on local policy and implementation constraints, the controller MAY execute a full rollback of the configuration, or it MAY allow the
partial deployment to proceed, provided that the TE constraints continue to be satisfied.

Junction Segments are deployed as a unicast SR Policy with a single Candidate Path using protocols such as PCEP, BGP, or NETCONF.

A BSID MUST be explicitly requested or signaled to the Junction node for assignment. If the controller opts for local node assigned value, it MUST wait to signal
upstream Junction nodes about their Junction segments in their outgoing SID lists. The BSID value MAY be a constant value globally, if assigned by PCE/Controller,
or may be a different value on each Junction Node whether it’s assigned by the controller or the local Junction Node itself.

Each SR Policy contains one or more SID lists. With the exception of the penultimate node, these SID lists
include at least two segment identifiers: one SID for forwarding to the downstream Junction node and one for the Junction Segment value (BSID) of the downstream node.
For directly connected neighbors, this may be the adjacency or node SID for the neighbor. For Junction nodes that are not directly connected,
additional SIDs MAY be used to steer the packet along an ECMP or non-ECMP path to the downstream Junction node.

Appendix A provides an example topology and resulting Junction segments.

## Tunnel with multiple ingress/egress

{{I-D.draft-kompella-teas-mpte}} specifies that an MPTE Tunnel could have multiple ingress and/or multiple egress nodes.
For controller-initiated tunnels, the intended ingress and egress node(s) can be provided to the controller based on implementation specific methods. These may be signaled to the network as
multiple tunnels to support multi-ingress scenarios. Each tunnel can use a null Endpoint value to support multi-egress.

MPTE SR Policies that are originated or defined by network devices are typically limited to a single ingress and a single egress endpoint unless protocols such as PCEP or NETCONF are extended
to encode additional intended destination node(s) for controller-based path computation.

As mentioned in Section 4.2, the DAG tunnel may be re-used by multiple ingress SR Policies. This mechanism is used to support achieving multiple ingress nodes originated from the network,
by way of the controller binding and attaching the ingress SR Policies to a pre-existing DAG sharing the same intent and endpoints.

## Load-balancing

When a packet with the BSID assigned to the Junction Segment is received at its Junction Node,
the node performs weighted non-equal cost flow-based forwarding across all egress SID lists associated with the Junction Segment.

As per {{RFC9256}} Section 2.11, The fraction of the flows associated with a given segment list is w/Sw, where w is the weight of the segment
list and Sw is the sum of the weights of the segment lists. To exclude forwarding via a specific egress interface
while preserving the forwarding structure, a weight value of zero is assigned to the corresponding segment list.

## Protection

As described in {{I-D.draft-kompella-teas-mpte}}, as there are multiple egress interfaces (SID Lists), the loss of one interface link does not result in traffic drops,
as long as one egress interface (SID List) remains although congestion may occur. For example, in the topology shown in Appendix A, Node C can
tolerate the loss of up to 3 egress links and traffic will still forward (potentially with congestion).

If a Junction Node experiences a failure of all egress links (including any protection for those links), it will initially
blackhole traffic until upstream nodes are notified by the controller to remove the failed Junction node from the DAG.

To reduce the risk of outages caused by single link failure, the controller MAY optimize DAG deployment by assigning
Junction Segments only to nodes with more than one egress segment list. In other words, if a node in the DAG has only one egress interface,
it functions solely as a transit node for an upstream Junction Segment and does not receive a Junction Segment itself.
For example, in the topology shown in Appendix A, Node E is omitted from receiving a Junction Segment as it only has 1 egress link.

Link protection from an upstream Junction node to its downstream Junction nodes can be achieved using existing Topology Independent Loop-Free Alternative (TI-LFA) {{RFC9855}} mechanisms,
applied per egress SID List on each Junction Segment. Since the top SID(s) in each SID List identify the path to the
next downstream Junction node, TI-LFA is applicable. Effectively, TI-LFA is used to protect traffic between Junction Segments along a path
within the DAG and is not intended to protect traffic directed toward the DAG’s egress nodes or the entire DAG.

Local computation for node protection on an upstream Junction node is not feasible, because it lacks visibility into the
DAG beyond the immediate downstream Junction node as it only knows the next Junction Segment.
A controller MAY be used to precompute backup SR Paths and signal these backup SID Lists to the upstream Junction segments.

## Hierarchy

The use of Junction Segments to achieve a DAG can be used in hierarchical organization and sharing of DAGs between end-to-end tunnels.

For instance, in a multi-area or multi-instance topology, one or more shared DAGs may be created per area, connecting the border ingress node(s) and egress node(s).
These DAGs can then be stitched together for use in an end-to-end SR tunnel.

Figure 1 is an example of two independent SR Policy Tunnels from Headend A and Headend B terminating on Egress X.
An instance of a DAG with MID 100 can be configured between area border routers (ABR)
ABR-1 and the Egress X node. ABR-1 Junction Segment which ingresses the DAG has a Binding Segment attached, for
example BSID-100. Therefore, the SID list on Headend A and Headend B SID lists would contain
the SR Path (ex: Node-SID-ABR-1) to reach ABR 1 followed by BSID-100. The instruction set from each Headend to ABR 1
could also be another instance of a DAG, either for independent use or shared.

~~~
          -------------------------------------------------------------
          |                             |                             |
          |                             |                             |
     +-------+                          |             ---\            |
     |       |--\     Path 1            |         ---/  | ---\        |
     |Headend|   ----\                  |     ---/   -----\   --\     |
     |   A   |         ----\         +-----+-/   ---/      ---\  --+------+
     +-------+              ----\    |     |  --/   -------|   --  |      |
          |                      --- | ABR |  -------------------  |Egress|
          |                      --- |  1  |        |DAG-100       |  X   |
     +-------+              ----/    +-----+ --\    |        ----- +------+
     |       |        -----/            |  -\   ----\-------/     --  |
     |Headend|   ----/                  |    --\     ---       --/    |
     |   B   |--/     Path 2            |       -\   |      --/       |
     +-------+                          |         --\    --/          |
          |                             |            ---/             |
          |                             |                             |
          -------------------------------------------------------------
                   Domain/Area 1                  Domain/Area 2

                             Figure 1 : Reusable DAG 100
~~~


## Directly connected Junction nodes

As described in the Operational section, all transit nodes in a DAG MAY be signaled with a Junction Segment.
Alternatively depending on the topological graph and TE requirements, two Junction segments may be interconnected via an
SR Path, with a SID list, where the SR Path itself may
correspond to an ECMP or TE Path.


# Optimization

## Local optimization

Local optimization refers to updates applied to a single Junction node that can be performed without requiring
coordinated updates across multiple nodes in the DAG. These operations are considered safe when they do not
introduce forwarding loops or cause reachability interruptions within the DAG.

Examples of local optimizations include:

- Manipulating the weight distribution of the outgoing SID lists
- Replacing an existing SID list with a more optimal one (e.g. using a new adjacency SID or updated SR path to the next Junction node).
- Adding a new SID list toward an egress node that is not itself a Junction node.
- Removing a SID list, provided that at least one other outgoing SID list remains active.
- Adding a new SID list to a Junction node that connects to a new downstream sub-graph which is not yet connected to the DAG structure.

While these operations are scoped to a single node, their correct application may still require awareness of the surrounding DAG structure
to avoid introducing loops or reachability issues. For instance, the addition of a new SID list may require confirmation that the new sub-graph
is not already part of the DAG, or that it connects loop free.

Local optimizations may also be chained in sequence across multiple nodes. When each step maintains loop free and reachable properties,
such a chain of updates is still considered local in nature.

For example:

- Cleaning up an upstream disconnected sub-graph following the removal of an upstream SID list (BSID is no longer used)

It is important to note that local optimization differs from global optimization in that it does not require versioning or re-signaling of the entire DAG structure.

Since a Junction Segment is realized via an SR Policy, the controller leverages existing protocol mechanisms to update a Junction segment forwarding
instructions, as the Junction Segment itself is represented as a candidate path within the SR Policy. When updating an existing candidate path,
the binding SID MUST NOT change, as doing so would cause traffic using that binding SID to drop until upstream consumers are updated.
Such a change should instead be treated as a global optimization, not a local one.

Multiple candidate paths could be used to support more comprehensive local optimizations. However, caution is required in how the MPTED version
is encoded and how version increments are signaled, since candidate paths are signaled within the context of the SR Policy.

If the same color value is reused and a new candidate path is deployed, the new candidate path will not be installed in the forwarding plane
until the existing one is removed, as only one candidate path may be active per SR Policy. This operation may not be hitless, depending on hardware implementation.

Alternatively, if a different color value is used, the new candidate path may be allowed to go active. However, it will still fail to do so since it should be using
the same binding SID as the existing candidate path. This results in a collision, rendering the new path ineligible, and may also violate
protocol constraints that prohibit such configurations.

Therefore, it is RECOMMENDED that local optimization be performed by updating the SID list of the existing candidate path, rather than introducing new candidate paths.

Appendix B provides examples of local optimization.

## Global optimization

Global optimization of a DAG requires coordinated updates across all participating Junction Nodes.
This process follows a make-before-break model, where a new version of the DAG is deployed alongside the existing one.
The ingress SR Policy (or policies) is the last element to be updated.

Since the Color field of an SR Policy indicates DAG membership, and is managed by the controller, global optimization
with make-before-break considerations necessitates deploying new Junction segments. This, in turn, requires the instantiation
of new SR Policies with unique Color values. These new SR Policies are deployed to all Junction Nodes participating in the
updated DAG instance and MUST be associated with distinct Binding SID values from those of the existing instance.

To minimize version churn and both color and label consumption, it is RECOMMENDED that controllers limit the number of
concurrently deployed DAG instances for the same tunnel. Under normal conditions, only a single active
DAG version should exist. During transitions, at most two versions (the current and the new one) should be present. Color and label values may be reused once globally released.

The Binding SID associated with a DAG instance MUST remain constant during local optimizations, but MUST change during global optimizations to avoid the risk of routing loops. Therefore, in the example
above, if a Junction Node is participating in DAG #1, version #1 continues to use one binding SID value and version #2 is using a distinct binding SID value on each Junction node.

Once all Junction segments are deployed to all Junction nodes, the ingress node is updated with new SID lists referencing the Binding SIDs of the new Junction segments downstream.

The Color field on ingress nodes is critical for traffic steering and therefore MUST remain constant during global optimizations. In contrast, Color values used on Junction Nodes
MAY be reused or encoded to reflect DAG membership or instance mappings. Similarly, the binding SID on the ingress SR Policy also remains constant.

The controller tracks the Color values and their corresponding usage for DAG ID and version. For example:

- Color value 500 – DAG #1 – Version #1
- Color value 501 – DAG #2 – Version #1
- Color value 502 – DAG #1 – Version #2

When performing global changes, controllers SHOULD ensure that all Junction Nodes are modified in a coordinated and consistent manner before activating the new DAG on the ingress.
If the update process fails partially, the controller SHOULD roll back the new deployment and retain the existing
stable DAG version and its Junction Segments to avoid inconsistencies in forwarding behavior.

Appendix C provides an example of global optimization.

# Manageability Considerations

## Control of Function through Configuration and Policy

This document describes using the existing SR Policy construct with a color value representing the DAG MID and version.
Since SR Policy color value is originally intended for ingress traffic
steering on matched routers, a deployment MUST allocate a color range which will be used for MPTE tunnels.

## Information and Data Models, e.g., MIB modules

TODO

## Liveness Detection and Monitoring

Liveness detection for an MPTE SR deployment has two scopes: the underlying IP links and the DAG tunnel itself.

### Topology

Seamless BFD (S-BFD) {{RFC7880}} is RECOMMENDED on each IP link to quickly detect link failures in the underlying topology. When a link
goes down, any tunnel S-BFD session that traverses the link will also fail (see the next section). The controller tracks IGP topology changes
and takes corrective action when necessary.

### DAG Tunnel

Liveness detection of the DAG tunnel must verify end-to-end forwarding across all Junction Nodes and the weighted forwarding performed
by each Junction Segment.

A single S-BFD session from the ingress headend to the DAG endpoint(s) follows one hashed path through the DAG. It cannot verify that every egress SID list on every Junction Node is functional.

Therefore, it is RECOMMENDED that each egress SID list of each Junction Segment run its own S-BFD session to the next downstream node.
On session failure:

- The affected SID list is marked inactive and removed from the weighted ECMP set.
- If a Junction Segment has no active egress SID lists remaining, the Junction Segment becomes inactive, which in turn fails any
  upstream S-BFD sessions that use it.

This per-Junction, per-SID-list approach monitors every forwarding segment of the DAG independently of hashing.

As a result, an egress node may receive many S-BFD packets, one per SID list, per Junction Node, per DAG.

## Verifying Correct Operation

This section discusses OAM for SR-MPLS ({{RFC8287}}). Applicability to SRv6 ({{RFC9259}}) is to be covered in a future revision.

As with S-BFD in Section 10.3.2, a single OAM probe from the ingress follows only one hashed path through the DAG and does not exercise every egress SID list on every Junction Node.

A controller can collect ping and traceroute results on a per-Junction, per-SID-list basis and aggregate them into a holistic view of the DAG. Each probe targets the path encoded in that SID list, terminating at the next downstream Junction Node or DAG egress node, so that every forwarding segment is verified independently of upstream hashing. The controller may request results on-demand or receive them periodically. The frequency in which to run OAM is deployment specific.

Because these tests run independently and not necessarily at the same instant, the aggregated view may contain timing skew between individual measurements. Multiple samples per test are RECOMMENDED.

## Requirements on Other Protocols and Functional Components

There are currently no new requirements on other protocols or functional components.

## Impact on Network Operation

TODO

# Security Considerations

TODO

# IANA Considerations

None at this time

--- back

# DAG Example

~~~
                  +------+       +------+
                  |      |  10   |      |
              --- |  B   | ----- |  E   |--
             /    |      |       |      |  \
         10 /     +------+       +------+   \ 10
           /         | 10                    \
          /          |                        \
    +------+      +------+       +------+     +------+
    |      |      |      |       |      |     |      |
    |  A   | -----|  C   | ----- |  F   |---- |  H   |
    |      |  10  |      |   5   |      | 10  |      |
    +------+      +------+\     /+------+     +------+
          \         | |5    \ /    5|(Pruned)  /
           \        | |     / \     |         /
         20 \    +------+ /    \+------+     / 10
             \   |      |   5   |      |    /
              ---|  D   | ----- |  G   |---
                 |      |       |      |
                 +------+       +------+

          Figure 2 : Example Topology to apply DAG

~~~

Figure 2 above presents a sample topology for one ingress and one egress MPTE Tunnel established as an SR Policy. The MPTE tunnel is from A to H.
The figure presents the bi-directional link weights for an arbitrary metric (IGP, TE, Delay etc).

Note there is a mesh between C, D, F and G of weight 5 per link. Pruned represents an excluded link due to TE constraints (for example, such as insufficient bandwidth).
In the below, the terminology {U,V} represents a unidirectional link between U and V. The terminology Adj-SID-XY represents an adjacency SID from X to Y. Node-SID-X represents the node SID path to X.

An MPTE DAG is computed to contain nodes B,C,D,E,F,G with the links {G,F} / {F,G} pruned.

The below Junction Segments are deployed to the network realized with an SR Policy with a single Candidate Path containing multiple segment lists. Note the following:

- When the DAG is computed, loops cannot exist. Therefore, in the above topology the links in direction {C,B} and {C,D} and {C,F} and {C,G} are chosen in the DAG.

- The path along {B,E,H} can be represented by a single Node SID. A Junction on node E is not required. Junction Segment B may also be omitted (as described in Section 5.3) but is utilized here for example purposes.

- Junction F is described below, but an optimization could be performed to exclude Junction F as it only has one egress link (see Section 5.3).

- In practice the Binding Segments MAY all be the same value. This example describes different BSID values for readability.

- Weights of each egress SID list is also currently omitted.

- The color on the ingress SR Policy still provides intent steering for ingress traffic, therefore SHOULD be unique to the color value of the Junction segments
comprising the DAG to permit global make-before-break behavior.

~~~

Junction Segment B:
  Color: 100
  BSID: BSID-B
  SID List 1: [Node-SID-H]

Junction Segment F:
  Color: 100
  BSID: BSID-F
  SID List 1: [Adj-SID-FH]

Junction Segment G:
  Color: 100
  BSID: BSID-G
  SID List 1: [Adj-SID-GH]

Junction Segment C:
  Color: 100
  BSID: BSID-C
  SID List 1: [Adj-SID-CB, BSID-B]
  SID List 2: [Adj-SID-CF, BSID-F]
  SID List 3: [Adj-SID-CG, BSID-G]
  SID List 4: [Node-SID-D, BSID-D]

Junction Segment D:
  Color: 100
  BSID: BSID-D
  SID List 1: [Adj-SID-DF, BSID-F]
  SID List 2: [Adj-SID-DG, BSID-G]

Then lastly, at ingress the SR Policy transport tunnel is configured with the following:

Ingress SR Policy (Using Junction Segments):
  Color: 50
  Candidate Path 1:
    SID List 1: [Adj-SID-AB, BSID-B]
    SID List 2: [Adj-SID-AC, BSID-C]
    SID List 3: [Adj-SID-AD, BSID-D]

~~~

In comparison, if the above DAG was encoded at ingress then the following individual segment lists could be used to represent the above DAG.
Note, some of the below could be compressed with a Node SID(s) but listed with adjacency for explicit example.

~~~
Ingress SR Policy (Ingress only):
  Color: 50
  Candidate Path 1:
    SID List 1: [Adj-SID-AC, Adj-SID-CF, Adj-SID-FH]
    SID List 2: [Adj-SID-AC, Node-SID-D, Adj-SID-DG, Adj-SID-GH]
    SID List 3: [Adj-SID-AC, Node-SID-D, Adj-SID-DF, Adj-SID-FH]
    SID List 4: [Adj-SID-AC, Adj-SID-CF, Adj-SID-CG, Adj-SID-GH]
    SID List 5: [Adj-SID-AC, Adj-SID-CF, Adj-SID-BE, Adj-SID-EH]
    SID List 6: [Adj-SID-AB, Node-SID-H]
    SID List 7: [Adj-SID-AD, Adj-SID-DG, Adj-SID-GH]
    SID List 8: [Adj-SID-AD, Adj-SID-DF, Adj-SID-FH]
~~~

# Local Optimization Example

The following examples use the topology and Junction Segments from the worked example in Appendix A.

Example 1 – Simple SID list update

- The Junction Segment B is updated. The SID list [Node-SID-H] is replaced with SID List [Adj-SID-BE, Adj-SID-EH]

Example 2 – Chained Local Optimization

- The ingress SR Policy SID list is updated to remove { SID List 1: [Adj-SID-AB, BSID-B] }. Only SID List 2 and SID List 3 remain in the SR Policy candidate path
- The Junction Segment B is no longer used.
- A delete is sent to remove Junction Segment B

# Global Optimization Example

~~~

    +------+      +------+       +------+     +------+
    |      |      |      |       |      |     |      |
    |  Z   | -----|  Y   | ----- |  X   |---- |  W   |
    |      |      |      |       |      |     |      |
    +------+      +------+       +------+     +------+
          \          |     \       |          /
           \         |      \      |         /
            \    +------+     \ +------+     /
             \   |      |       |      |    /
              ---|  V   | ----- |  U   |---
                 |      |       |      |
                 +------+       +------+

          Figure 3 : Global MBB Example Topology

~~~

**Step 0 - Initial State**

Figure 3 gives an example topology containing a DAG which is to undergo optimization.
Node Z is the ingress Node with an SR Policy color 1000 representing a service intent. Node W is the egress node.
Color 2000 is the DAG MID and version tracked by the controller. In the existing state of the DAG, traffic flows from west to east, and north to south on nodes Y and X. The diagonal link
between Y and U is currently not used.

~~~
Junction Segment Y1:
  Color: 2000
  BSID: BSID-Y1
  SID List 1: [Adj-SID-YX, BSID-X1]
  SID List 2: [Adj-SID-YV, Adj-SID-VU]

Junction Segment X1:
  Color: 2000
  BSID: BSID-X1
  SID List 1: [Adj-SID-XW]
  SID List 2: [Adj-SID-XU, Adj-SID-UW]

Ingress SR Policy:
  Color: 1000
  Candidate Path 1:
  SID List 1: [Adj-SID-ZY, BSID-Y1]
  SID List 2: [Adj-SID-ZV, Adj-SID-VU, Adj-SID-UW]
~~~

**Step 1 - Deploy new Junction Segments**

An optimization calculation occurs requiring the DAG to change, the traffic will now flow from south to north instead of north to south, and the diagonal link between Y and U will be used.

The controller assigns and tracks color 2001 for the new DAG. The following new Junction segments are deployed in the following order. The BSID values are either assigned by the controller,
or requested by the node depending on the protocol in use. Note that after this deployment node Y has two Junction segments installed on it, both of which are active.

~~~
CREATE Junction Segment U2:
  Color: 2001
  BSID: BSID-U2
  SID List 1: [Adj-SID-UX, ADJ-SID-XW]
  SID List 2: [Adj-SID-UW]

CREATE Junction Segment Y2:
  Color: 2001
  BSID: BSID-Y2
  SID List 1: [Adj-SID-YX, ADJ-SID-XW]
  SID List 2: [Adj-SID-YU, BSID-U2]

CREATE Junction Segment V2:
  Color: 2001
  BSID: BSID-V2
  SID List 1: [Adj-SID-VY, BSID-Y2]
  SID List 2: [Adj-SID-VU, BSID-U2]
~~~

**Step 2 - Update ingress nodes**

The existing SR Policy candidate path is updated to use the new DAG. Note, the color and any associated binding SID values remain.

~~~
UPDATE Ingress SR Policy:
  Color: 1000
  Candidate Path 1:
    SID List 1: [Adj-SID-ZY, BSID-Y2]
    SID List 2: [Adj-SID-ZV, BSID-V2]
~~~

**Step 3 - Delete the no longer used Junction segments**

~~~
DELETE Junction Segment Y1:
  Color: 2000
  BSID: BSID-Y1

DELETE Junction Segment X1:
  Color: 2000
  BSID: BSID-X1
~~~

# Acknowledgments
{:numbered="false"}

Thanks to Matthew Bocci, Richard "Footer" Foote, and Martin Vigoureux for discussions on this document.
