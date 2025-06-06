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
## Просмотр данных с коммутаторов:

Команды используемые для анализа:\
show bgp summary\
show ethernet-switching vxlan-tunnel-end-point remote\
show ethernet-switching vxlan-tunnel-end-point source\
show ethernet-switching table\
show evpn database\
show route
