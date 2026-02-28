# BGP
This lab configures BGP for both IPv4 and IPv6

![Topology](/Topology.png)

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
