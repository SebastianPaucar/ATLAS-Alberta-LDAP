# Mitigating a Open Recursive DNS vulnerability

`thuner-srv1` runs BIND (`named`) as the DNS service for the cluster infrastructure. The zone file `/var/named/cpp.ualberta.ca.zone` contains the authoritative DNS records that map cluster hostnames to their IP addresses. Clients querying names within `cpp.ualberta.ca` receive answers directly from this database. The complete picture is:

```bash
                    Compute Nodes
                (thuner001, thuner002, ...)
                           |
                    DNS query ("Where is thuner003?")
                           |
                           ▼
                 +----------------------+
                 | named (BIND daemon)  |
                 +----------------------+
                           |
             loads authoritative records from
                           |
                           ▼
          /var/named/cpp.ualberta.ca.zone
                           |
                           ▼
     thuner003 -> 192.168.1.XX
     thuner-srv1 -> 142.244.83.XX
     thuner-srv2 -> 142.244.83.XX
                           |
                           ▼
        BIND returns the requested IP address
                           |
                           ▼
      Linux routes packets using the network interfaces
      (em1 or em2), routers, and switches.
```

The networking infrastructure (interfaces, switches, routers, routing tables) is responsible for delivering packets. The BIND infrastructure is responsible only for translating hostnames into IP addresses. The zone file is the authoritative database that allows BIND to perform that translation for the `cpp.ualberta.ca` domain. BIND loads `/var/named/cpp.ualberta.ca.zone` at startup and uses it to answer DNS queries for the `cpp.ualberta.ca` zone. The zone file is therefore input to BIND. It is not an executable program. It is simply a text file containing resource records that BIND loads into memory.

## `cpp.ualberta.ca.zone` describes the network

When `named` starts with `systemctl start named`, systemd executes `/usr/sbin/named -u named -c /etc/named.conf` and the daemon reads `/etc/named.conf`, and declares the server as the authoritative server for the zone `cpp.ualberta.ca`, able to load the DNS records from `/var/named/cpp.ualberta.ca.zone`. After reading the file, BIND stores the records in memory. Clients never access the zone file directly; every query is answered from BIND's in-memory database. The startup sequence is therefore:

```bash
systemd
     │
     ▼
named
     │
     ▼
reads /etc/named.conf
     │
     ▼
opens cpp.ualberta.ca.zone
     │
     ▼
loads records into memory
     │
     ▼
starts answering DNS queries
```

That zone file contains:

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

The zone file is a mapping between hostnames and IP addresses. This means the hostanem `thunerXYZ.cpp.ualberta.ca` has the IPv4 address `192.168.1.XX`. Similarly, `thuner-XYZ.cpp.ualberta.ca` has IPv4 address `142.244.83.XX`. The zone file is simply the authoritative hostname database. The zone file is what allows software to use hostnames instead of IP addresses. Earlier, the topology looked like:

```bash
                    University Network
                     142.244.83.0/24
                            |
                     +----------------+
                     | thuner-srv1    |
                     | em1            |
                     |142.244.83.Xem1 |
                     |                |
                     | em2            |
                     |192.168.1.Xem2  |
                     +----------------+
                            |
                    Internal Cluster LAN
                      192.168.1.0/24
                            |
          ------------------------------------
          |        |        |         |
      thuner001 thuner002 thuner003 ...
```

So instead of connecting to `129.128.122.XX`, a user can connect to its correspondig `thuner-gwXX.cpp.ualberta.ca`. BIND performs the translation.

> The network itself never uses the zone file. Routers forward packets using IP addresses. Switches forward Ethernet frames using MAC addresses. Only DNS applications read the zone file.

The topology exists independently of DNS, DNS simply documents it. For example:

```bash
thuner001 IN A 192.168.1.XX
thuner002 IN A 192.168.1.XX
...
thuner051 IN A 192.168.1.XX
```

Immediately tells there is an internal subnet `192.168.1.0/24` and numerous cluster nodes belong to it. Therefore, the DNS database reflects the network architecture. It does not create the network architecture.

## DNS software and network exposure 

Let's identify the DNS-related services executing on `thuner-srv1`:

```bash
[root@thuner-srv1 ~]# ps aux | egrep 'named|bind|dnsmasq|unbound'
rpc       1178  0.0  0.0  69256  1460 ?        Ss   Nov27   0:00 /sbin/rpcbind -w
root      1859  0.0  0.0 341440  3028 ?Ssl  Nov27   0:01 /usr/sbin/ypbind -n
named     1874  1.1  0.6 2488432 404056 ?      Ssl  Nov27 136:26 /usr/sbin/named -u named -c /etc/named.conf
nobody    2266  0.0  0.0  60372  1144 ?        S    Nov27   0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root      2267  0.0  0.0  60344   416 ?        S    Nov27   0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root     32448  0.0  0.0 112948   996 pts/0    S+   14:46   0:00 grep -E --color=auto named|bind|dnsmasq|unbound
```

The output shows that the primary DNS server is BIND (`named`), running with the configuration file `/etc/named.conf`. This process is responsible for providing DNS services for the cluster, including serving the authoritative `cpp.ualberta.ca` zone. The output also shows a separate `dnsmasq` instance associated with `libvirt` (`/var/lib/libvirt/dnsmasq/default.conf`), which is used exclusively for the virtual bridge (`virbr0`) and is unrelated to the cluster DNS infrastructure.

