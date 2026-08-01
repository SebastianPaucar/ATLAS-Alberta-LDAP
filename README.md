## The `thuner-srv1` server

`thuner-srv1` hosts the LDAP and BIND/DNS services for user credential and hostname resolution, respectively. Let's walk through the networking infrastructure of thuner-srv1 server by trigger the Linux kernel to display every network interface with `ip addr`, together with its Layer 2 (Ethernet) and Layer 3 (IP) configuration. Each interface represents a connection between the operating system and some network:

```bash
[root@thuner-srv1 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:cb:29:ed:38:eb brd ff:ff:ff:ff:ff:ff
    inet 142.244.83.9/24 brd 142.244.83.255 scope global noprefixroute em1
       valid_lft forever preferred_lft forever
    inet6 fe80::bacb:29ff:feed:38eb/64 scope link 
       valid_lft forever preferred_lft forever
3: em2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:cb:29:ed:38:ec brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.203/24 brd 192.168.1.255 scope global noprefixroute em2
       valid_lft forever preferred_lft forever
    inet6 fe80::bacb:29ff:feed:38ec/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:9a:25:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
5: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:9a:25:b6 brd ff:ff:ff:ff:ff:ff
```

There are five (5) interfaces:

* **Loopback interface** (`lo`)`: Every Linux machine has this interface. `inet 127.0.0.1/8` is the IPv4 loopback address. This interface never leaves the computer. Whenever a program connects to `127.0.0.1/8`, the packets never reach a physical network card. They travel entirely inside the kernel. `inet6 ::1/128` is the IPv6 equivalent one. Every Linux system has `::1` even if IPv6 is completely disconnected. All this  means programs on the machine may communicate with themselves using IPv6 or IPv4.
  * `ldapsearch -H ldap://127.0.0.1` never generates Ethernet traffic. The LDAP server receives the request internally.
  * `dig @127.0.0.1 google.com` queries the local BIND server without touching the network.
* **First physical Ethernet adapter** (`em1`): `inet 142.244.83.Xem1/24` contains both host's IP address (`142.244.83.X`) and the network that interface belongs to (`142.244.83.0/24`). `/24` describes the network boundary. This is a public IPv4 address (the address belongs to a globally routable address block allocated to the University of Alberta).
  * Because the network is directly attached, Linux assumes that every other machine in the same subnet (`142.244.83.0/24`) can be reached without involving a router.
  * `brd 142.244.83.255` is the broadcast address. Broadcast packets sent there are delivered to every machine on that subnet
  * The interface is `UP` (Linux has enabled it) and `LOWER_UP` (the Ethernet controller detects an active electrical link). The cable is plugged in and the switch port is active.
  * Since `inet6 fe80::bacb:29ff:feed:38eb/64` begins with `fe80::` (link-local IPv6), it is not a globally routable IPv6 address. Every IPv6-enabled Ethernet interface automatically generates one (Linux creates it automatically). Link-local addresses exist only on the local Ethernet segment, routers never forward them. `fe80::bacb:29ff:feed:38eb/64` cannot reach another subnet (it only communicates with neighboring devices).
  * `em1` tells how `thuner-srv` fits into the university's larger network. 
*  Interface `em2`: `inet 192.168.1.Xem2/24` means that the second Ethernet interface is connected to the network `192.168.1.0/24`. This is not a public network. The entire `192.168.0.0/16` address block is reserved by RFC 1918 for private networks. Routers on the public Internet do not forward packets whose destination is a `192.168.X.X` address. `em2` physically connects `thuner-srv1` to a different Layer 2 network than `em1`. 
  * `em2` connects to the laboratory's internal LAN. It `tells how `thuner-srv` fits into the the cluster itself.
  * `192.168.1.0/24` network is the internal cluster network. The compute nodes, switches, and several servers all communicate on this private subnet.
* `libvirt` interface (`virbr0`): OS virtualization installation automatically creates `virbr0` via `libvirt`. `inet 192.168.122.1/24` is the default NAT network for virtual machines.  `NO-CARRIER` and `state DOWN` tells us no virtual machines are currently using it. 

> The public interface (`em1`) connects the infrastructure server to the university network. Systems elsewhere on campus can reach services such as SSH, DNS, or LDAP through this interface, subject to firewall rules.

> The private interface (`em2`) connects the same server directly to the cluster's internal Ethernet. The compute nodes authenticate against LDAP, perform DNS lookups, and communicate with infrastructure services without sending that traffic through the university network.

## `thuner-srv1` networking infrastructure

The toplogy diagram is as follows:

```bash
                         University Network
                     Subnet: 142.244.83.0/24
                                 |
                    Default Gateway / Router
                                 |
                    =========================
                    Ethernet / VLAN / Switch
                    =========================
                                 |
                          em1 (142.244.83.Xem1)
                      +-----------------------+
                      |      thuner-srv1      |
                      |                       |
                      | LDAP                 |
                      | BIND DNS             |
                      |                       |
                      | em2 (192.168.1.Xem2)  |
                      +-----------------------+
                                 |
                    =========================
                    Internal Cluster Switch
                    =========================
                                 |
               -----------------------------------------
               |         |          |          |        |
          thuner001  thuner002  thuner003   ...  thunerN
         192.168.1.X1  .X2        .X3             .XN
```

The server functions as an infrastructure node that:

* Provides authoritative DNS for `cpp.ualberta.ca`.
* Runs LDAP for authentication,
* Is  connected to both a public-facing network and a private internal LAN
* Has virtualization support installed through `libvirt`.

## BIND/DNS services

As mentioned, `thuner-srv1` is the cluster's DNS authority since it hosts the systemd `named` service.

```bash
[root@thuner-srv1 ~]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2025-11-27 14:31:12 MST; 1 weeks 1 days ago
  Process: 17702 ExecReload=/bin/sh -c /usr/sbin/rndc reload > /dev/null 2>&1 || /bin/kill -HUP $MAINPID (code=exited, status=0/SUCCESS)
  Process: 1873 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 1627 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 1874 (named)
    Tasks: 35
   CGroup: /system.slice/named.service
           └─1874 /usr/sbin/named -u named -c /etc/named.conf

Dec 05 14:43:03 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'copilot-proxy.githubusercontent.com/A/IN': 2600:9000:5307:4b00::1#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2620:4d:4000:...9:0:3#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2a00:edc0:6259:7:9::2#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2600:9000:530...00::1#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2600:9000:530...00::1#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2600:9000:530...00::1#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2620:4d:4000:...9:0:1#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'telemetry.individual.githubcopilot.com/A/IN': 2a00:edc0:6259:7:9::4#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'copilot-telemetry-service.githubusercontent.com/A/IN': 2a00...:1::2#53
Dec 05 14:45:19 thuner-srv1.cpp.ualberta.ca named[1874]: network unreachable resolving 'copilot-telemetry-service.githubusercontent.com/A/IN': 2600...00::1#53
```

Although IPv6 is enabled in the Linux kernel (which is why link-local addresses exist), the network has not been deployed with globally routable IPv6 connectivity. Consequently, when BIND attempts to contact authoritative DNS servers that publish only IPv6 transport addresses, the kernel cannot determine a route and returns `ENETUNREACH` (`"Network is unreachable"`). BIND then falls back to querying an IPv4 address for the same DNS server, which is why DNS resolution generally continues to work despite the log messages.