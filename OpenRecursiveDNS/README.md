## Mitigating an Open Recursive DNS vulnerabity from the LDAP/DNS server

thuner-srv1 hosts the LDAP and BIND/DNS services for user credential and hostname resolution, respectively. Let's walk through the networking infrastructure of thuner-srv1 server by trigger the Linux kernel to display every network interface with `ip addr`, together with its Layer 2 (Ethernet) and Layer 3 (IP) configuration. Each interface represents a connection between the operating system and some network:

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
```

```bash
[root@thuner-srv1 ~]# cat /var/named/cpp.ualberta.ca.zone
$TTL 86400
@   IN  SOA  thuner-srv1.cpp.ualberta.ca. root.thuner-srv1.cpp.ualberta.ca. (
        2025112701 ; Serial (bump this)
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@   IN  NS   thuner-srv1.cpp.ualberta.ca.

thuner-srv1  IN  A  142.244.83.XX
thuner-srv2  IN  A  142.244.83.XX
thuner-gw5   IN  A  142.244.83.XX
thuner-gw07  IN  A  142.244.83.XX
thuner-gw08  IN  A  142.244.93.XX
thuner-gw13  IN  A  129.128.122.XX
...
thuner-gw15  IN  A  129.128.122.XX
thuner-gw51  IN  A  129.128.122.XX

srv001       IN  A  192.168.1.XX
srv002       IN  A  192.168.1.XX
thuner003    IN  A  192.168.1.XX
thuner004    IN  A  192.168.1.XX
...
thuner051    IN  A  192.168.1.XX
```

```bash
[root@thuner-srv1 ~]# ps aux | egrep 'named|bind|dnsmasq|unbound'
rpc       1178  0.0  0.0  69256  1460 ?        Ss   Nov27   0:00 /sbin/rpcbind -w
root      1859  0.0  0.0 341440  3028 ?Ssl  Nov27   0:01 /usr/sbin/ypbind -n
named     1874  1.1  0.6 2488432 404056 ?      Ssl  Nov27 136:26 /usr/sbin/named -u named -c /etc/named.conf
nobody    2266  0.0  0.0  60372  1144 ?        S    Nov27   0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root      2267  0.0  0.0  60344   416 ?        S    Nov27   0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root     32448  0.0  0.0 112948   996 pts/0    S+   14:46   0:00 grep -E --color=auto named|bind|dnsmasq|unbound
```

```bash
[root@thuner-srv1 ~]# ss -tulnp | grep :53
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("dnsmasq",pid=2266,fd=5))
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("named",pid=1874,fd=586),("named",pid=1874,fd=585),("named",pid=1874,fd=584),("named",pid=1874,fd=583),("named",pid=1874,fd=582),("named",pid=1874,fd=581),("named",pid=1874,fd=580),("named",pid=1874,fd=579),("named",pid=1874,fd=578),("named",pid=1874,fd=577),("named",pid=1874,fd=576),("named",pid=1874,fd=575),("named",pid=1874,fd=574),("named",pid=1874,fd=573),("named",pid=1874,fd=572))
udp    UNCONN     0      0      192.168.1.203:53                    *:*                   users:(("named",pid=1874,fd=571),("named",pid=1874,fd=570),("named",pid=1874,fd=569),("named",pid=1874,fd=568),("named",pid=1874,fd=567),("named",pid=1874,fd=566),("named",pid=1874,fd=565),("named",pid=1874,fd=564),("named",pid=1874,fd=563),("named",pid=1874,fd=562),("named",pid=1874,fd=561),("named",pid=1874,fd=560),("named",pid=1874,fd=559),("named",pid=1874,fd=558),("named",pid=1874,fd=557))
udp    UNCONN     0      0      142.244.83.9:53                    *:*                   users:(("named",pid=1874,fd=556),("named",pid=1874,fd=555),("named",pid=1874,fd=554),("named",pid=1874,fd=553),("named",pid=1874,fd=552),("named",pid=1874,fd=551),("named",pid=1874,fd=550),("named",pid=1874,fd=549),("named",pid=1874,fd=548),("named",pid=1874,fd=547),("named",pid=1874,fd=546),("named",pid=1874,fd=545),("named",pid=1874,fd=544),("named",pid=1874,fd=543),("named",pid=1874,fd=542))
udp    UNCONN     0      0      127.0.0.1:53                    *:*                   users:(("named",pid=1874,fd=541),("named",pid=1874,fd=540),("named",pid=1874,fd=539),("named",pid=1874,fd=538),("named",pid=1874,fd=537),("named",pid=1874,fd=536),("named",pid=1874,fd=535),("named",pid=1874,fd=534),("named",pid=1874,fd=533),("named",pid=1874,fd=532),("named",pid=1874,fd=531),("named",pid=1874,fd=530),("named",pid=1874,fd=529),("named",pid=1874,fd=528),("named",pid=1874,fd=527))
udp    UNCONN     0      0      [::]:53                 [::]:*                   users:(("named",pid=1874,fd=526),("named",pid=1874,fd=525),("named",pid=1874,fd=524),("named",pid=1874,fd=523),("named",pid=1874,fd=522),("named",pid=1874,fd=521),("named",pid=1874,fd=520),("named",pid=1874,fd=519),("named",pid=1874,fd=518),("named",pid=1874,fd=517),("named",pid=1874,fd=516),("named",pid=1874,fd=515),("named",pid=1874,fd=514),("named",pid=1874,fd=513),("named",pid=1874,fd=512))
tcp    LISTEN     0      10     192.168.122.1:53                    *:*                   users:(("named",pid=1874,fd=27))
tcp    LISTEN     0      10     192.168.1.203:53                    *:*                   users:(("named",pid=1874,fd=24))
tcp    LISTEN     0      10     142.244.83.9:53                    *:*                   users:(("named",pid=1874,fd=23))
tcp    LISTEN     0      10     127.0.0.1:53                    *:*                   users:(("named",pid=1874,fd=22))
tcp    LISTEN     0      10     [::]:53                 [::]:*                   users:(("named",pid=1874,fd=21))
```

```bash
[root@thuner-srv1 ~]# grep -E "options|recursion|allow-query|allow-recursion|listen-on" -n /etc/named.conf
12:options {
13:        listen-on port 53 { any; };
14:	listen-on-v6 port 53 { any; };
15:        allow-query { any; };
24:	 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
26:	   recursion. 
33:	recursion yes;
```

```bash
[root@thuner-srv1 ~]# firewall-cmd --permanent --remove-port=53/udp
success
[root@thuner-srv1 ~]# firewall-cmd --permanent --remove-port=53/tcp
success
[root@thuner-srv1 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="53" protocol="udp" accept'
success
[root@thuner-srv1 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="53" protocol="tcp" accept'
success
[root@thuner-srv1 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.0/24" port port="53" protocol="udp" accept'
success
[root@thuner-srv1 ~]# firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.0/24" port port="53" protocol="tcp" accept'
success
[root@thuner-srv1 ~]# firewall-cmd --reload
success
[root@thuner-srv1 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: em1 em2
  sources: 
  services: dhcpv6-client ldap ssh
  ports: 
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	rule family="ipv4" source address="192.168.1.0/24" port port="53" protocol="udp" accept
	rule family="ipv4" source address="192.168.1.0/24" port port="53" protocol="tcp" accept
	rule family="ipv4" source address="192.168.122.0/24" port port="53" protocol="udp" accept
	rule family="ipv4" source address="192.168.122.0/24" port port="53" protocol="tcp" accept
```


```bash
[root@thuner-srv1 ~]# emacs /etc/named.conf
[root@thuner-srv1 ~]# named-checkconf /etc/named.conf
[root@thuner-srv1 ~]# systemctl restart named
```

```bash
[root@thuner-srv1 ~]# ss -tulnp | grep :53
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("named",pid=1289,fd=556),("named",pid=1289,fd=555),("named",pid=1289,fd=554),("named",pid=1289,fd=553),("named",pid=1289,fd=552),("named",pid=1289,fd=551),("named",pid=1289,fd=550),("named",pid=1289,fd=549),("named",pid=1289,fd=548),("named",pid=1289,fd=547),("named",pid=1289,fd=546),("named",pid=1289,fd=545),("named",pid=1289,fd=544),("named",pid=1289,fd=543),("named",pid=1289,fd=542))
udp    UNCONN     0      0      192.168.1.203:53                    *:*                   users:(("named",pid=1289,fd=541),("named",pid=1289,fd=540),("named",pid=1289,fd=539),("named",pid=1289,fd=538),("named",pid=1289,fd=537),("named",pid=1289,fd=536),("named",pid=1289,fd=535),("named",pid=1289,fd=534),("named",pid=1289,fd=533),("named",pid=1289,fd=532),("named",pid=1289,fd=531),("named",pid=1289,fd=530),("named",pid=1289,fd=529),("named",pid=1289,fd=528),("named",pid=1289,fd=527))
udp    UNCONN     0      0      127.0.0.1:53                    *:*                   users:(("named",pid=1289,fd=526),("named",pid=1289,fd=525),("named",pid=1289,fd=524),("named",pid=1289,fd=523),("named",pid=1289,fd=522),("named",pid=1289,fd=521),("named",pid=1289,fd=520),("named",pid=1289,fd=519),("named",pid=1289,fd=518),("named",pid=1289,fd=517),("named",pid=1289,fd=516),("named",pid=1289,fd=515),("named",pid=1289,fd=514),("named",pid=1289,fd=513),("named",pid=1289,fd=512))
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("dnsmasq",pid=2266,fd=5))
tcp    LISTEN     0      10     192.168.122.1:53                    *:*                   users:(("named",pid=1289,fd=23))
tcp    LISTEN     0      10     192.168.1.203:53                    *:*                   users:(("named",pid=1289,fd=22))
tcp    LISTEN     0      10     127.0.0.1:53                    *:*                   users:(("named",pid=1289,fd=21))
```