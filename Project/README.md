# Построение сети ЦОД для связности двух сайтов по L2 и L3.

В данной работе рассмотрим обеспечение связности между двумя сайтами с использованием VXLAN EVPN, и используя технологии которые мы рассматривали в лабах, такие как ESI multihomming, l2vni, l3vni.

Для упрощения схемы мы не будем дублировать некоторые устройства, такие как SPINE или PE.

Фабрика строится на Juniper vQFX

---
## Схема:
![img_1.png](scheme.png)

На каждом сайте будут по 2 лифа, к которым будут подключены роутеры для выхода во вне, а также хосты. По одному хосту на каждом сайте мы вынесем в обзий L2 домен, для проверки L2 связности, и по одному хосту оставим каждый в своем влане, для проверки L3 связности. Маршрутизаторы будут подключаться с помощью агрегата. Так же на каждом сайте будет по одному бордер лифу, для подключения к суперспайну и обеспечения связности между сайтами. Так же добавим подключение клиентского роутера и хоста за ним для эмуляции спуска full view bgp по l2 с PE маршрутизатора.

P.S. В качестве PE маршрутизаторов будет использоваться образ Cisco IOL L3, он не умеет 802.3ad, поэтому между ним и лифами стоит свитч прослойка для эмуляции lacp, в конфиге роутера будет только один интерфейс.

## Адресация:

В качестве сетей для распределения выбраны сети:
| Название | Сеть | Vlan |
|----------|------|------|
| P2P FABRIC | 10.100.0.0/24 | - |
| Loopback | 10.200.0.0/24 | - |
| P2P WAN | 100.64.0.0/24 | 1000 |
| CLIENT_1 | 100.64.10.0/24 | 100 |
| CLIENT_2 | 100.64.20.0/24 | 200 |
| CLIENT_3 | 100.64.30.0/24 | 300 |
| CLIENT_4_WAN | 100.64.40.0/24 | 400 |
| CLIENT_4_LAN | 10.0.0.0/24 | - |

### Распределение адресации:

P2P FABRIC:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-A-SPINE-1      | xe-0/0/0      | SITE-A-LEAF-1       | xe-0/0/0      | 10.100.0.0/31         | 10.100.0.1/31  |
| SITE-A-SPINE-1     | xe-0/0/1      | SITE-A-LEAF-2       | xe-0/0/0      | 10.100.0.2/31         | 10.100.0.3/31  |
| SITE-A-SPINE-1     | xe-0/0/2      | SITE-A-BLEAF-1       | xe-0/0/0      | 10.100.0.4/31         | 10.100.0.5/31  |Add commentMore actions
| SITE-B-SPINE-1      | xe-0/0/0      | SITE-B-LEAF-1       | xe-0/0/0      | 10.100.0.6/31         | 10.100.0.7/31  |
| SITE-B-SPINE-1      | xe-0/0/1      | SITE-B-LEAF-2       | xe-0/0/0      | 10.100.0.8/31         | 10.100.0.9/31  |
| SITE-B-SPINE-1      | xe-0/0/2      | SITE-B-BLEAF-1       | xe-0/0/0      | 10.100.0.10/31         | 10.100.0.11/31  |
| SUPER-SPINE      | xe-0/0/0      | SITE-A-BLEAF-1       | xe-0/0/1      | 10.100.0.12/31         | 10.100.0.13/31  |
| SUPER-SPINE      | xe-0/0/1      | SITE-B-BLEAF-1       | xe-0/0/1      | 10.100.0.14/31         | 10.100.0.15/31  |
Add comment

Loopback:

| Устройство   | Интерфейс | IP-адрес       |
|--------------|-----------|----------------|
| SITE-A-SPINE-1       | Loopback0 | 10.200.0.1/32  |
| SITE-B-SPINE-1      | Loopback0 | 10.200.0.2/32  |
| SUPER-SPINE       | Loopback0 | 10.200.0.3/32  |
| SITE-A-LEAF-1       | Loopback0 | 10.200.0.4/32  |
| SITE-A-LEAF-2       | Loopback0 | 10.200.0.5/32  |
| SITE-A-BLEAF-1       | Loopback0 | 10.200.0.6/32  |
| SITE-B-LEAF-1       | Loopback0 | 10.200.0.7/32  |
| SITE-B-LEAF-2       | Loopback0 | 10.200.0.8/32  |
| SITE-B-BLEAF-1       | Loopback0 | 10.200.0.9/32  |

P2P WAN:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-A-PE-1      | e0/1(Po1.1000)      | SITE-A-LEAF-1       | xe-0/0/1(irb.1000)      | 100.64.0.1/24         | 100.64.0.2/24  |
| SITE-A-PE-1     | e0/2(Po1.1000)      | SITE-A-LEAF-2       | xe-0/0/1(irb.1000)      | 100.64.0.1/24         | 100.64.0.3/24  |
| SITE-B-PE-1     | e0/1(Po1.1000)       | SITE-B-LEAF-1       | xe-0/0/1(irb.1000)      | 100.64.0.4/24         | 100.64.0.5/24  |
| SITE-B-PE-1      | e0/2(Po1.1000)       | SITE-B-LEAF-2       | xe-0/0/1(irb.1000)      | 100.64.0.4/24         | 100.64.0.6/24  |

CLIENT_1:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-A-HOST_L2-1      | e0      | SITE-A-LEAF-1       | xe-0/0/2(irb.100)      | 100.64.10.10/24         | 100.64.10.1/24(anycast)  |
| SITE-B-HOST_L2-1     | e0      | SITE-B-LEAF-1       | xe-0/0/2(irb.100)      | 100.64.10.20/24         | 100.64.10.1/24(anycast)  |

CLIENT_2:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-A-HOST_L3-1      | e0      | SITE-A-LEAF-2       | xe-0/0/2(irb.200)      | 100.64.20.10/24         | 100.64.20.1/24  |

CLIENT_3:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-B-HOST_L3-1      | e0      | SITE-B-LEAF-2       | xe-0/0/2(irb.300)      | 100.64.30.10/24         | 100.64.30.1/24  |

CLIENT_4_WAN:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-A-PE-1      | Po1.400      | SITE-B-R-1       | e0/0      | 100.64.40.1/24         | 100.64.40.3/24  |
| SITE-B-PE-1    | Po1.400      | SITE-B-R-1       | e0/0      | 100.64.40.2/24         | 100.64.40.3/24  |

CLIENT_4_LAN:

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|
| SITE-B-R-1      | e0/1      | SITE-B-HOST-BGP-1       | e0      | 10.0.0.1/24         | 10.0.0.2/24  |

В качестве underlay будет использоваться IS-IS, поэтому распределим адреса

ISO Loopback:

| Устройство   | Интерфейс | ISO       |
|--------------|-----------|----------------|
| SITE-A-SPINE-1       | Loopback0 | 49.0001.0000.0000.0001.00  |
| SITE-B-SPINE-1      | Loopback0 | 49.0001.0000.0000.0002.00 |
| SUPER-SPINE       | Loopback0 | 49.0001.0000.0000.0003.00  |
| SITE-A-LEAF-1       | Loopback0 | 49.0001.0000.0000.0004.00  |
| SITE-A-LEAF-2       | Loopback0 | 49.0001.0000.0000.0005.00  |
| SITE-A-BLEAF-1       | Loopback0 | 49.0001.0000.0000.0006.00  |
| SITE-B-LEAF-1       | Loopback0 | 49.0001.0000.0000.0007.00  |
| SITE-B-LEAF-2       | Loopback0 | 49.0001.0000.0000.0008.00  |
| SITE-B-BLEAF-1       | Loopback0 | 49.0001.0000.0000.0009.00  |

### Распределение ASN

Для overlay для каждого сайта будет использоваться ibgp, но для связности между сайтами ebgp.

| Устройство   | AS |
|--------------|-----------|
| iBGP       | 65000 |
| SITE-A       | 65010 |
| SITE-B     | 65020 |
| SUPER-SPINE       | 65001 |
| SITE-A-PE-1       | 65101 |
| SITE-A-PE-1       | 65102 |
| CLIENT_BGP       | 65200 |

## Конфиги:

- [SITE-A-LEAF-1](SITE-A-LEAF-1.manifest)
- [SITE-A-LEAF-2](SITE-A-LEAF-2.manifest)
- [SITE-A-BLEAF-1](SITE-A-BLEAF-1.manifest)
- [SITE-A-SPINE-1](SITE-A-SPINE-1.manifest)
- [SITE-B-LEAF-1](SITE-B-LEAF-1.manifest)
- [SITE-B-LEAF-2](SITE-B-LEAF-2.manifest)
- [SITE-B-BLEAF-1](SITE-B-BLEAF-1.manifest)
- [SITE-B-SPINE-1](SITE-B-SPINE-1.manifest)
- [SUPER-SPINE](SUPER-SPINE.manifest)
- [SITE-A-PE-1](SITE-A-PE-1.manifest)
- [SITE-B-PE-1](SITE-B-PE-1.manifest)
- [SITE-B-R-1](SITE-B-R-1.manifest)

## Проверки связности:

Проверка выхода в интернет с хоста SITE-A-HOST_L2-1 и связность с хостом SITE-B-HOST_L2-1 по L2:
```
gns3@box:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN 
    link/ether 2a:c7:de:7f:d1:7e brd ff:ff:ff:ff:ff:ff
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip_vti0@NONE: <NOARP> mtu 1364 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 50:14:02:00:33:00 brd ff:ff:ff:ff:ff:ff
    inet 100.64.10.10/24 brd 100.64.10.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5214:2ff:fe00:3300/64 scope link 
       valid_lft forever preferred_lft forever
gns3@box:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=111 time=102.966 ms
64 bytes from 8.8.8.8: seq=1 ttl=111 time=178.607 ms
64 bytes from 8.8.8.8: seq=2 ttl=111 time=104.421 ms
64 bytes from 8.8.8.8: seq=3 ttl=111 time=112.692 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 102.966/124.671/178.607 ms
gns3@box:~$ ping 100.64.10.20
PING 100.64.10.20 (100.64.10.20): 56 data bytes
64 bytes from 100.64.10.20: seq=0 ttl=64 time=399.797 ms
64 bytes from 100.64.10.20: seq=1 ttl=64 time=295.632 ms
^C
--- 100.64.10.20 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 102.966/124.671/178.607 ms
gns3@box:~$
```
Проверка выхода в интернет с хоста SITE-B-HOST_L3-1 и проверкасвязности со всеми хостами по L3:

