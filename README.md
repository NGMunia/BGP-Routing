# BGP
This lab configures BGP for both IPv4 and IPv6

![Topology](/Topology.png)


## AS123:

### OSPFv3

OSFv3 is used as the underlay transport network in the iBGP network.
Unique local addreses are used on Loopback in order to advertise the loopback interface on the OSPF network.


Loopback interfaces are used in iBGP peering.

On R1:

```bash
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ipv6 address FD00::1/128
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
!
interface Ethernet0/0
 ip address 10.0.0.1 255.255.255.252
 duplex auto
 ipv6 address FE80::1 link-local
 ospfv3 network point-to-point
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
!
interface Ethernet0/1
 ip address 10.0.0.5 255.255.255.252
 duplex auto
 ipv6 address FE80::1 link-local
 ospfv3 network point-to-point
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
 ```
---

### Route Reflection

AS-123 is a non-transit AS.
R1 acts as a route reflector.
in BGP, there's a rule:  
**A route learned from iBGP peer cannot be advertised to another iBGP peer**
This is to prevent routing loops inside an AS.
A route reflector allows iBGP routers to pair with it and not with each other instead of full mesh.

```bash
address-family ipv4
  neighbor 2.2.2.2 route-reflector-client
```

---
### Update-source Loopback
By default, BGP uses the outgoing physical interface's IP to source its TCP session. This is a problem because if that interface goes down, the BGP session drops — even if another path to the neighbor exists.

Using a loopback as the update-source makes the session much more stable, since loopbacks never go down unless the router itself does.


This must be done on both routers, otherwise the TCP session won't form — one side will be sourcing from the loopback IP, and the other won't recognize it as the expected neighbor.

The loopback IPs must be reachable between the two routers. In an iBGP setup this is typically handled by your IGP (OSPF, EIGRP, IS-IS):

```bash
router bgp 123
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 123
 neighbor 2.2.2.2 update-source Loopback0
```

---
### Next-hop-self
When a router learns a route via eBGP, the next-hop attribute is set to the IP of the external peer that advertised it.

When that route is then passed to iBGP peers, the next-hop stays unchanged — pointing to an external IP that internal routers may have no idea how to reach.

Next-hop-self command the router: "when advertising routes to this iBGP neighbor, replace the next-hop with your own IP" — typically your loopback.
Now internal routers have a reachable next-hop they can actually forward traffic to.

```bash
address-family ipv6
  neighbor FD00::1 activate
  neighbor FD00::1 next-hop-self
```

Below is a Sample configuration:

```bash
router bgp 123
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 123
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 3.3.3.3 remote-as 123
 neighbor 3.3.3.3 update-source Loopback0
 neighbor FD00::2 remote-as 123
 neighbor FD00::2 update-source Loopback0
 neighbor FD00::3 remote-as 123
 neighbor FD00::3 update-source Loopback0
 !
 address-family ipv4
  network 123.123.123.0 mask 255.255.255.0
  network 124.124.124.0 mask 255.255.255.0
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 route-reflector-client
  neighbor 3.3.3.3 activate
  no neighbor FD00::2 activate
  no neighbor FD00::3 activate
 exit-address-family
 !
 address-family ipv6
  network 2001:123:1::/48
  network 2001:124:1::/48
  neighbor FD00::2 activate
  neighbor FD00::2 route-reflector-client
  neighbor FD00::3 activate
 exit-address-family
```

---
### Local Preference and AS prepending (Asymetric load balancing)

**Local Preference** is a BGP attribute used to decide the outbound path from your AS.
It answers this question:
“If I have multiple ways to reach a destination, which exit point should my AS prefer?”


**AS Prepending** on the other hand, is a BGP (Border Gateway Protocol) technique used to influence how other autonomous systems (ASes) choose the best path to reach your network.
You basically add your own AS number multiple times to the AS path of the route you advertise.

The longer the AS path, the less attractive the route appears to other networks.
we can use AS-Prepending for asymmetric load balancing:

Example use case of local preference and AS prepending in  Aysmetric load balancing:

* Outbound traffic from the data center should prefer R2 as their egress router (local prefence)

* Inbound traffic should ingress via R3 (use of AS-prepending {out})

*Configuring Local Preference on R2*


```bash
route-map LOCAL_PREF_MAP permit 10
 set local-preference 400

R2(config)#router bgp 123
R2(config-router)#add
R2(config-router)#address-family ipv6
R2(config-router-af)#neighbor 2001:100:1:A::1 route-map LOCAL_PREF_MAP in
R2(config-router-af)#neighbor 2001:180:1:A::1 route-map LOCAL_PREF_MAP in
R2(config-router-af)#end

R2#clear ip bgp * soft

```
---
*Verifying*


From R1 we can see egress traffic uses R2 as its next-hop:


```bash
R1#sh bgp ipv6 unicast
! Output omitted for brevity

     Network          Next Hop            Metric LocPrf Weight Path
 *>   2001:123:1::/48  ::                       0         32768 i
 *>   2001:124:1::/48  ::                       0         32768 i
 *>i  2001:150:150:1::/64
                       FD00::2                  0    400      0 180 150 i
 *>i  2001:160:160:160::/64
                       FD00::2                  0    400      0 100 130 140 160 i

```
---
*Configuring AS-Prepending on R2:( since we want Ingress traffic to go via R3)*

```bash
ip as-path access-list 10 permit ^$
route-map AS_PREPEND_MAP permit 10
 match as-path 10
 set as-path prepend 123 123 123 123
 !

R2(config)#router bgp 123
R2(config-router)#address-family ipv6
R2(config-router-af)#neighbor 2001:180:1:A::1 route-map AS_PREPEND_MAP out
R2(config-router-af)#neighbor 2001:100:1:A::1 route-map AS_PREPEND_MAP out
R2(config-router-af)#end

R2#clear ip bgp * soft
```
---
*Verifying from AS180*

To reach 2001:123:1::/24 NLRI the traffic goes via ASes 150, 130, 110 qnd finally 123.

```bash
R180#sh bgp ipv6 unicast
BGP table version is 11, local router ID is 180.180.1.5
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
              t secondary path,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   2001:123:1::/48  2001:180:1:B::2                        0 150 130 110 123 i
 *                     2001:180:1:A::2                        0 123 123 123 123 123 i
 *>   2001:124:1::/48  2001:180:1:B::2                        0 150 130 110 123 i
 *                     2001:180:1:A::2                        0 123 123 123 123 123 i
 ```

That way asymmetric load balancing of traffic is achieved.


---
### AS123 as a Non-Transit Network:

A non-transit network in BGP is an Autonomous System (AS) that does not allow traffic to pass through it between two other external networks.

It advertises its own prefixes and receives routes from upstream providers — but it does NOT carry traffic from one provider to another.

In this topology, Non-transit functionality (Route filtering) is done using filter list:

```bash
address-family ipv6
  neighbor 2001:100:1:A::1 activate
  neighbor 2001:100:1:A::1 filter-list 10 out
  neighbor 2001:180:1:A::1 activate
  neighbor 2001:180:1:A::1 filter-list 10 out
  neighbor FD00::1 activate
  neighbor FD00::1 next-hop-self
 exit-address-family
!
ip forward-protocol nd
!
ip as-path access-list 10 permit ^$
```

---

*Testing reachability From Branch-3 to Branch-1:*

```bash
Branch-3#ping 150.150.150.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.150.150.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
Branch-3#
```
---
