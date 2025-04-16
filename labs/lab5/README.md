# VxLAN. EVPN L2
---
## Конфигурация Overlay

В качестве протокола underlay был выбран IS-IS и настроен ранее.\
Пример конфигурации приводится для LEAF-1 и SPINE-1, на парных железках конфигурация схожа.
---
### Leaf-1:

Для начала настроим параметры для балансировки и построения bgp соседства, включим ECMP в VXLAN и Ingress replication для EVPN:
```
policy-options {
    policy-statement load-balancing-policy {
        then {
            load-balance per-packet;
        }
    }
}
routing-options {
    router-id 10.200.0.3;
    autonomous-system 65000;
    forwarding-table {
        export load-balancing-policy;
        dynamic-list-next-hop;
        chained-composite-next-hop {
            ingress {
                evpn;
            }
        }                               
```
Настроим RD и укажем VTEP интерфейс:
```
switch-options {
    vtep-source-interface lo0.0;
    route-distinguisher 10.200.0.3:1;
    vrf-target {
        target:1:1;
        auto;
    }                                   
}
```
Тут стоит отметить, что target:1:1 не нужно задавать при исполтьзовании ibgp в overlay, но для ebgp оно понадобится, т.к. AS будут разные а таргет формируется так - target:<local-AS>:<VNI>. Я буду использовать ibgp, но для наглядности оставлю команду.

Настройка evpn, укажем инкапсуляцию, режим работы, и разрешим инкапсуляцию для всех vni, чтобы не задавать их в ручную:
```
protocols {
    evpn {                              
            encapsulation vxlan;
            multicast-mode ingress-replication;
            extended-vni-list all;
        }
}
```

Настройка bgp overlay:
```
protocols {
    bgp {
        group OVERLAY {
            type internal;
            local-address 10.200.0.3;
            family evpn {
                signaling;
            }
            neighbor 10.200.0.1 {
                description SPINE-1;
            }
            neighbor 10.200.0.2 {
                description SPINE-2;
            }
        }
    }
```
---
### Spine-1:
Тут мы настраиваем только параметры для bgp, балансировку и сам overlay:
```
policy-options {
    policy-statement load-balancing-policy {
        then {
            load-balance per-packet;
        }
    }
}
routing-options {
    router-id 10.200.0.1;
    autonomous-system 65000;
    forwarding-table {
        export load-balancing-policy;   
    }
}
protocols {
    bgp {
        group OVERLAY {
            type internal;
            local-address 10.200.0.1;
            family evpn {
                signaling;
            }
            cluster 10.200.0.1;
            neighbor 10.200.0.3 {
                description LEAF-1;
            }
            neighbor 10.200.0.4 {
                description LEAF-2;
            }
        }
    }
```
cluster 10.200.0.1 - включает RR. на другом spine другой кластер id.

Проверка bgp соседств на leaf-1:
```
root@LEAF-1> show bgp summary 
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.evpn.0           
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.200.0.1            65000        185        183       0       0     1:15:46 Establ
  bgp.evpn.0: 0/0/0/0
  default-switch.evpn.0: 0/0/0/0
  __default_evpn__.evpn.0: 0/0/0/0
10.200.0.2            65000        181        179       0       0     1:14:08 Establ
  bgp.evpn.0: 0/0/0/0
  default-switch.evpn.0: 0/0/0/0
  __default_evpn__.evpn.0: 0/0/0/0

{master:0}
```
Соседства поднялись, можно переходить к следующему этапу

---
## Конфигурация l2 vni 

Пример будет представлен для LEAF-1, на втором коммутаторе конфиг аналогичен.

Настроим порт в сторону сервера и vlan с vni:
```
vlans {
    v100 {
        description L2TEST;
        vlan-id 100;
        vxlan {
            vni 100;
        }
    }
}
interfaces {                            
    xe-0/0/2 {
        description HOST-1;
        unit 0 {
            family ethernet-switching {
                interface-mode access;  
                vlan {
                    members v100;
                }
            }
        }
    }
```