```
gns3@box:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN 
    link/ether 26:62:ba:bc:e2:f4 brd ff:ff:ff:ff:ff:ff
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip_vti0@NONE: <NOARP> mtu 1364 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 50:0f:58:00:36:00 brd ff:ff:ff:ff:ff:ff
    inet 100.64.30.10/24 brd 100.64.30.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::520f:58ff:fe00:3600/64 scope link 
       valid_lft forever preferred_lft forever
gns3@box:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=110 time=1108.304 ms
64 bytes from 8.8.8.8: seq=1 ttl=110 time=431.876 ms
64 bytes from 8.8.8.8: seq=2 ttl=110 time=474.513 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 3 packets received, 25% packet loss
round-trip min/avg/max = 431.876/671.564/1108.304 ms
gns3@box:~$ ping 100.64.10.10
PING 100.64.10.10 (100.64.10.10): 56 data bytes
64 bytes from 100.64.10.10: seq=0 ttl=62 time=371.662 ms
64 bytes from 100.64.10.10: seq=1 ttl=62 time=485.039 ms
64 bytes from 100.64.10.10: seq=2 ttl=62 time=474.585 ms
^C
--- 100.64.10.10 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 371.662/443.762/485.039 ms
gns3@box:~$ ping 100.64.10.20
PING 100.64.10.20 (100.64.10.20): 56 data bytes
64 bytes from 100.64.10.20: seq=0 ttl=62 time=280.365 ms
64 bytes from 100.64.10.20: seq=1 ttl=62 time=282.422 ms
64 bytes from 100.64.10.20: seq=2 ttl=62 time=273.313 ms
^C
--- 100.64.10.20 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 273.313/278.700/282.422 ms
gns3@box:~$ ping 100.64.20.10
PING 100.64.20.10 (100.64.20.10): 56 data bytes
64 bytes from 100.64.20.10: seq=0 ttl=62 time=361.094 ms
64 bytes from 100.64.20.10: seq=1 ttl=62 time=426.238 ms
64 bytes from 100.64.20.10: seq=2 ttl=62 time=172.305 ms
^C
--- 100.64.20.10 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 172.305/319.879/426.238 ms
gns3@box:~$ ping 100.64.40.3
PING 100.64.40.3 (100.64.40.3): 56 data bytes
64 bytes from 100.64.40.3: seq=0 ttl=252 time=1222.672 ms
64 bytes from 100.64.40.3: seq=1 ttl=252 time=848.290 ms
64 bytes from 100.64.40.3: seq=2 ttl=252 time=692.158 ms
^C
--- 100.64.40.3 ping statistics ---
4 packets transmitted, 3 packets received, 25% packet loss
round-trip min/avg/max = 692.158/921.040/1222.672 ms
gns3@box:~$
```
Проверка выхода в интернет с хоста SITE-B-HOST-BGP-1 и проверка связности со всеми хостами по L3

```
gns3@box:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN 
    link/ether 1e:9b:e5:1e:a6:ca brd ff:ff:ff:ff:ff:ff
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip_vti0@NONE: <NOARP> mtu 1364 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 50:43:2b:00:38:00 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5243:2bff:fe00:3800/64 scope link 
       valid_lft forever preferred_lft forever
gns3@box:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=111 time=167.121 ms
64 bytes from 8.8.8.8: seq=1 ttl=111 time=131.727 ms
^C
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 131.727/149.424/167.121 ms
gns3@box:~$ ping 100.64.10.10
PING 100.64.10.10 (100.64.10.10): 56 data bytes
64 bytes from 100.64.10.10: seq=0 ttl=61 time=625.169 ms
64 bytes from 100.64.10.10: seq=1 ttl=61 time=506.501 ms
^C
--- 100.64.10.10 ping statistics ---
3 packets transmitted, 2 packets received, 33% packet loss
round-trip min/avg/max = 506.501/565.835/625.169 ms
gns3@box:~$ ping 100.64.10.20
PING 100.64.10.20 (100.64.10.20): 56 data bytes
64 bytes from 100.64.10.20: seq=0 ttl=61 time=209.333 ms
64 bytes from 100.64.10.20: seq=1 ttl=61 time=273.944 ms
^C
--- 100.64.10.20 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 209.333/241.638/273.944 ms
gns3@box:~$ ping 100.64.20.10
PING 100.64.20.10 (100.64.20.10): 56 data bytes
64 bytes from 100.64.20.10: seq=0 ttl=61 time=233.142 ms
64 bytes from 100.64.20.10: seq=1 ttl=61 time=441.499 ms
^C
--- 100.64.20.10 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 233.142/337.320/441.499 ms
gns3@box:~$ ping 100.64.30.10
PING 100.64.30.10 (100.64.30.10): 56 data bytes
64 bytes from 100.64.30.10: seq=0 ttl=61 time=305.403 ms
64 bytes from 100.64.30.10: seq=1 ttl=61 time=303.691 ms
64 bytes from 100.64.30.10: seq=2 ttl=61 time=310.622 ms
^C
--- 100.64.30.10 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 303.691/306.572/310.622 ms
gns3@box:~$ 
```

## Просмотр данных с коммутаторов:

Команды используемые для анализа:

show bgp summary\
show ethernet-switching vxlan-tunnel-end-point remote\
show ethernet-switching vxlan-tunnel-end-point source\
show ethernet-switching table\
show evpn database\
show route

SITE-A-LEAF-1:
```
root@SITE-A-LEAF-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      40         40          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.1            65000       3428       3070       0       0    22:31:25 Establ
  bgp.evpn.0: 40/40/40/0
  default-switch.evpn.0: 28/28/28/0
  __default_evpn__.evpn.0: 3/3/3/0
  WAN.evpn.0: 9/9/9/0
100.64.0.1            65101        212        207       0       0     1:29:29 Establ
  WAN.inet.0: 1/1/1/0

{master:0}
root@SITE-A-LEAF-1> show ethernet-switching vxlan-tunnel-end-point remote 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.4       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.5       default-switch           572       vtep.32769   1770    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.7       default-switch           576       vtep.32771   1797    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    100           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.8       default-switch           557       vtep.32770   1793    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0               

{master:0}
root@SITE-A-LEAF-1> show ethernet-switching vxlan-tunnel-end-point source 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.4       lo0.0    0  
    L2-RTT                   Bridge Domain              VNID     MC-Group-IP
    default-switch           v100+100                   100      0.0.0.0        
    default-switch           v1000+1000                 1000     0.0.0.0        
    default-switch           v400+400                   400      0.0.0.0        

{master:0}
root@SITE-A-LEAF-1> show ethernet-switching table 

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 12 entries, 12 learned
Routing instance : default-switch
   Vlan                MAC                 MAC      Logical                SVLBNH/      Active
   name                address             flags    interface              VENH Index   source
   v100                50:0f:0b:00:33:00   D        xe-0/0/2.0           
   v100                50:14:02:00:33:00   D        xe-0/0/2.0           
   v100                50:2f:a7:00:33:00   D        xe-0/0/2.0           
   v100                50:36:be:00:35:00   D        vtep.32771                          10.200.0.7                    
   v1000               02:05:86:71:04:00   D        vtep.32770                          10.200.0.8                    
   v1000               02:05:86:71:3a:00   D        vtep.32769                          10.200.0.5                    
   v1000               02:05:86:71:cb:00   D        vtep.32771                          10.200.0.7                    
   v1000               aa:bb:cc:00:01:10   DL       ae10.0               
   v1000               aa:bb:cc:00:02:10   DR       esi.1742                            00:01:01:01:01:01:01:01:01:02 
   v400                aa:bb:cc:00:01:10   DL       ae10.0               
   v400                aa:bb:cc:00:02:10   DR       esi.1742                            00:01:01:01:01:01:01:01:01:02 
   v400                aa:bb:cc:00:03:00   D        vtep.32770                          10.200.0.8                    

{master:0}
root@SITE-A-LEAF-1> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        00:00:5e:00:53:01  irb.100                        Jun 06 09:41:45  100.64.10.1
     100        50:0f:0b:00:33:00  xe-0/0/2.0                     Jun 06 09:56:41
     100        50:14:02:00:33:00  xe-0/0/2.0                     Jun 06 10:00:56  100.64.10.10
     100        50:2f:a7:00:33:00  xe-0/0/2.0                     Jun 06 09:42:06
     100        50:36:be:00:35:00  10.200.0.7                     Jun 06 11:35:19  100.64.10.20
     400        aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 12:28:35  100.64.40.1
     400        aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:38:05
     400        aa:bb:cc:00:03:00  10.200.0.8                     Jun 06 12:26:59  100.64.40.3
     1000       02:05:86:71:04:00  10.200.0.8                     Jun 06 11:58:50  100.64.0.6
     1000       02:05:86:71:3a:00  10.200.0.5                     Jun 06 12:15:09  100.64.0.3
     1000       02:05:86:71:cb:00  10.200.0.7                     Jun 06 12:15:51  100.64.0.5
     1000       02:05:86:71:fc:00  irb.1000                       Jun 06 09:25:54  100.64.0.2
     1000       aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 13:22:53  100.64.0.1
     1000       aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:42:59  100.64.0.4

{master:0}
root@SITE-A-LEAF-1> show route 

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[Direct/0] 22:31:55
                    >  via xe-0/0/0.0
10.100.0.1/32      *[Local/0] 22:31:55
                       Local via xe-0/0/0.0
10.100.0.2/31      *[IS-IS/18] 22:20:22, metric 20
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.4/31      *[IS-IS/18] 22:20:22, metric 20
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 02:23:49, metric 60
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.8/31      *[IS-IS/18] 01:46:11, metric 60
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.10/31     *[IS-IS/18] 02:24:15, metric 50
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.12/31     *[IS-IS/18] 03:18:20, metric 30
                    >  to 10.100.0.0 via xe-0/0/0.0
10.100.0.14/31     *[IS-IS/18] 02:56:27, metric 40
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.1/32      *[IS-IS/18] 22:31:55, metric 10
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 02:23:49, metric 50
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.3/32      *[IS-IS/18] 02:56:27, metric 30
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.4/32      *[Direct/0] 22:59:16
                    >  via lo0.0
10.200.0.5/32      *[IS-IS/18] 22:15:44, metric 20
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.6/32      *[IS-IS/18] 03:18:20, metric 20
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.7/32      *[IS-IS/18] 02:23:39, metric 60
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.8/32      *[IS-IS/18] 01:46:04, metric 60
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.9/32      *[IS-IS/18] 02:24:15, metric 40
                    >  to 10.100.0.0 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 22:59:16
                    >  via em1.0
169.254.0.2/32     *[Local/0] 22:59:16
                       Local via em1.0

:vxlan.inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[Direct/0] 22:31:55
                    >  via xe-0/0/0.0
10.100.0.1/32      *[Local/0] 22:31:55
                       Local via xe-0/0/0.0
10.200.0.4/32      *[Direct/0] 22:38:55
                    >  via lo0.0
10.200.0.5/32      *[Static/1] 04:21:03, metric2 20
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.7/32      *[Static/1] 02:09:22, metric2 60
                    >  to 10.100.0.0 via xe-0/0/0.0
10.200.0.8/32      *[Static/1] 01:45:52, metric2 60
                    >  to 10.100.0.0 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 22:38:55
                    >  via em1.0
169.254.0.2/32     *[Local/0] 22:38:55
                       Local via em1.0

WAN.inet.0: 9 destinations, 16 routes (9 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[BGP/170] 01:29:42, localpref 100
                      AS path: 65101 I, validation-state: unverified
                    >  to 100.64.0.1 via irb.1000
                    [EVPN/170] 01:29:31
                    >  to 10.100.0.0 via xe-0/0/0.0
                    [EVPN/170] 00:01:38
                    >  to 10.100.0.0 via xe-0/0/0.0
                    [EVPN/170] 00:01:38
                    >  to 10.100.0.0 via xe-0/0/0.0
100.64.0.0/24      *[Direct/0] 01:29:45
                    >  via irb.1000
                    [EVPN/170] 01:29:33
                    >  to 10.100.0.0 via xe-0/0/0.0
                    [EVPN/170] 01:28:51
                    >  to 10.100.0.0 via xe-0/0/0.0
                    [EVPN/170] 01:28:40
                    >  to 10.100.0.0 via xe-0/0/0.0
100.64.0.1/32      *[EVPN/7] 01:29:45
                    >  via irb.1000
100.64.0.2/32      *[Local/0] 01:29:45
                       Local via irb.1000
100.64.10.0/24     *[Direct/0] 01:29:45
                    >  via irb.100
                    [EVPN/170] 01:28:51 
                    >  to 10.100.0.0 via xe-0/0/0.0
100.64.10.1/32     *[Local/0] 01:29:45
                       Local via irb.100
100.64.10.10/32    *[EVPN/7] 01:29:45
                    >  via irb.100
100.64.20.0/24     *[EVPN/170] 01:29:33
                    >  to 10.100.0.0 via xe-0/0/0.0
100.64.30.0/24     *[EVPN/170] 01:28:39
                    >  to 10.100.0.0 via xe-0/0/0.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0004/72                
                   *[Direct/0] 22:59:16
                    >  via lo0.0

mpls.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

19                 *[VPN/0] 01:29:45
                    >  via lsi.1 (WAN), Pop      
                                        
inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:fc00/128
                   *[Direct/0] 22:59:16
                    >  via lo0.0
ff02::2/128        *[INET6/0] 22:59:17
                       MultiRecv

WAN.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

ff02::2/128        *[INET6/0] 01:29:45
                       MultiRecv

bgp.evpn.0: 61 destinations, 61 routes (61 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 04:21:52
                       Indirect
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[EVPN/170] 04:21:53 
                       Indirect
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 04:21:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 04:21:03, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:08, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:10, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[EVPN/170] 01:16:07
                       Indirect
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect         
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 03:25:49, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:06:36, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:06:39, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:17:43, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:45:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[EVPN/170] 01:29:45
                       Indirect
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[EVPN/170] 03:25:56
                       Indirect
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[EVPN/170] 01:16:07
                       Indirect
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[EVPN/170] 01:29:44
                       Indirect
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[EVPN/170] 03:25:56 
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:28:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:28:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:01:43, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:17:43, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:45:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[EVPN/170] 03:25:56
                       Indirect
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[EVPN/170] 01:17:43
                       Indirect
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[EVPN/170] 03:25:56
                       Indirect
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:17:28, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 03:25:49, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:07:07, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:09:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:17:43, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:45:52, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[EVPN/170] 04:21:53
                       Indirect         
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 04:21:03, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:07:10, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 01:29:42
                       Indirect
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:29:45
                       Indirect
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[EVPN/170] 01:29:45
                       Indirect
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:29:31, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:01:38, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:28:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:28:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:01:38, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:28:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:28:39, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0

default-switch.evpn.0: 44 destinations, 44 routes (44 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[EVPN/170] 04:21:53
                       Indirect
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:08, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:10, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 04:02:57
                       Indirect         
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[EVPN/170] 03:48:01
                       Indirect
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[EVPN/170] 03:46:09
                       Indirect
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[EVPN/170] 04:02:35
                       Indirect
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[EVPN/170] 01:16:07
                       Indirect
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[EVPN/170] 04:18:48
                       Indirect
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[EVPN/170] 04:18:35
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:06:36, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:06:39, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[EVPN/170] 01:29:45
                       Indirect
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[EVPN/170] 03:43:45
                       Indirect
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[EVPN/170] 01:16:07
                       Indirect
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[EVPN/170] 01:29:45
                       Indirect
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[EVPN/170] 04:18:35
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:01:43, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[EVPN/170] 04:03:39
                       Indirect
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[EVPN/170] 01:17:43
                       Indirect
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[EVPN/170] 04:21:51
                       Indirect
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:17:28, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:07:07, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0

__default_evpn__.evpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 04:21:52
                       Indirect         
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[EVPN/170] 04:21:53
                       Indirect
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:07:09, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:07:10, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0

WAN.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 01:29:42
                       Indirect
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:29:45
                       Indirect
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[EVPN/170] 01:29:45
                       Indirect
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:01:38, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:01:38, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:17:40, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.0 via xe-0/0/0.0

{master:0}
```

