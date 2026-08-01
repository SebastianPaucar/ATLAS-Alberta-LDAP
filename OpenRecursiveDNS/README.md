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