На ВМ настроены адреса 10.10.10.200 и 10.10.10.100, проверяем связность:
```
gns3@box:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN 
    link/ether 46:1a:a4:f8:9e:60 brd ff:ff:ff:ff:ff:ff
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip_vti0@NONE: <NOARP> mtu 1364 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 50:9d:51:00:09:00 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.250/24 brd 10.10.10.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::529d:51ff:fe00:900/64 scope link 
       valid_lft forever preferred_lft forever
gns3@box:~$ sudo ifconfig eth0 10.10.10.100 netmask 255.255.255.0
gns3@box:~$ ping 10.10.10.200
PING 10.10.10.200 (10.10.10.200): 56 data bytes
64 bytes from 10.10.10.200: seq=0 ttl=64 time=520.690 ms
64 bytes from 10.10.10.200: seq=1 ttl=64 time=436.920 ms
64 bytes from 10.10.10.200: seq=2 ttl=64 time=464.149 ms
64 bytes from 10.10.10.200: seq=3 ttl=64 time=315.563 ms
64 bytes from 10.10.10.200: seq=4 ttl=64 time=331.791 ms
64 bytes from 10.10.10.200: seq=5 ttl=64 time=304.921 ms
64 bytes from 10.10.10.200: seq=6 ttl=64 time=322.065 ms
64 bytes from 10.10.10.200: seq=7 ttl=64 time=351.362 ms
64 bytes from 10.10.10.200: seq=8 ttl=64 time=310.064 ms
64 bytes from 10.10.10.200: seq=9 ttl=64 time=239.116 ms
64 bytes from 10.10.10.200: seq=10 ttl=64 time=234.422 ms
64 bytes from 10.10.10.200: seq=11 ttl=64 time=451.534 ms
64 bytes from 10.10.10.200: seq=12 ttl=64 time=339.804 ms
64 bytes from 10.10.10.200: seq=13 ttl=64 time=357.276 ms
64 bytes from 10.10.10.200: seq=14 ttl=64 time=380.169 ms
64 bytes from 10.10.10.200: seq=15 ttl=64 time=297.154 ms
64 bytes from 10.10.10.200: seq=16 ttl=64 time=333.167 ms
64 bytes from 10.10.10.200: seq=17 ttl=64 time=259.404 ms
64 bytes from 10.10.10.200: seq=18 ttl=64 time=293.853 ms
^C
--- 10.10.10.200 ping statistics ---
20 packets transmitted, 19 packets received, 5% packet loss
round-trip min/avg/max = 234.422/344.390/520.690 ms
```

Связность есть, заглянем в таблицу маршрутизации на коммутаторе, будем смотреть в default-switch чтобы видеть и локальные маршруты:
```
root@LEAF-1> show route table default-switch.evpn.0 

default-switch.evpn.0: 4 destinations, 6 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2:10.200.0.3:1::100::50:9d:51:00:09:00/304 MAC/IP        
                   *[EVPN/170] 00:06:28
                      Indirect
2:10.200.0.4:1::100::50:32:93:00:08:00/304 MAC/IP        
                   *[BGP/170] 00:06:25, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0
                    [BGP/170] 00:06:25, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0
3:10.200.0.3:1::100::10.200.0.3/248 IM            
                   *[EVPN/170] 00:06:33
                      Indirect
3:10.200.0.4:1::100::10.200.0.4/248 IM            
                   *[BGP/170] 00:06:27, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0
                    [BGP/170] 00:06:27, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0

{master:0}
```
Тут мы видим локальные и удаленные type 2 и 3 маршруты, но в type 2 видим только mac, и не видим ip. Однако длина префикса 304 бита, из чего можно сделать вывод что адрес включен в него, просто он не интерпритируется в читаемый вид данной платформой. Скорее всего, это обусловлено тем, что в виртуальной среде используются vqfx с версией junos 18, в которой нельзя включить arp-suppression без использования irb. В 20+ версиях оно включено по умолчанию, и поидее, там должны быть и маршруты 2ого типа с адресами.

Заглянем в evpn:
```
root@LEAF-1> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        50:32:93:00:08:00  10.200.0.4                     Apr 15 19:36:26
     100        50:9d:51:00:09:00  xe-0/0/2.0                     Apr 15 19:36:21

{master:0}
```

Посмотрим в мак тиаблицу:
```
root@LEAF-1# run show ethernet-switching table vlan-id 100 

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 3 entries, 3 learned
Routing instance : default-switch
   Vlan                MAC                 MAC      Logical                Active
   name                address             flags    interface              source
   v100                50:32:93:00:08:00   D        vtep.32769             10.200.0.4                    
   v100                50:9d:51:00:09:00   D        xe-0/0/2.0
```

Как видно, все необходимые маршруты и мак адреса извесны, трафик между ВМ ходит корректно.