SITE-A-LEAF-2:
```
root@SITE-A-LEAF-2> show bgp summary 
show ethernet-switching vxlan-tunnel-end-point remoteThreading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      51         51          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.1            65000       3434       3030       0       0    22:16:06 Establ
  bgp.evpn.0: 51/51/51/0
  default-switch.evpn.0: 39/39/39/0
  __default_evpn__.evpn.0: 3/3/3/0
  WAN.evpn.0: 9/9/9/0
100.64.0.1            65101        211        207       0       0     1:30:04 Establ
  WAN.inet.0: 1/1/1/0

{master:0}
root@SITE-A-LEAF-2> show ethernet-switching vxlan-tunnel-end-point remote 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.5       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.4       default-switch           571       vtep.32769   1765    RNVE      
    VNID          MC-Group-IP      
    100           0.0.0.0         
    400           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.7       default-switch           576       vtep.32771   1794    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    100           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.8       default-switch           561       vtep.32770   1796    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0               
    1000          0.0.0.0         

{master:0}
root@SITE-A-LEAF-2> show ethernet-switching vxlan-tunnel-end-point source 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.5       lo0.0    0  
    L2-RTT                   Bridge Domain              VNID     MC-Group-IP
    default-switch           v100+100                   100      0.0.0.0        
    default-switch           v1000+1000                 1000     0.0.0.0        
    default-switch           v200+200                   200      0.0.0.0        
    default-switch           v400+400                   400      0.0.0.0        

{master:0}
root@SITE-A-LEAF-2> show ethernet-switching table 

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 14 entries, 14 learned
Routing instance : default-switch
   Vlan                MAC                 MAC      Logical                SVLBNH/      Active
   name                address             flags    interface              VENH Index   source
   v100                00:00:5e:00:53:01   D        vtep.32769                          10.200.0.4                    
   v100                50:0f:0b:00:33:00   D        vtep.32769                          10.200.0.4                    
   v100                50:14:02:00:33:00   D        vtep.32769                          10.200.0.4                    
   v100                50:2f:a7:00:33:00   D        vtep.32769                          10.200.0.4                    
   v100                50:36:be:00:35:00   D        vtep.32771                          10.200.0.7                    
   v1000               02:05:86:71:04:00   D        vtep.32770                          10.200.0.8                    
   v1000               02:05:86:71:cb:00   D        vtep.32771                          10.200.0.7                    
   v1000               02:05:86:71:fc:00   D        vtep.32769                          10.200.0.4                    
   v1000               aa:bb:cc:00:01:10   DR       ae10.0               
   v1000               aa:bb:cc:00:02:10   DR       esi.1742                            00:01:01:01:01:01:01:01:01:02 
   v200                50:11:2b:00:34:00   D        xe-0/0/2.0           
   v400                aa:bb:cc:00:01:10   DR       ae10.0               
   v400                aa:bb:cc:00:02:10   DR       esi.1742                            00:01:01:01:01:01:01:01:01:02 
   v400                aa:bb:cc:00:03:00   D        vtep.32770                          10.200.0.8                    

{master:0}
root@SITE-A-LEAF-2> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        00:00:5e:00:53:01  10.200.0.4                     Jun 06 13:27:19  100.64.10.1
     100        50:0f:0b:00:33:00  10.200.0.4                     Jun 06 13:27:19
     100        50:14:02:00:33:00  10.200.0.4                     Jun 06 13:27:19  100.64.10.10
     100        50:2f:a7:00:33:00  10.200.0.4                     Jun 06 13:27:19
     100        50:36:be:00:35:00  10.200.0.7                     Jun 06 13:27:20  100.64.10.20
     200        00:00:5e:00:53:01  irb.200                        Jun 06 10:10:01  100.64.20.1
     200        50:11:2b:00:34:00  xe-0/0/2.0                     Jun 06 10:12:44  100.64.20.10
     400        aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 12:28:37  100.64.40.1
     400        aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:38:06
     400        aa:bb:cc:00:03:00  10.200.0.8                     Jun 06 12:27:13  100.64.40.3
     1000       02:05:86:71:04:00  10.200.0.8                     Jun 06 11:58:51  100.64.0.6
     1000       02:05:86:71:3a:00  irb.1000                       Jun 06 09:25:16  100.64.0.3
     1000       02:05:86:71:cb:00  10.200.0.7                     Jun 06 12:15:53  100.64.0.5
     1000       02:05:86:71:fc:00  10.200.0.4                     Jun 06 12:14:59  100.64.0.2
     1000       aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 13:22:55  100.64.0.1
     1000       aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:43:00  100.64.0.4

{master:0}
root@SITE-A-LEAF-2> show route 

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 22:16:27, metric 20
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.2/31      *[Direct/0] 22:16:36
                    >  via xe-0/0/0.0
10.100.0.3/32      *[Local/0] 22:16:36
                       Local via xe-0/0/0.0
10.100.0.4/31      *[IS-IS/18] 22:16:27, metric 20
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 02:24:31, metric 60
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.8/31      *[IS-IS/18] 01:46:54, metric 60
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.10/31     *[IS-IS/18] 02:24:58, metric 50
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.12/31     *[IS-IS/18] 03:19:03, metric 30
                    >  to 10.100.0.2 via xe-0/0/0.0
10.100.0.14/31     *[IS-IS/18] 02:57:09, metric 40
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.1/32      *[IS-IS/18] 22:16:27, metric 10
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 02:24:31, metric 50
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.3/32      *[IS-IS/18] 02:57:09, metric 30
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.4/32      *[IS-IS/18] 22:16:17, metric 20
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.5/32      *[Direct/0] 22:18:33
                    >  via lo0.0
10.200.0.6/32      *[IS-IS/18] 03:19:03, metric 20
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.7/32      *[IS-IS/18] 02:24:22, metric 60
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.8/32      *[IS-IS/18] 01:46:47, metric 60
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.9/32      *[IS-IS/18] 02:24:58, metric 40
                    >  to 10.100.0.2 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 22:18:33
                    >  via em1.0
169.254.0.2/32     *[Local/0] 22:18:33
                       Local via em1.0

:vxlan.inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.2/31      *[Direct/0] 22:16:36
                    >  via xe-0/0/0.0
10.100.0.3/32      *[Local/0] 22:16:36
                       Local via xe-0/0/0.0
10.200.0.4/32      *[Static/1] 04:22:35, metric2 20
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.5/32      *[Direct/0] 22:18:31
                    >  via lo0.0
10.200.0.7/32      *[Static/1] 02:10:05, metric2 60
                    >  to 10.100.0.2 via xe-0/0/0.0
10.200.0.8/32      *[Static/1] 01:46:35, metric2 60
                    >  to 10.100.0.2 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 22:18:31
                    >  via em1.0
169.254.0.2/32     *[Local/0] 22:18:31
                       Local via em1.0

WAN.inet.0: 9 destinations, 16 routes (9 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[BGP/170] 01:30:14, localpref 100
                      AS path: 65101 I, validation-state: unverified
                    >  to 100.64.0.1 via irb.1000
                    [EVPN/170] 01:30:17
                    >  to 10.100.0.2 via xe-0/0/0.0
                    [EVPN/170] 00:02:21
                    >  to 10.100.0.2 via xe-0/0/0.0
                    [EVPN/170] 00:02:21
                    >  to 10.100.0.2 via xe-0/0/0.0
100.64.0.0/24      *[Direct/0] 01:30:17
                    >  via irb.1000
                    [EVPN/170] 01:30:17
                    >  to 10.100.0.2 via xe-0/0/0.0
                    [EVPN/170] 01:29:33
                    >  to 10.100.0.2 via xe-0/0/0.0
                    [EVPN/170] 01:29:23
                    >  to 10.100.0.2 via xe-0/0/0.0
100.64.0.1/32      *[EVPN/7] 01:30:17
                    >  via irb.1000
100.64.0.3/32      *[Local/0] 01:30:17
                       Local via irb.1000
100.64.10.0/24     *[EVPN/170] 01:30:17
                    >  to 10.100.0.2 via xe-0/0/0.0
                    [EVPN/170] 01:29:33 
                    >  to 10.100.0.2 via xe-0/0/0.0
100.64.20.0/24     *[Direct/0] 01:30:17
                    >  via irb.200
100.64.20.1/32     *[Local/0] 01:30:17
                       Local via irb.200
100.64.20.10/32    *[EVPN/7] 01:30:17
                    >  via irb.200
100.64.30.0/24     *[EVPN/170] 01:29:22
                    >  to 10.100.0.2 via xe-0/0/0.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0005/72                
                   *[Direct/0] 22:18:33
                    >  via lo0.0

mpls.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

19                 *[VPN/0] 01:30:17
                    >  via lsi.1 (WAN), Pop      
                                        
inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:3a00/128
                   *[Direct/0] 22:18:33
                    >  via lo0.0
ff02::2/128        *[INET6/0] 22:18:34
                       MultiRecv

WAN.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

ff02::2/128        *[INET6/0] 01:30:17
                       MultiRecv

bgp.evpn.0: 66 destinations, 66 routes (66 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 04:22:34, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 04:22:35, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 04:21:45
                       Indirect
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[EVPN/170] 04:21:46
                       Indirect
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:52, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:53, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:16:49, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 03:26:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 03:26:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 03:26:33
                       Indirect
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[EVPN/170] 03:26:33
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[EVPN/170] 03:26:33
                       Indirect
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:07:19, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:10:05, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:07:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:18:12, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:46:34, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:16:49, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:30:27, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 03:26:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[EVPN/170] 01:30:16
                       Indirect
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[EVPN/170] 03:26:33 
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[EVPN/170] 01:30:16
                       Indirect
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:02:26, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:18:12, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:46:34, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:18:12, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 03:26:33, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[EVPN/170] 03:26:33
                       Indirect
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[EVPN/170] 01:18:12
                       Indirect         
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[EVPN/170] 03:26:33
                       Indirect
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 00:18:06, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:07:49, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:10:05, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:18:12, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:46:35, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 04:21:50, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[EVPN/170] 04:21:46
                       Indirect
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:07:52, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:07:53, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:30:24, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:30:27, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:30:27, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 01:30:14
                       Indirect
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:30:17
                       Indirect
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[EVPN/170] 01:30:17
                       Indirect
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:02:21, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:29:33, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:02:21, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:29:23, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:29:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0

default-switch.evpn.0: 49 destinations, 49 routes (49 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[EVPN/170] 04:21:46
                       Indirect
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:52, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:51, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:53, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 03:35:24
                       Indirect
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[EVPN/170] 03:33:27
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[EVPN/170] 04:20:09
                       Indirect
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:07:19, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:07:22, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[EVPN/170] 01:30:16
                       Indirect
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[EVPN/170] 03:32:42
                       Indirect
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[EVPN/170] 01:30:16
                       Indirect
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:02:26, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[EVPN/170] 03:35:23
                       Indirect
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[EVPN/170] 01:18:12
                       Indirect
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[EVPN/170] 04:21:45 
                       Indirect
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:07:49, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
                                        
__default_evpn__.evpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 04:21:45
                       Indirect
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[EVPN/170] 04:21:46
                       Indirect
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:07:52, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:07:53, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
                                        
WAN.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 01:30:14
                       Indirect
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:30:17
                       Indirect
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[EVPN/170] 01:30:17 
                       Indirect
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:02:21, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:02:21, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 00:18:02, localpref 100, from 10.200.0.1
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.2 via xe-0/0/0.0

{master:0}
```