> **Finding.** The host runs BIND as its primary DNS server
> **Security implication.** None by itself. This command only identifies the software responsible for DNS services.

Now let's determine the network exposure of the DNS service:

```bash
[root@thuner-srv1 ~]# ss -tulnp | grep :53
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("dnsmasq",pid=2266,fd=5))
udp    UNCONN     0      0      192.168.122.1:53                    *:*                   users:(("named",pid=1874,fd=586),("named",pid=1874,fd=585),("named",pid=1874,fd=584),("named",pid=1874,fd=583),("named",pid=1874,fd=582),("named",pid=1874,fd=581),("named",pid=1874,fd=580),("named",pid=1874,fd=579),("named",pid=1874,fd=578),("named",pid=1874,fd=577),("named",pid=1874,fd=576),("named",pid=1874,fd=575),("named",pid=1874,fd=574),("named",pid=1874,fd=573),("named",pid=1874,fd=572))
udp    UNCONN     0      0      192.168.1.XX:53                    *:*                   users:(("named",pid=1874,fd=571),("named",pid=1874,fd=570),("named",pid=1874,fd=569),("named",pid=1874,fd=568),("named",pid=1874,fd=567),("named",pid=1874,fd=566),("named",pid=1874,fd=565),("named",pid=1874,fd=564),("named",pid=1874,fd=563),("named",pid=1874,fd=562),("named",pid=1874,fd=561),("named",pid=1874,fd=560),("named",pid=1874,fd=559),("named",pid=1874,fd=558),("named",pid=1874,fd=557))
udp    UNCONN     0      0      142.244.83.XX:53                    *:*                   users:(("named",pid=1874,fd=556),("named",pid=1874,fd=555),("named",pid=1874,fd=554),("named",pid=1874,fd=553),("named",pid=1874,fd=552),("named",pid=1874,fd=551),("named",pid=1874,fd=550),("named",pid=1874,fd=549),("named",pid=1874,fd=548),("named",pid=1874,fd=547),("named",pid=1874,fd=546),("named",pid=1874,fd=545),("named",pid=1874,fd=544),("named",pid=1874,fd=543),("named",pid=1874,fd=542))
udp    UNCONN     0      0      127.0.0.1:53                    *:*                   users:(("named",pid=1874,fd=541),("named",pid=1874,fd=540),("named",pid=1874,fd=539),("named",pid=1874,fd=538),("named",pid=1874,fd=537),("named",pid=1874,fd=536),("named",pid=1874,fd=535),("named",pid=1874,fd=534),("named",pid=1874,fd=533),("named",pid=1874,fd=532),("named",pid=1874,fd=531),("named",pid=1874,fd=530),("named",pid=1874,fd=529),("named",pid=1874,fd=528),("named",pid=1874,fd=527))
udp    UNCONN     0      0      [::]:53                 [::]:*                   users:(("named",pid=1874,fd=526),("named",pid=1874,fd=525),("named",pid=1874,fd=524),("named",pid=1874,fd=523),("named",pid=1874,fd=522),("named",pid=1874,fd=521),("named",pid=1874,fd=520),("named",pid=1874,fd=519),("named",pid=1874,fd=518),("named",pid=1874,fd=517),("named",pid=1874,fd=516),("named",pid=1874,fd=515),("named",pid=1874,fd=514),("named",pid=1874,fd=513),("named",pid=1874,fd=512))
tcp    LISTEN     0      10     192.168.122.1:53                    *:*                   users:(("named",pid=1874,fd=27))
tcp    LISTEN     0      10     192.168.1.XX:53                    *:*                   users:(("named",pid=1874,fd=24))
tcp    LISTEN     0      10     142.244.83.XX:53                    *:*                   users:(("named",pid=1874,fd=23))
tcp    LISTEN     0      10     127.0.0.1:53                    *:*                   users:(("named",pid=1874,fd=22))
tcp    LISTEN     0      10     [::]:53                 [::]:*                   users:(("named",pid=1874,fd=21))
```

All sockets listening on the DNS service port (`53`) are enumerated. The output shows that `named` accepts both UDP and TCP DNS requests on:

* `127.0.0.1`: local host only.
* `192.168.1.XX`: private cluster network.
* `142.244.83.X`: University of Alberta public network.
* `[::]`: all IPv6 interfaces.

This demonstrates that the DNS service is exposed simultaneously on the loopback interface, the internal cluster network, the institutional network, and all IPv6 interfaces.

> **Finding.** The DNS server is reachable from multiple network segments rather than being restricted to a single interface.
> **Security implication.** Listening on multiple interfaces is common for infrastructure DNS servers and is not itself a vulnerability. However, it increases the potential attack surface because any DNS functionality enabled by the server becomes accessible from every interface on which it listens.

Finally, let's examine the DNS security configuration:

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

The configuration directives governing client access and recursive resolution. The directives shown indicate that:

* BIND listens on every configured IPv4 and IPv6 interface.
* DNS queries are accepted from any client.
* Recursive resolution is enabled.
* The configuration does not show an `allow-recursion` access-control list.

> **Finding.** The server is configured as both an authoritative DNS server and a recursive resolver.

> **Security implication.** Running an authoritative DNS server with recursion enabled increases the attack surface. If recursive queries are not explicitly restricted through `allow-recursion`, `allow-query-cache`, views, or firewall rules, the server may operate as an open recursive resolver. Open recursive resolvers are recognized security risks because they can be abused for DNS amplification and reflection attacks and provide recursive resolution services to unauthorized clients.

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