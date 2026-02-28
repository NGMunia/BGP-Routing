# BGP
This lab configures BGP for both IPv4 and IPv6

![Topology](/Topology.png)


## AS123:

### Route Reflection

AS-123 is a non-transit AS.
R1 acts as a route reflector.
in BGP, there's a rule:  
**A route learned from iBGP peer cannot be advertised to another iBGP peer**
This is to prevent routing loops inside an AS.
A route reflector allows iBGP routers to pair with it and not with each other instead of full mesh.

---
### Update-source Loopback
By default, BGP uses the outgoing physical interface's IP to source its TCP session. This is a problem because if that interface goes down, the BGP session drops — even if another path to the neighbor exists.

Using a loopback as the update-source makes the session much more stable, since loopbacks never go down unless the router itself does.


This must be done on both routers, otherwise the TCP session won't form — one side will be sourcing from the loopback IP, and the other won't recognize it as the expected neighbor.

The loopback IPs must be reachable between the two routers. In an iBGP setup this is typically handled by your IGP (OSPF, EIGRP, IS-IS):

---
### Next-hop-self
When a router learns a route via eBGP, the next-hop attribute is set to the IP of the external peer that advertised it.

When that route is then passed to iBGP peers, the next-hop stays unchanged — pointing to an external IP that internal routers may have no idea how to reach.

Next-hop-self command the router: "when advertising routes to this iBGP neighbor, replace the next-hop with your own IP" — typically your loopback.
Now internal routers have a reachable next-hop they can actually forward traffic to.


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


LOCAL PREFERENCE:
* From AS123, traffic destined to AS150 should exit R2
* From AS123, traffic destined to AS140 should exit R3.

AS-PREPENDING:
* Traffic destined to 123.123.123.0/24 and 2001:123:1::/48 
  prefix should enter through R2
* Traffic destined to 124.124.124.0/24 and 2001:123:2::/48 
  prefix should enter through R3

* AS123 should be a non-Transit AS
* Branch routers are configured with static routes to test connectivity/reachability