SITE-A-BLEAF-1:
```
root@SITE-A-BLEAF-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
show ethernet-switching vxlan-tunnel-end-point remote                      71         71          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.1            65000        522        524       0       0     3:19:14 Establ
  bgp.evpn.0: 36/36/36/0
  default-switch.evpn.0: 4/4/4/0
  __default_evpn__.evpn.0: 0/0/0/0
10.200.0.3            65001        476        431       0       0     2:47:07 Establ
  bgp.evpn.0: 35/35/35/0
  default-switch.evpn.0: 4/4/4/0
  __default_evpn__.evpn.0: 0/0/0/0

{master:0}
root@SITE-A-BLEAF-1> ... vxlan-tunnel-end-point remote                       
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.6       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.4       default-switch           571       vtep.32769   1768    RNVE      
 10.200.0.5       default-switch           572       vtep.32770   1770    RNVE      
 10.200.0.7       default-switch           554       vtep.32772   1714    RNVE      
 10.200.0.8       default-switch           552       vtep.32771   1713    RNVE      

{master:0}
root@SITE-A-BLEAF-1> ... vxlan-tunnel-end-point source                       
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.6       lo0.0    0  

{master:0}
root@SITE-A-BLEAF-1> show ethernet-switching table 

{master:0}
root@SITE-A-BLEAF-1> show evpn database 

{master:0}
root@SITE-A-BLEAF-1> show route 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 03:19:57, metric 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.100.0.2/31      *[IS-IS/18] 03:19:57, metric 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.100.0.4/31      *[Direct/0] 03:20:14
                    >  via xe-0/0/0.0
10.100.0.5/32      *[Local/0] 03:20:14
                       Local via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 02:25:26, metric 40
                    >  to 10.100.0.12 via xe-0/0/1.0
10.100.0.8/31      *[IS-IS/18] 01:47:49, metric 40
                    >  to 10.100.0.12 via xe-0/0/1.0
10.100.0.10/31     *[IS-IS/18] 02:25:53, metric 30
                    >  to 10.100.0.12 via xe-0/0/1.0
10.100.0.12/31     *[Direct/0] 03:20:14
                    >  via xe-0/0/1.0
10.100.0.13/32     *[Local/0] 03:20:14
                       Local via xe-0/0/1.0
10.100.0.14/31     *[IS-IS/18] 02:58:04, metric 20
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.1/32      *[IS-IS/18] 03:19:57, metric 10
                    >  to 10.100.0.4 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 02:25:26, metric 30
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.3/32      *[IS-IS/18] 02:58:04, metric 10
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.4/32      *[IS-IS/18] 03:19:47, metric 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.200.0.5/32      *[IS-IS/18] 03:19:47, metric 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.200.0.6/32      *[Direct/0] 03:22:14
                    >  via lo0.0
10.200.0.7/32      *[IS-IS/18] 02:25:16, metric 40
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.8/32      *[IS-IS/18] 01:47:41, metric 40
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.9/32      *[IS-IS/18] 02:25:53, metric 20
                    >  to 10.100.0.12 via xe-0/0/1.0
169.254.0.0/24     *[Direct/0] 03:22:14
                    >  via em1.0
169.254.0.2/32     *[Local/0] 03:22:14
                       Local via em1.0
                                        
:vxlan.inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.4/31      *[Direct/0] 03:20:15
                    >  via xe-0/0/0.0
10.100.0.5/32      *[Local/0] 03:20:15
                       Local via xe-0/0/0.0
10.100.0.12/31     *[Direct/0] 03:20:15
                    >  via xe-0/0/1.0
10.100.0.13/32     *[Local/0] 03:20:15
                       Local via xe-0/0/1.0
10.200.0.4/32      *[Static/1] 03:19:25, metric2 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.200.0.5/32      *[Static/1] 03:19:25, metric2 20
                    >  to 10.100.0.4 via xe-0/0/0.0
10.200.0.6/32      *[Direct/0] 03:22:12
                    >  via lo0.0
10.200.0.7/32      *[Static/1] 00:08:48, metric2 40
                    >  to 10.100.0.12 via xe-0/0/1.0
10.200.0.8/32      *[Static/1] 00:08:48, metric2 40
                    >  to 10.100.0.12 via xe-0/0/1.0
169.254.0.0/24     *[Direct/0] 03:22:12
                    >  via em1.0        
169.254.0.2/32     *[Local/0] 03:22:12
                       Local via em1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0006/72                
                   *[Direct/0] 03:22:15
                    >  via lo0.0

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:e200/128
                   *[Direct/0] 03:22:15
                    >  via lo0.0
ff02::2/128        *[INET6/0] 03:22:16
                       MultiRecv

bgp.evpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:08:47, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:08:48, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:08:48, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:08:49, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:17:44, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:08:15, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:08:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:47:31, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[BGP/170] 01:47:31, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:24:40, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:47:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:17:44, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:31:22, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[BGP/170] 01:31:11, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:31:11, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:30:29, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:30:29, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:03:21, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[BGP/170] 01:47:31, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[BGP/170] 01:46:10, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:22:06, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:47:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 03:17:18, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:19:21, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:19:06, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:08:45, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:11:01, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[BGP/170] 01:47:31, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:28:14, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:47:31, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 03:17:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:08:48, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:08:49, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:31:20, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:31:22, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:31:22, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:31:09, localpref 100, from 10.200.0.1
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:31:11, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:31:11, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:03:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:30:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:30:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:03:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:30:18, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:30:18, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0

default-switch.evpn.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.4 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:07:17, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.12 via xe-0/0/1.0

  {master:0}
```

