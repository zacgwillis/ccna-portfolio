# Single-Area OSPF Overview
This lab goes into the basics of OSPF, demonstrating a single-area design with config/verify & the purpose of OSPF

## Objective
* Describe what OSPF is, why it's needed, & how it works
* Configure & verify a single-area OSPF topology using the default broadcast network type
* Briefly go over certain misconfigurations that either allow adjacencies, but prevent routes from being learned, or prevent adjacencies altogether

## Topology
![topology-ospf-single-area.png](./images/topology-ospf.png)


## OSPF Overview
what it is:  
* OSPF is a link-state routing protocol that enables routers to build a complete map of the network topology. It does this by exchanging LSAs, which describe a routers directly connected links, costs & other details. As the most commonly deployed IGP, OSPF uses these LSAs to create an identical LSDB on every router in the same area. The Dijkstra's SPF algorithm used on these LSAs calculates the most efficient path  

why it's needed:  
* Routers can learn routes either statically (time-consuming, difficult to manage at scale) or dynamically via a routing protocol. OSPF provides dynamic route discovery, automatic adaption to topology changes (like link failures) & scales efficiently in larger networks. Ideal for environments where manual management is impractical & fast convergence is essential  

how it works:  
* OSPF routers form adjacencies w/ neighbors using Hello packets, then exchange database summaries & request missing information. They flood LSAs inside LSU packets to share topology details across the area, ensuring all routers build the same LSDB. Once sync'd, each router runs the SPF algorithm on its own LSDB to compute the shortest path to every destination & installs those routes into its routing table; ex: a router advertises its own local subnets as part of its Type 1 Router LSAs to other routers in the same area

With broadcast network types, OSPF elects one Designated Router (DR) & one Backup Designated Router (BDR) to minimize LSA flooding & adjacency count. Non-DR/BDR routers (DROTHERs) form full adjacencies only with the DR/BDR & stay in 2WAY state with other DRothers

## Initial Setup
Begin by assigning appropriate IP addresses for each interface/link, bring the interface up, & test basic connectivity

## Procedure
You can either configure OSPF w/ the traditional method using **network** commands, or the modern way w/ inteface config, which uses per-interface commands to specify which OSPF process ID & area to use

### Traditional Method:
OSPF/Router Config, using **network** commands  
```
R1(config)# router ospf 1  
R1(config-router)# network 10.1.0.0 0.0.15.255 area 0    // Matches using wildcard masks
R1(config-router)# router-id 1.1.1.1    // Optional RID config
```
### Interface Config:
```
R1(config)# int f3/0
R1(config-if)# ip ospf 1 area 0
R1(config-if)# int g0/0
R1(config-if)# ip ospf 1 area 0
```

## Core OSPF Verification Commands
* There are only two commands that will distinguish whether OSPF was configured using the traditional method versus the interface method  
```
show ip protocols        // Routing for Networks vs. Routing on Interfaces Configured Explicitly
show ip ospf int g0/0    // Attached via Network Statement vs. Attached via Interface Enable
```

* Use these commands from any router to confirm OSPF operation:  

`show running-config` Verifies configuration commands used to enable OSPF  

`show ip ospf interface [{g0/0 | brief}]` OSPF-enabled interfaces w/ details like area membership, network type, state (DR/BDR/DRother), cost, timers, priority, hello/dead intervals  

`show ip ospf neighbor [g0/0]` OSPF neighbor adjacencies including neighboring RID, priority, state (FULL/DR), dead time, & connecting interface/IP  

`show ip ospf database` Lists contents of the OSPF LSDB, including Router LSAs (Type 1) & Network LSAs (Type 2) that build the single-area topology map  

`show ip ospf rib` Displays OSPF local Routing Information Base, showing routes computed by SPF before they are installed in the global routing table  

`show ip route [ospf]` Displays OSPF-learned routes installed in the routing table, including AD, metric, next-hop, & outgoing interface  

![verify.png](./images/verify3.png)


## Troubleshooting
### Cases where neighbors form but routes are missing/incomplete
The misconfigurations below will result in neighbors being formed, but routes not being learned  

**Network Type Mismatch**  
`R4(config-if)# ip ospf network point-to-point`  
(ex: broadcast > point-to-point) Neighbors may form & reach a FULL state, but DR/BDR election fails or LSAs aren't properly advertised. SPF can't calculate routes correctly, resulting in missing or incomplete routes in the routing table. Check that Network Types match on both ends of the link  

**MTU Mismatch**  
`R4(config-if)# ip mtu 1400`  
Fails to complete LSDB exchange. Continually reaches EXSTART then DOWN. Verify w/ `show interfaces` MTU values  

**OSPF Zero Priority**  
`R4(config-if)# ip ospf priority 0`  
OSPF priority value 0 refuses DR/BDR role; results in 2WAY/DROTHER state

### Cases where no neighbor relationships form at all
These prevent any adjacency
* Area ID Mismatch
* Hello/Dead Timer Mismatch
* Duplicate Router ID
* Passive Interface Misconfiguration

## Notes
**DR/BDR Election Tiebreakers**  
Highest priority value wins (default 1; configurable); if tied, highest RID wins. RID configuration priorities are as follows:  
1) Manual RID Config  
2) Highest numerical loopback IP address  
3) Highest numerical active physical IP address

Point-to-point OSPF network types do not utilize a DR/BDR election  

**Passive Interfaces**  
Suppresses OSPF Hellos on unnecessary interfaces, like user-facing ports. Will continue to advertise those links/subnets as part of the OSPF topology, but will not send OSPF messages onto them  

**OSPF Default Routes**  
Advertises default routes w/ `default-information originate`; useful for internet edge  

**Hello/Dead Timers**  
Defaults: Hello every 10sec, Dead at 40sec on broadcast. Tuning for faster convergence possible but increases CPU usage

## Key Takeaways
* OSPF is a scalable link-state IGP that builds a full topology map via LSAs & SPF for fast, efficient routing
* Keeping it to a single area (usually Area 0) makes setup & troubleshooting straightforward. It's all you need for smaller or medium-sized networks, & multi-area designs build on this foundation
* You can enable OSPF the traditional way with network statements under the router process or the cleaner modern way directly on interfaces; both work great, but interface config is clearer & easier to read
* A few key `show` commands tell the whole story: check neighbor states for FULL adjacencies, make sure the LSDB looks consistent across routers, & confirm OSPF routes show up in the routing table
* Misconfigurations usually break things in two ways: either no neighbors form at all (area mismatch, timers off, duplicate RID, passive where it shouldn't be) or neighbors come up but routes stay missing (network type mismatch, MTU issues, zero priority messing with DR/BDR). Always start with `show ip ospf neighbor` & `show ip ospf interface` & compare to spot the problem fast











