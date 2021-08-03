# Multi-WAN Upstream Routing

This AlgoVPN has been modified to support multiple upstream Wireguard WAN uplinks. The key points to know are:

* Default routes for the incoming downstream clients are handled in the `upstream` routing table, defined in `/etc/iproute2/rt_tables`.

* The AlgoVPN cloud instance servces as the 'server' to all upstream WAN and downstream client instances. It will accept traffic from clients and pass it on to the upstream clients for routing to the Internet.

* Wireguard configuration has been augmented to automatically add and prune routes and routing rules based on what upstream clients are connected.

* Routes are implemented in a balanced manner. By default this should work per flow for multiple upstream WAN connections, but may present some asyncronous routing scenarios due to NAT.

## Routing Diagram

![Firewall Illustration](/docs/images/multi-wan.png)

### Network Address Translation

This version of the modified AlgoVPN provides iptables MASQUERADE on all egress interfaces and tunnels. This allows for maximal compatiblility for upstream tunnel nodes, as well as a fallback position for the downstream VPN clients should the upstream tunnels disconnect. This preserves the user-observable function of the VPNs without breakage while the upstreams are in flux. 

### VPN Relay

The default configuration of AlgoVPN is to relay the incoming client traffic directly out the default interface of the host. This modified version allows the addition of two upstream WAN tunnels to relay the client traffic on to those upstreams. 

A properly configured AlgoVPN hop point should look similar to the following :

```
root@vpn-relay:~# ip route show 
default via 172.26.0.1 dev eth0 proto dhcp src 172.26.12.83 metric 100
172.26.0.0/20 dev eth0 proto kernel scope link src 172.26.12.83
172.26.0.1 dev eth0 proto dhcp scope link src 172.26.12.83 metric 100
192.168.233.0/24 dev wg0 proto kernel scope link src 192.168.233.1
192.168.234.0/24 dev wg1 proto kernel scope link src 192.168.234.1

root@vpn-relay:~# ip route show table upstream
default dev wg1 scope link

root@vpn-relay:~# ip rule show
0:	from all lookup local
32764:	from 192.168.233.0/24 lookup upstream
32766:	from all lookup main
32767:	from all lookup default
```

This represents a single upstream WAN. Multiple WANs are possible and will be configured in a balanced setup by intiating client flow. **However, there may be some untested asyncronous routing issues to work out.** Wireguard `PostUp` and `PostDown` features are set to manage getting incoming Wireguard clients to the upstream interfaces. If there is an upstream Wireguard tunnel active, it will be added to the `upstream` routingtable as a default route and a routing rule added to direct incoming client traffic to the WAN interface. Should all of these upstream tunnels disconnect, the default behavior is to fall back to the default routing position for the host, allowing client traffic to leave the cloud AlgoVPN instance directly via the `main` routing table defaults. 