SITE-A-SPINE-1:
```
root@SITE-A-SPINE-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 3 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      71         71          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.4            65000       3156       3580       0       0    23:08:59 Establ
  bgp.evpn.0: 21/21/21/0
10.200.0.5            65000       3115       3589       0       0    22:52:54 Establ
  bgp.evpn.0: 15/15/15/0
10.200.0.6            65000        605        601       0       0     3:55:09 Establ
  bgp.evpn.0: 35/35/35/0

{master:0}
root@SITE-A-SPINE-1> ... vxlan-tunnel-end-point remote                       

{master:0}
root@SITE-A-SPINE-1> ... vxlan-tunnel-end-point source                       

{master:0}
root@SITE-A-SPINE-1> show ethernet-switching table 

{master:0}
root@SITE-A-SPINE-1> show evpn database 
show route
{master:0}
root@SITE-A-SPINE-1> show route 

inet.0: 22 destinations, 22 routes (22 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[Direct/0] 23:09:44
                    >  via xe-0/0/0.0
10.100.0.0/32      *[Local/0] 23:09:44
                       Local via xe-0/0/0.0
10.100.0.2/31      *[Direct/0] 22:57:56
                    >  via xe-0/0/1.0
10.100.0.2/32      *[Local/0] 22:57:56
                       Local via xe-0/0/1.0
10.100.0.4/31      *[Direct/0] 22:57:56
                    >  via xe-0/0/2.0
10.100.0.4/32      *[Local/0] 22:57:56
                       Local via xe-0/0/2.0
10.100.0.6/31      *[IS-IS/18] 03:01:23, metric 50
                    >  to 10.100.0.5 via xe-0/0/2.0
10.100.0.8/31      *[IS-IS/18] 02:23:46, metric 50
                    >  to 10.100.0.5 via xe-0/0/2.0
10.100.0.10/31     *[IS-IS/18] 03:01:49, metric 40
                    >  to 10.100.0.5 via xe-0/0/2.0
10.100.0.12/31     *[IS-IS/18] 03:55:54, metric 20
                    >  to 10.100.0.5 via xe-0/0/2.0
10.100.0.14/31     *[IS-IS/18] 03:34:01, metric 30
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.1/32      *[Direct/0] 23:33:59
                    >  via lo0.0
10.200.0.2/32      *[IS-IS/18] 03:01:23, metric 40
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.3/32      *[IS-IS/18] 03:34:01, metric 20
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.4/32      *[IS-IS/18] 23:09:29, metric 10
                    >  to 10.100.0.1 via xe-0/0/0.0
10.200.0.5/32      *[IS-IS/18] 22:53:18, metric 10
                    >  to 10.100.0.3 via xe-0/0/1.0
10.200.0.6/32      *[IS-IS/18] 03:55:54, metric 10
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.7/32      *[IS-IS/18] 03:01:13, metric 50
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.8/32      *[IS-IS/18] 02:23:38, metric 50
                    >  to 10.100.0.5 via xe-0/0/2.0
10.200.0.9/32      *[IS-IS/18] 03:01:49, metric 30
                    >  to 10.100.0.5 via xe-0/0/2.0
169.254.0.0/24     *[Direct/0] 23:33:59
                    >  via em1.0
169.254.0.2/32     *[Local/0] 23:33:59  
                       Local via em1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0001/72                
                   *[Direct/0] 23:33:59
                    >  via lo0.0

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:cb00/128
                   *[Direct/0] 23:33:59
                    >  via lo0.0
ff02::2/128        *[INET6/0] 23:34:05
                       MultiRecv

bgp.evpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 04:59:26, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 04:59:27, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 04:58:36, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 04:58:37, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:44:43, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:44:44, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:44:43, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:44:44, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:53:40, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 04:03:25, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[BGP/170] 04:03:24, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 04:03:25, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:44:11, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:44:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:23:27, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[BGP/170] 02:23:27, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 02:00:36, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 02:23:26, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:07:19, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:53:40, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 02:07:18, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[BGP/170] 02:07:07, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[BGP/170] 04:03:24, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 02:07:07, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:06:25, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 02:06:25, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:39:17, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[BGP/170] 02:23:27, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[BGP/170] 02:22:06, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:58:02, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 02:23:26, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:55:17, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 04:03:30, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[BGP/170] 04:03:24, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:55:03, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 04:03:25, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:44:41, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:46:57, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[BGP/170] 02:23:27, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 02:04:10, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 02:23:26, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 04:59:27, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 04:58:37, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:44:44, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:44:44, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 02:07:16, localpref 100, from 10.200.0.4
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 02:07:19, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 02:07:19, localpref 100, from 10.200.0.4
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.1 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 02:07:05, localpref 100, from 10.200.0.5
                      AS path: 65101 I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 02:07:07, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 02:07:07, localpref 100, from 10.200.0.5
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.3 via xe-0/0/1.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:39:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 02:06:26, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 02:06:26, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:39:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 02:06:14, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 02:06:14, localpref 100, from 10.200.0.6
                      AS path: 65010 65001 I, validation-state: unverified
                    >  to 10.100.0.5 via xe-0/0/2.0

{master:0}
```
SIPER-SPINE:
```
root@SIPER-SPINE> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      71         71          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.6            65010        436        479       0       0     2:49:14 Establ
  bgp.evpn.0: 36/36/36/0
10.200.0.9            65020        414        417       0       0     2:27:39 Establ
  bgp.evpn.0: 35/35/35/0

{master:0}
root@SIPER-SPINE> show ethernet-switching vxlan-tunnel-end-point remote 

{master:0}
root@SIPER-SPINE> show ethernet-switching vxlan-tunnel-end-point source 

{master:0}
root@SIPER-SPINE> show ethernet-switching table 

{master:0}
root@SIPER-SPINE> show evpn database 

{master:0}
root@SIPER-SPINE> show route 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 02:59:56, metric 30
                    >  to 10.100.0.13 via xe-0/0/0.0
10.100.0.2/31      *[IS-IS/18] 02:59:56, metric 30
                    >  to 10.100.0.13 via xe-0/0/0.0
10.100.0.4/31      *[IS-IS/18] 03:00:06, metric 20
                    >  to 10.100.0.13 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 02:27:29, metric 30
                    >  to 10.100.0.15 via xe-0/0/1.0
10.100.0.8/31      *[IS-IS/18] 01:49:51, metric 30
                    >  to 10.100.0.15 via xe-0/0/1.0
10.100.0.10/31     *[IS-IS/18] 02:27:55, metric 20
                    >  to 10.100.0.15 via xe-0/0/1.0
10.100.0.12/31     *[Direct/0] 03:00:23
                    >  via xe-0/0/0.0
10.100.0.12/32     *[Local/0] 03:00:23
                       Local via xe-0/0/0.0
10.100.0.14/31     *[Direct/0] 03:00:23
                    >  via xe-0/0/1.0
10.100.0.14/32     *[Local/0] 03:00:23
                       Local via xe-0/0/1.0
10.200.0.1/32      *[IS-IS/18] 02:59:56, metric 20
                    >  to 10.100.0.13 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 02:27:29, metric 20
                    >  to 10.100.0.15 via xe-0/0/1.0
10.200.0.3/32      *[Direct/0] 03:02:22
                    >  via lo0.0
10.200.0.4/32      *[IS-IS/18] 02:59:56, metric 30
                    >  to 10.100.0.13 via xe-0/0/0.0
10.200.0.5/32      *[IS-IS/18] 02:59:56, metric 30
                    >  to 10.100.0.13 via xe-0/0/0.0
10.200.0.6/32      *[IS-IS/18] 03:00:06, metric 10
                    >  to 10.100.0.13 via xe-0/0/0.0
10.200.0.7/32      *[IS-IS/18] 02:27:19, metric 30
                    >  to 10.100.0.15 via xe-0/0/1.0
10.200.0.8/32      *[IS-IS/18] 01:49:44, metric 30
                    >  to 10.100.0.15 via xe-0/0/1.0
10.200.0.9/32      *[IS-IS/18] 02:27:55, metric 10
                    >  to 10.100.0.15 via xe-0/0/1.0
169.254.0.0/24     *[Direct/0] 03:02:22
                    >  via em1.0
169.254.0.2/32     *[Local/0] 03:02:22
                       Local via em1.0
                                        
iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0003/72                
                   *[Direct/0] 03:02:23
                    >  via lo0.0

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:d700/128
                   *[Direct/0] 03:02:23
                    >  via lo0.0
ff02::2/128        *[INET6/0] 03:02:26
                       MultiRecv

bgp.evpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:14:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:14:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:50, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:10:51, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:50, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:10:51, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:19:46, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:10:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:10:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:49:34, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[BGP/170] 01:49:34, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:26:43, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:49:33, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:19:46, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:33:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[BGP/170] 01:33:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:33:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:32:32, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:32:32, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:05:24, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[BGP/170] 01:32:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[BGP/170] 01:48:13, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:24:09, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:32:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:21:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:21:09, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 02:14:23, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:10:48, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:13:04, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[BGP/170] 01:49:34, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:30:17, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:49:33, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 02:14:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 02:14:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:10:51, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:10:51, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:33:22, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 65101 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:33:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:33:24, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:33:11, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 65101 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:33:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:33:13, localpref 100, from 10.200.0.6
                      AS path: 65010 65000 I, validation-state: unverified
                    >  to 10.100.0.13 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:05:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 65102 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:32:33, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:32:33, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:05:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 65102 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:32:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:32:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65000 I, validation-state: unverified
                    >  to 10.100.0.15 via xe-0/0/1.0

{master:0}
```

SITE-B-BLEAF-1:
```
root@SITE-B-BLEAF-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      71         71          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.2            65000        403        386       0       0     2:28:27 Establ
  bgp.evpn.0: 35/35/35/0
  default-switch.evpn.0: 4/4/4/0
  __default_evpn__.evpn.0: 0/0/0/0
10.200.0.3            65001        422        417       0       0     2:28:49 Establ
  bgp.evpn.0: 36/36/36/0
  default-switch.evpn.0: 4/4/4/0
  __default_evpn__.evpn.0: 0/0/0/0

{master:0}
root@SITE-B-BLEAF-1> ... vxlan-tunnel-end-point remote                       
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.9       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.4       default-switch           573       vtep.32771   1778    RNVE      
 10.200.0.5       default-switch           574       vtep.32772   1779    RNVE      
 10.200.0.7       default-switch           554       vtep.32770   1714    RNVE      
 10.200.0.8       default-switch           552       vtep.32769   1713    RNVE      

{master:0}
root@SITE-B-BLEAF-1> ... vxlan-tunnel-end-point source                       
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.9       lo0.0    0  

{master:0}
root@SITE-B-BLEAF-1> show ethernet-switching table 

{master:0}
root@SITE-B-BLEAF-1> show evpn database 

{master:0}
root@SITE-B-BLEAF-1> show route 

inet.0: 21 destinations, 21 routes (21 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 02:29:00, metric 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.100.0.2/31      *[IS-IS/18] 02:29:00, metric 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.100.0.4/31      *[IS-IS/18] 02:29:00, metric 30
                    >  to 10.100.0.14 via xe-0/0/1.0
10.100.0.6/31      *[IS-IS/18] 02:28:44, metric 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.100.0.8/31      *[IS-IS/18] 01:51:07, metric 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.100.0.10/31     *[Direct/0] 02:29:27
                    >  via xe-0/0/0.0
10.100.0.11/32     *[Local/0] 02:29:27
                       Local via xe-0/0/0.0
10.100.0.12/31     *[IS-IS/18] 02:29:11, metric 20
                    >  to 10.100.0.14 via xe-0/0/1.0
10.100.0.14/31     *[Direct/0] 02:29:27
                    >  via xe-0/0/1.0
10.100.0.15/32     *[Local/0] 02:29:27
                       Local via xe-0/0/1.0
10.200.0.1/32      *[IS-IS/18] 02:29:00, metric 30
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.2/32      *[IS-IS/18] 02:28:44, metric 10
                    >  to 10.100.0.10 via xe-0/0/0.0
10.200.0.3/32      *[IS-IS/18] 02:29:11, metric 10
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.4/32      *[IS-IS/18] 02:29:00, metric 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.5/32      *[IS-IS/18] 02:29:00, metric 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.6/32      *[IS-IS/18] 02:29:00, metric 20
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.7/32      *[IS-IS/18] 02:28:34, metric 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.200.0.8/32      *[IS-IS/18] 01:51:00, metric 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.200.0.9/32      *[Direct/0] 02:31:37
                    >  via lo0.0
169.254.0.0/24     *[Direct/0] 02:31:37
                    >  via em1.0
169.254.0.2/32     *[Local/0] 02:31:37
                       Local via em1.0
                                        
:vxlan.inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.10/31     *[Direct/0] 02:29:27
                    >  via xe-0/0/0.0
10.100.0.11/32     *[Local/0] 02:29:27
                       Local via xe-0/0/0.0
10.100.0.14/31     *[Direct/0] 02:29:27
                    >  via xe-0/0/1.0
10.100.0.15/32     *[Local/0] 02:29:27
                       Local via xe-0/0/1.0
10.200.0.4/32      *[Static/1] 02:15:37, metric2 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.5/32      *[Static/1] 02:15:37, metric2 40
                    >  to 10.100.0.14 via xe-0/0/1.0
10.200.0.7/32      *[Static/1] 00:12:05, metric2 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.200.0.8/32      *[Static/1] 00:12:06, metric2 20
                    >  to 10.100.0.10 via xe-0/0/0.0
10.200.0.9/32      *[Direct/0] 02:31:35
                    >  via lo0.0
169.254.0.0/24     *[Direct/0] 02:31:35
                    >  via em1.0        
169.254.0.2/32     *[Local/0] 02:31:35
                       Local via em1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0009/72                
                   *[Direct/0] 02:31:37
                    >  via lo0.0

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:7f00/128
                   *[Direct/0] 02:31:37
                    >  via lo0.0
ff02::2/128        *[INET6/0] 02:31:38
                       MultiRecv

bgp.evpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:12:05, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:12:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:12:06, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:12:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:21:02, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:11:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:11:36, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:50:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[BGP/170] 01:50:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:27:58, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:50:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:21:02, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:34:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[BGP/170] 01:34:28, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[BGP/170] 02:15:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:34:28, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:33:48, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:33:48, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:06:40, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[BGP/170] 01:33:37, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[BGP/170] 01:49:28, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:25:24, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:33:37, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:22:38, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:22:24, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:12:04, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:28:35, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[BGP/170] 01:50:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:31:33, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:50:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 02:15:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:12:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:12:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:34:37, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:34:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:34:39, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:34:26, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:34:28, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:34:28, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:06:35, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:33:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:33:49, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:06:35, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:33:37, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:33:37, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0

default-switch.evpn.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.3
                      AS path: 65001 I, validation-state: unverified
                    >  to 10.100.0.14 via xe-0/0/1.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:10:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.10 via xe-0/0/0.0

{master:0}
```

SITE-B-LEAF-1:
```
root@SITE-B-LEAF-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      43         43          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.2            65000        570        402       0       0     2:39:02 Establ
  bgp.evpn.0: 43/43/43/0
  default-switch.evpn.0: 31/31/31/0
  __default_evpn__.evpn.0: 3/3/3/0
  WAN.evpn.0: 9/9/9/0
100.64.0.4            65102         27         23       0       1        7:35 Establ
  WAN.inet.0: 1/1/1/0

{master:0}
root@SITE-B-LEAF-1> show ethernet-switching vxlan-tunnel-end-point remote 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.7       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.4       default-switch           575       vtep.32770   1786    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0         
    100           0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.5       default-switch           576       vtep.32771   1795    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.8       default-switch           574       vtep.32769   1792    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0               

{master:0}
root@SITE-B-LEAF-1> show ethernet-switching vxlan-tunnel-end-point source 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.7       lo0.0    0  
    L2-RTT                   Bridge Domain              VNID     MC-Group-IP
    default-switch           v100+100                   100      0.0.0.0        
    default-switch           v1000+1000                 1000     0.0.0.0        
    default-switch           v400+400                   400      0.0.0.0        

{master:0}
root@SITE-B-LEAF-1> show ethernet-switching table 

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 12 entries, 12 learned
Routing instance : default-switch
   Vlan                MAC                 MAC      Logical                SVLBNH/      Active
   name                address             flags    interface              VENH Index   source
   v100                50:0f:0b:00:33:00   D        vtep.32770                          10.200.0.4                    
   v100                50:14:02:00:33:00   D        vtep.32770                          10.200.0.4                    
   v100                50:2f:a7:00:33:00   D        vtep.32770                          10.200.0.4                    
   v100                50:36:be:00:35:00   D        xe-0/0/2.0           
   v1000               02:05:86:71:04:00   D        vtep.32769                          10.200.0.8                    
   v1000               02:05:86:71:3a:00   D        vtep.32771                          10.200.0.5                    
   v1000               02:05:86:71:fc:00   D        vtep.32770                          10.200.0.4                    
   v1000               aa:bb:cc:00:01:10   DR       esi.1796                            00:01:01:01:01:01:01:01:01:01 
   v1000               aa:bb:cc:00:02:10   DL       ae10.0               
   v400                aa:bb:cc:00:01:10   DR       esi.1796                            00:01:01:01:01:01:01:01:01:01 
   v400                aa:bb:cc:00:02:10   DL       ae10.0               
   v400                aa:bb:cc:00:03:00   D        vtep.32769                          10.200.0.8                    

{master:0}
root@SITE-B-LEAF-1> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        00:00:5e:00:53:01  irb.100                        Jun 06 11:10:58  100.64.10.1
     100        50:0f:0b:00:33:00  10.200.0.4                     Jun 06 11:34:02
     100        50:14:02:00:33:00  10.200.0.4                     Jun 06 11:34:02  100.64.10.10
     100        50:2f:a7:00:33:00  10.200.0.4                     Jun 06 11:34:02
     100        50:36:be:00:35:00  xe-0/0/2.0                     Jun 06 11:19:36  100.64.10.20
     400        aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 12:28:38  100.64.40.1
     400        aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:38:04
     400        aa:bb:cc:00:03:00  10.200.0.8                     Jun 06 12:24:15  100.64.40.3
     1000       02:05:86:71:04:00  10.200.0.8                     Jun 06 12:16:02  100.64.0.6
     1000       02:05:86:71:3a:00  10.200.0.5                     Jun 06 12:15:12  100.64.0.3
     1000       02:05:86:71:cb:00  irb.1000                       Jun 06 11:18:24  100.64.0.5
     1000       02:05:86:71:fc:00  10.200.0.4                     Jun 06 12:15:01  100.64.0.2
     1000       aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 13:22:56  100.64.0.1
     1000       aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:42:58  100.64.0.4

{master:0}
root@SITE-B-LEAF-1> show route 

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 02:29:48, metric 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.2/31      *[IS-IS/18] 02:29:48, metric 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.4/31      *[IS-IS/18] 02:29:48, metric 50
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.6/31      *[Direct/0] 02:39:57
                    >  via xe-0/0/0.0
10.100.0.7/32      *[Local/0] 02:39:57
                       Local via xe-0/0/0.0
10.100.0.8/31      *[IS-IS/18] 01:52:21, metric 20
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.10/31     *[IS-IS/18] 02:39:49, metric 20
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.12/31     *[IS-IS/18] 02:29:49, metric 40
                    >  to 10.100.0.6 via xe-0/0/0.0
10.100.0.14/31     *[IS-IS/18] 02:29:59, metric 30
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.1/32      *[IS-IS/18] 02:29:48, metric 50
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 02:39:49, metric 10
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.3/32      *[IS-IS/18] 02:29:49, metric 30
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.4/32      *[IS-IS/18] 02:29:48, metric 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.5/32      *[IS-IS/18] 02:29:48, metric 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.6/32      *[IS-IS/18] 02:29:48, metric 40
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.7/32      *[Direct/0] 02:42:05
                    >  via lo0.0
10.200.0.8/32      *[IS-IS/18] 01:52:14, metric 20
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.9/32      *[IS-IS/18] 02:29:59, metric 20
                    >  to 10.100.0.6 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 02:42:05
                    >  via em1.0
169.254.0.2/32     *[Local/0] 02:42:05
                       Local via em1.0

:vxlan.inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.6/31      *[Direct/0] 02:39:57
                    >  via xe-0/0/0.0
10.100.0.7/32      *[Local/0] 02:39:57
                       Local via xe-0/0/0.0
10.200.0.4/32      *[Static/1] 02:16:52, metric2 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.5/32      *[Static/1] 02:16:52, metric2 60
                    >  to 10.100.0.6 via xe-0/0/0.0
10.200.0.7/32      *[Direct/0] 02:42:05
                    >  via lo0.0
10.200.0.8/32      *[Static/1] 01:52:02, metric2 20
                    >  to 10.100.0.6 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 02:42:05
                    >  via em1.0
169.254.0.2/32     *[Local/0] 02:42:05
                       Local via em1.0

WAN.inet.0: 9 destinations, 16 routes (9 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[BGP/170] 00:07:49, localpref 100
                      AS path: 65102 I, validation-state: unverified
                    >  to 100.64.0.4 via irb.1000
                    [EVPN/170] 00:07:49
                    >  to 10.100.0.6 via xe-0/0/0.0
                    [EVPN/170] 01:35:03
                    >  to 10.100.0.6 via xe-0/0/0.0
                    [EVPN/170] 01:35:03
                    >  to 10.100.0.6 via xe-0/0/0.0
100.64.0.0/24      *[Direct/0] 01:35:03
                    >  via irb.1000
                    [EVPN/170] 01:34:51
                    >  to 10.100.0.6 via xe-0/0/0.0
                    [EVPN/170] 01:35:03
                    >  to 10.100.0.6 via xe-0/0/0.0
                    [EVPN/170] 01:35:03
                    >  to 10.100.0.6 via xe-0/0/0.0
100.64.0.4/32      *[EVPN/7] 00:07:54
                    >  via irb.1000
100.64.0.5/32      *[Local/0] 01:35:03
                       Local via irb.1000
100.64.10.0/24     *[Direct/0] 01:35:03
                    >  via irb.100
                    [EVPN/170] 01:35:03 
                    >  to 10.100.0.6 via xe-0/0/0.0
100.64.10.1/32     *[Local/0] 01:35:03
                       Local via irb.100
100.64.10.20/32    *[EVPN/7] 01:35:03
                    >  via irb.100
100.64.20.0/24     *[EVPN/170] 01:35:03
                    >  to 10.100.0.6 via xe-0/0/0.0
100.64.30.0/24     *[EVPN/170] 01:34:50
                    >  to 10.100.0.6 via xe-0/0/0.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0007/72                
                   *[Direct/0] 02:42:05
                    >  via lo0.0

mpls.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

19                 *[VPN/0] 01:35:03
                    >  via lsi.1 (WAN), Pop      
                                        
inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:cb00/128
                   *[Direct/0] 02:42:05
                    >  via lo0.0
ff02::2/128        *[INET6/0] 02:42:08
                       MultiRecv

WAN.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

ff02::2/128        *[INET6/0] 01:35:04
                       MultiRecv

bgp.evpn.0: 61 destinations, 61 routes (61 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 00:13:20
                       Indirect
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[EVPN/170] 00:13:21
                       Indirect
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:13:20, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:13:21, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:22:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 02:39:55
                       Indirect
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[EVPN/170] 02:34:35
                       Indirect
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[EVPN/170] 00:12:49
                       Indirect
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[EVPN/170] 02:32:28
                       Indirect         
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[EVPN/170] 00:12:50
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:29:12, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:52:02, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:22:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:35:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:35:41, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[EVPN/170] 01:35:03
                       Indirect
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[EVPN/170] 02:31:17
                       Indirect
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[EVPN/170] 01:35:03
                       Indirect
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[EVPN/170] 00:07:54 
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:26:38, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:34:51, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:23:51, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:23:37, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 02:16:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[EVPN/170] 02:42:03
                       Indirect
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[EVPN/170] 00:13:19
                       Indirect
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[EVPN/170] 02:42:03
                       Indirect
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:32:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:52:03, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 02:16:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[EVPN/170] 00:13:21
                       Indirect
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:13:21, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:35:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:35:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:35:52, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:35:39, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:35:41, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:35:41, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 00:07:49
                       Indirect
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:35:03
                       Indirect
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[EVPN/170] 01:35:03
                       Indirect
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:07:49, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:34:51, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:34:50, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0

default-switch.evpn.0: 44 destinations, 44 routes (44 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[EVPN/170] 00:13:21
                       Indirect
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:13:20, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:13:21, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:22:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 02:39:55
                       Indirect
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[EVPN/170] 02:34:35
                       Indirect
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[EVPN/170] 00:12:49
                       Indirect
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[EVPN/170] 02:32:28
                       Indirect
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[EVPN/170] 00:12:50 
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:29:12, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:22:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[EVPN/170] 01:35:03
                       Indirect
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[EVPN/170] 02:31:17
                       Indirect
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[EVPN/170] 01:35:03
                       Indirect
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[EVPN/170] 00:07:54
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:26:38, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:23:51, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:23:37, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[EVPN/170] 02:42:03
                       Indirect
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[EVPN/170] 00:13:19
                       Indirect
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[EVPN/170] 02:42:03
                       Indirect
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
                                        
__default_evpn__.evpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 00:13:20
                       Indirect
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[EVPN/170] 00:13:21
                       Indirect
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:13:21, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
                                        
WAN.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 00:07:49
                       Indirect
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:35:03
                       Indirect
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[EVPN/170] 01:35:03
                       Indirect
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:07:49, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:32:26, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.6 via xe-0/0/0.0

{master:0}
```

SITE-B-LEAF-2:
```
root@SITE-B-LEAF-2> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      49         49          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.2            65000        915        313       0       0     1:52:58 Establ
  bgp.evpn.0: 49/49/49/0
  default-switch.evpn.0: 37/37/37/0
  __default_evpn__.evpn.0: 3/3/3/0
  WAN.evpn.0: 9/9/9/0
100.64.0.4            65102         29         26       0       1        8:44 Establ
  WAN.inet.0: 1/1/1/0

{master:0}
root@SITE-B-LEAF-2> show ethernet-switching vxlan-tunnel-end-point remote 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.8       lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.4       default-switch           576       vtep.32771   1783    RNVE      
    VNID          MC-Group-IP      
    100           0.0.0.0         
    400           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.5       default-switch           575       vtep.32770   1782    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    1000          0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode Flags
 10.200.0.7       default-switch           574       vtep.32769   1781    RNVE      
    VNID          MC-Group-IP      
    400           0.0.0.0         
    100           0.0.0.0               
    1000          0.0.0.0         

{master:0}
root@SITE-B-LEAF-2> show ethernet-switching vxlan-tunnel-end-point source 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode
<default>                 0   10.200.0.8       lo0.0    0  
    L2-RTT                   Bridge Domain              VNID     MC-Group-IP
    default-switch           v100+100                   100      0.0.0.0        
    default-switch           v1000+1000                 1000     0.0.0.0        
    default-switch           v300+300                   300      0.0.0.0        
    default-switch           v400+400                   400      0.0.0.0        

{master:0}
root@SITE-B-LEAF-2> show ethernet-switching table 

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 12 entries, 12 learned
Routing instance : default-switch
   Vlan                MAC                 MAC      Logical                SVLBNH/      Active
   name                address             flags    interface              VENH Index   source
   v100                00:00:5e:00:53:01   D        vtep.32771                          10.200.0.4                    
   v100                50:0f:0b:00:33:00   D        vtep.32771                          10.200.0.4                    
   v100                50:14:02:00:33:00   D        vtep.32771                          10.200.0.4                    
   v100                50:2f:a7:00:33:00   D        vtep.32771                          10.200.0.4                    
   v100                50:36:be:00:35:00   D        vtep.32769                          10.200.0.7                    
   v1000               02:05:86:71:cb:00   D        vtep.32769                          10.200.0.7                    
   v1000               aa:bb:cc:00:01:10   DR       esi.1785                            00:01:01:01:01:01:01:01:01:01 
   v1000               aa:bb:cc:00:02:10   DR       ae10.0               
   v300                50:0f:58:00:36:00   D        xe-0/0/2.0           
   v400                aa:bb:cc:00:01:10   DR       esi.1785                            00:01:01:01:01:01:01:01:01:01 
   v400                aa:bb:cc:00:02:10   DR       ae10.0               
   v400                aa:bb:cc:00:03:00   D        xe-0/0/3.0           

{master:0}
root@SITE-B-LEAF-2> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        00:00:5e:00:53:01  10.200.0.4                     Jun 06 13:26:47  100.64.10.1
     100        50:0f:0b:00:33:00  10.200.0.4                     Jun 06 13:26:47
     100        50:14:02:00:33:00  10.200.0.4                     Jun 06 13:26:47  100.64.10.10
     100        50:2f:a7:00:33:00  10.200.0.4                     Jun 06 13:26:47
     100        50:36:be:00:35:00  10.200.0.7                     Jun 06 13:26:47  100.64.10.20
     300        00:00:5e:00:53:01  irb.300                        Jun 06 11:56:43  100.64.30.1
     300        50:0f:58:00:36:00  xe-0/0/2.0                     Jun 06 12:00:11  100.64.30.10
     400        aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 12:28:39  100.64.40.1
     400        aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:38:05
     400        aa:bb:cc:00:03:00  xe-0/0/3.0                     Jun 06 12:24:15  100.64.40.3
     1000       02:05:86:71:04:00  irb.1000                       Jun 06 11:58:50  100.64.0.6
     1000       02:05:86:71:3a:00  10.200.0.5                     Jun 06 12:15:12  100.64.0.3
     1000       02:05:86:71:cb:00  10.200.0.7                     Jun 06 12:15:52  100.64.0.5
     1000       02:05:86:71:fc:00  10.200.0.4                     Jun 06 12:15:01  100.64.0.2
     1000       aa:bb:cc:00:01:10  00:01:01:01:01:01:01:01:01:01  Jun 06 13:22:58  100.64.0.1
     1000       aa:bb:cc:00:02:10  00:01:01:01:01:01:01:01:01:02  Jun 06 13:43:00  100.64.0.4

{master:0}
root@SITE-B-LEAF-2> show route 

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 01:53:14, metric 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.2/31      *[IS-IS/18] 01:53:14, metric 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.4/31      *[IS-IS/18] 01:53:14, metric 50
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 01:53:24, metric 20
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.8/31      *[Direct/0] 01:55:22
                    >  via xe-0/0/0.0
10.100.0.9/32      *[Local/0] 01:55:22
                       Local via xe-0/0/0.0
10.100.0.10/31     *[IS-IS/18] 01:53:24, metric 20
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.12/31     *[IS-IS/18] 01:53:14, metric 40
                    >  to 10.100.0.8 via xe-0/0/0.0
10.100.0.14/31     *[IS-IS/18] 01:53:14, metric 30
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.1/32      *[IS-IS/18] 01:53:14, metric 50
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.2/32      *[IS-IS/18] 01:53:24, metric 10
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.3/32      *[IS-IS/18] 01:53:14, metric 30
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.4/32      *[IS-IS/18] 01:53:14, metric 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.5/32      *[IS-IS/18] 01:53:14, metric 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.6/32      *[IS-IS/18] 01:53:14, metric 40
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.7/32      *[IS-IS/18] 01:53:14, metric 20
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.8/32      *[Direct/0] 01:57:39
                    >  via lo0.0
10.200.0.9/32      *[IS-IS/18] 01:53:14, metric 20
                    >  to 10.100.0.8 via xe-0/0/0.0
169.254.0.0/24     *[Direct/0] 01:57:39
                    >  via em1.0
169.254.0.2/32     *[Local/0] 01:57:39
                       Local via em1.0

:vxlan.inet.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.8/31      *[Direct/0] 01:55:22
                    >  via xe-0/0/0.0
10.100.0.9/32      *[Local/0] 01:55:22
                       Local via xe-0/0/0.0
10.200.0.4/32      *[Static/1] 01:53:14, metric2 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.5/32      *[Static/1] 01:53:14, metric2 60
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.7/32      *[Static/1] 01:53:14, metric2 20
                    >  to 10.100.0.8 via xe-0/0/0.0
10.200.0.8/32      *[Direct/0] 01:57:38
                    >  via lo0.0
169.254.0.0/24     *[Direct/0] 01:57:38
                    >  via em1.0
169.254.0.2/32     *[Local/0] 01:57:38
                       Local via em1.0

WAN.inet.0: 9 destinations, 16 routes (9 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[BGP/170] 00:09:00, localpref 100
                      AS path: 65102 I, validation-state: unverified
                    >  to 100.64.0.4 via irb.1000
                    [EVPN/170] 00:08:59
                    >  to 10.100.0.8 via xe-0/0/0.0
                    [EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
                    [EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
100.64.0.0/24      *[Direct/0] 01:36:02
                    >  via irb.1000
                    [EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
                    [EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
                    [EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
100.64.0.4/32      *[EVPN/7] 00:09:04
                    >  via irb.1000
100.64.0.6/32      *[Local/0] 01:36:02
                       Local via irb.1000
100.64.10.0/24     *[EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
                    [EVPN/170] 01:36:02 
                    >  to 10.100.0.8 via xe-0/0/0.0
100.64.20.0/24     *[EVPN/170] 01:36:02
                    >  to 10.100.0.8 via xe-0/0/0.0
100.64.30.0/24     *[Direct/0] 01:36:02
                    >  via irb.300
100.64.30.1/32     *[Local/0] 01:36:02
                       Local via irb.300
100.64.30.10/32    *[EVPN/7] 01:36:02
                    >  via irb.300

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0008/72                
                   *[Direct/0] 01:57:39
                    >  via lo0.0

mpls.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

19                 *[VPN/0] 01:36:02
                    >  via lsi.1 (WAN), Pop      
                                        
inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:400/128
                   *[Direct/0] 01:57:39
                    >  via lo0.0
ff02::2/128        *[INET6/0] 01:57:41
                       MultiRecv

WAN.inet6.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

ff02::2/128        *[INET6/0] 01:36:03
                       MultiRecv

bgp.evpn.0: 66 destinations, 66 routes (66 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:14:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:14:31, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 00:14:31
                       Indirect
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[EVPN/170] 00:14:32
                       Indirect
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:23:25, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:25:17, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 00:25:17, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:13:59, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:14:00, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 01:55:21
                       Indirect
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[EVPN/170] 01:55:18
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[EVPN/170] 01:30:23
                       Indirect
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[EVPN/170] 01:53:13
                       Indirect
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:23:25, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:37:02, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:36:51, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:25:17, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 00:25:17, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:36:12, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:09:04, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[EVPN/170] 01:36:01
                       Indirect
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[EVPN/170] 01:51:53
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[EVPN/170] 01:27:49
                       Indirect         
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[EVPN/170] 01:36:01
                       Indirect
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 00:25:16, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:25:01, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:24:47, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 00:25:17, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:14:29, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 01:53:14, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[EVPN/170] 01:57:37
                       Indirect
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[EVPN/170] 01:33:58
                       Indirect
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[EVPN/170] 01:57:37
                       Indirect
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 01:53:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 01:53:15, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:14:32, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[EVPN/170] 00:14:33
                       Indirect
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:37:01, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:37:04, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:37:04, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:36:50, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:36:53, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:36:53, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:09:00, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:36:14, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:36:14, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 00:09:01
                       Indirect
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:36:03
                       Indirect
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[EVPN/170] 01:36:03
                       Indirect

default-switch.evpn.0: 49 destinations, 49 routes (49 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:14:31, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:14:32, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[EVPN/170] 00:14:33
                       Indirect
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:14:00, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:14:01, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 01:55:22
                       Indirect
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[EVPN/170] 01:55:19
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[EVPN/170] 01:30:24
                       Indirect
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[EVPN/170] 01:53:14
                       Indirect
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:09:05, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[EVPN/170] 01:36:02
                       Indirect
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[EVPN/170] 01:51:54
                       Indirect
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[EVPN/170] 01:27:50
                       Indirect
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[EVPN/170] 01:36:02
                       Indirect
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:14:30, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[EVPN/170] 01:57:38
                       Indirect
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[EVPN/170] 01:33:59
                       Indirect
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[EVPN/170] 01:57:38
                       Indirect

__default_evpn__.evpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[EVPN/170] 00:14:32
                       Indirect
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:14:32, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[EVPN/170] 00:14:33
                       Indirect

WAN.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:09:00, localpref 100, from 10.200.0.2
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 00:25:13, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.8 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[EVPN/170] 00:09:01
                       Indirect
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[EVPN/170] 01:36:03
                       Indirect
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[EVPN/170] 01:36:03
                       Indirect

{master:0}
```

SITE-B-SPINE-1:
```
root@SITE-B-SPINE-1> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 3 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                      71         71          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.7            65000        409        575       0       0     2:41:29 Establ
  bgp.evpn.0: 18/18/18/0
10.200.0.8            65000        317        918       0       0     1:54:15 Establ
  bgp.evpn.0: 17/17/17/0
10.200.0.9            65000        395        410       0       0     2:32:06 Establ
  bgp.evpn.0: 36/36/36/0

{master:0}
root@SITE-B-SPINE-1> ... vxlan-tunnel-end-point remote                       

{master:0}
root@SITE-B-SPINE-1> ... vxlan-tunnel-end-point source                       

{master:0}
root@SITE-B-SPINE-1> show ethernet-switching table 

{master:0}
root@SITE-B-SPINE-1> show evpn database 

{master:0}
root@SITE-B-SPINE-1> show route 

inet.0: 22 destinations, 22 routes (22 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[IS-IS/18] 02:32:17, metric 50
                    >  to 10.100.0.11 via xe-0/0/2.0
10.100.0.2/31      *[IS-IS/18] 02:32:17, metric 50
                    >  to 10.100.0.11 via xe-0/0/2.0
10.100.0.4/31      *[IS-IS/18] 02:32:17, metric 40
                    >  to 10.100.0.11 via xe-0/0/2.0
10.100.0.6/31      *[Direct/0] 02:54:33
                    >  via xe-0/0/0.0
10.100.0.6/32      *[Local/0] 02:54:33
                       Local via xe-0/0/0.0
10.100.0.8/31      *[Direct/0] 01:54:51
                    >  via xe-0/0/1.0
10.100.0.8/32      *[Local/0] 01:54:51
                       Local via xe-0/0/1.0
10.100.0.10/31     *[Direct/0] 02:54:34
                    >  via xe-0/0/2.0
10.100.0.10/32     *[Local/0] 02:54:34
                       Local via xe-0/0/2.0
10.100.0.12/31     *[IS-IS/18] 02:32:17, metric 30
                    >  to 10.100.0.11 via xe-0/0/2.0
10.100.0.14/31     *[IS-IS/18] 02:32:27, metric 20
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.1/32      *[IS-IS/18] 02:32:17, metric 40
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.2/32      *[Direct/0] 02:56:35
                    >  via lo0.0
10.200.0.3/32      *[IS-IS/18] 02:32:17, metric 20
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.4/32      *[IS-IS/18] 02:32:17, metric 50
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.5/32      *[IS-IS/18] 02:32:17, metric 50
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.6/32      *[IS-IS/18] 02:32:17, metric 30
                    >  to 10.100.0.11 via xe-0/0/2.0
10.200.0.7/32      *[IS-IS/18] 02:42:17, metric 10
                    >  to 10.100.0.7 via xe-0/0/0.0
10.200.0.8/32      *[IS-IS/18] 01:54:43, metric 10
                    >  to 10.100.0.9 via xe-0/0/1.0
10.200.0.9/32      *[IS-IS/18] 02:32:27, metric 10
                    >  to 10.100.0.11 via xe-0/0/2.0
169.254.0.0/24     *[Direct/0] 02:56:35
                    >  via em1.0
169.254.0.2/32     *[Local/0] 02:56:35  
                       Local via em1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0002/72                
                   *[Direct/0] 02:56:35
                    >  via lo0.0

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fe80::205:860f:fc71:3f00/128
                   *[Direct/0] 02:56:35
                    >  via lo0.0
ff02::2/128        *[INET6/0] 02:56:43
                       MultiRecv

bgp.evpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.200.0.4:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:19:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
1:10.200.0.4:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:19:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
1:10.200.0.5:0::010101010101010101::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 02:19:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
1:10.200.0.5:1::010101010101010101::0/192 AD/EVI        
                   *[BGP/170] 02:19:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
1:10.200.0.7:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:15:48, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
1:10.200.0.7:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:15:49, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
1:10.200.0.8:0::010101010101010102::FFFF:FFFF/192 AD/ESI        
                   *[BGP/170] 00:15:49, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
1:10.200.0.8:1::010101010101010102::0/192 AD/EVI        
                   *[BGP/170] 00:15:49, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::100::50:0f:0b:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::100::50:14:02:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::100::50:2f:a7:00:33:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 01:24:43, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::200::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::200::50:11:2b:00:34:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.7:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 02:41:45, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00/304 MAC/IP        
                   *[BGP/170] 02:37:04, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::400::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:15:17, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00/304 MAC/IP        
                   *[BGP/170] 02:34:56, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10/304 MAC/IP        
                   *[BGP/170] 00:15:18, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01/304 MAC/IP        
                   *[BGP/170] 01:54:32, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00/304 MAC/IP        
                   *[BGP/170] 01:54:32, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00/304 MAC/IP        
                   *[BGP/170] 01:31:41, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00/304 MAC/IP        
                   *[BGP/170] 01:54:31, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.4:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::100::50:14:02:00:33:00::100.64.10.10/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::400::aa:bb:cc:00:01:10::100.64.40.1/304 MAC/IP        
                   *[BGP/170] 01:24:43, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::1000::02:05:86:71:fc:00::100.64.0.2/304 MAC/IP        
                   *[BGP/170] 01:38:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.4:1::1000::aa:bb:cc:00:01:10::100.64.0.1/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::200::00:00:5e:00:53:01::100.64.20.1/304 MAC/IP        
                   *[BGP/170] 01:38:10, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::200::50:11:2b:00:34:00::100.64.20.10/304 MAC/IP        
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.5:1::1000::02:05:86:71:3a:00::100.64.0.3/304 MAC/IP        
                   *[BGP/170] 01:38:10, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
2:10.200.0.7:1::100::00:00:5e:00:53:01::100.64.10.1/304 MAC/IP        
                   *[BGP/170] 01:37:31, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::100::50:36:be:00:35:00::100.64.10.20/304 MAC/IP        
                   *[BGP/170] 02:33:45, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::1000::02:05:86:71:cb:00::100.64.0.5/304 MAC/IP        
                   *[BGP/170] 01:37:31, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.7:1::1000::aa:bb:cc:00:02:10::100.64.0.4/304 MAC/IP        
                   *[BGP/170] 00:10:22, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
2:10.200.0.8:1::300::00:00:5e:00:53:01::100.64.30.1/304 MAC/IP        
                   *[BGP/170] 01:37:19, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::300::50:0f:58:00:36:00::100.64.30.10/304 MAC/IP        
                   *[BGP/170] 01:53:11, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::400::aa:bb:cc:00:03:00::100.64.40.3/304 MAC/IP        
                   *[BGP/170] 01:29:07, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
2:10.200.0.8:1::1000::02:05:86:71:04:00::100.64.0.6/304 MAC/IP        
                   *[BGP/170] 01:37:19, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 02:19:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.4:1::400::10.200.0.4/248 IM            
                   *[BGP/170] 01:26:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.4:1::1000::10.200.0.4/248 IM            
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.5:1::200::10.200.0.5/248 IM            
                   *[BGP/170] 02:19:20, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.5:1::400::10.200.0.5/248 IM            
                   *[BGP/170] 01:26:05, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.5:1::1000::10.200.0.5/248 IM            
                   *[BGP/170] 02:19:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
3:10.200.0.7:1::100::10.200.0.7/248 IM            
                   *[BGP/170] 02:41:45, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
3:10.200.0.7:1::400::10.200.0.7/248 IM            
                   *[BGP/170] 00:15:47, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
3:10.200.0.7:1::1000::10.200.0.7/248 IM            
                   *[BGP/170] 02:41:45, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
3:10.200.0.8:1::300::10.200.0.8/248 IM            
                   *[BGP/170] 01:54:32, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
3:10.200.0.8:1::400::10.200.0.8/248 IM            
                   *[BGP/170] 01:35:15, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
3:10.200.0.8:1::1000::10.200.0.8/248 IM            
                   *[BGP/170] 01:54:32, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
4:10.200.0.4:0::010101010101010101:10.200.0.4/296 ES            
                   *[BGP/170] 02:19:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
4:10.200.0.5:0::010101010101010101:10.200.0.5/296 ES            
                   *[BGP/170] 02:19:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
4:10.200.0.7:0::010101010101010102:10.200.0.7/296 ES            
                   *[BGP/170] 00:15:49, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
4:10.200.0.8:0::010101010101010102:10.200.0.8/296 ES            
                   *[BGP/170] 00:15:49, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
5:10.200.0.4:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:38:19, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.4:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:38:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.4:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:38:21, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.5:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 01:38:08, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.5:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:38:10, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.5:2000::0::100.64.20.0::24/248               
                   *[BGP/170] 01:38:10, localpref 100, from 10.200.0.9
                      AS path: 65020 65001 I, validation-state: unverified
                    >  to 10.100.0.11 via xe-0/0/2.0
5:10.200.0.7:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:10:18, localpref 100, from 10.200.0.7
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:37:31, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
5:10.200.0.7:2000::0::100.64.10.0::24/248               
                   *[BGP/170] 01:37:31, localpref 100, from 10.200.0.7
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.7 via xe-0/0/0.0
5:10.200.0.8:2000::0::0.0.0.0::0/248               
                   *[BGP/170] 00:10:18, localpref 100, from 10.200.0.8
                      AS path: 65102 I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.0.0::24/248               
                   *[BGP/170] 01:37:19, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0
5:10.200.0.8:2000::0::100.64.30.0::24/248               
                   *[BGP/170] 01:37:19, localpref 100, from 10.200.0.8
                      AS path: I, validation-state: unverified
                    >  to 10.100.0.9 via xe-0/0/1.0

{master:0}
```
