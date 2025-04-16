# VxLAN. EVPN L3
---

Будут использованы следующие vlan и сети:
|VLAN|УСТРОЙСТВО|АДРЕС|
|---|---|---|
|100|LEAF-1|10.10.10.1|
|100|VM-1|10.10.10.200|
|200|LEAF-2|10.10.20.1|
|200|VM-2|10.10.20.200|

## Схема
![img_1.png](scheme.png)

## Конфигурация L3VNI
Leaf-1:
```
routing-instances {
    WAN {
        instance-type vrf;
        interface irb.100;
        route-distinguisher 10.200.0.3:100;
        vrf-target target:65001:1;
        vrf-table-label;
        routing-options {
            multipath {
                vpn-unequal-cost equal-external-internal;
            }
        }
        protocols {
            evpn {
                ip-prefix-routes {
                    advertise direct-nexthop;
                    encapsulation vxlan;
                    vni 65001;
                }
            }
        }                               
    }
}
```
Можно использовать instance-type evpn если нужны bridge-domains.
Назначаем rd, vni, включаем мультипас. vrf-table-label - назначает mpls метку чтобы маршруты попадали в нужную таблицу и балансировка работала корректно.
А также сразу включим irb.100 в vrf, на leaf-2 будет irb.200.

Настроим l3 интерфейс и привяжем его к vlan:
```
interfaces { 
    irb {
        unit 100 {
            family inet {
                address 10.10.10.1/24;
            }
            mac 00:00:5e:00:53:01;
        }
    }
}
vlans {
    v100 {
        l3-interface irb.100;
}                    
```
Задаем мак чтобы в дальнейшем можно было использовать anycast gateway.
На Leaf-2 соответственно будет использоваться сеть 10.10.20.0/24 и vlan 200.

Проверяем связность между ВМ:
```
gns3@box:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN 
    link/ether ba:f5:33:fd:1c:ee brd ff:ff:ff:ff:ff:ff
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip_vti0@NONE: <NOARP> mtu 1364 qdisc noop state DOWN 
    link/ipip 0.0.0.0 brd 0.0.0.0
5: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 50:c0:14:00:09:00 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.200/24 brd 10.10.10.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::52c0:14ff:fe00:900/64 scope link 
       valid_lft forever preferred_lft forever
gns3@box:~$ ping 10.10.20.200
PING 10.10.20.200 (10.10.20.200): 56 data bytes
64 bytes from 10.10.20.200: seq=0 ttl=63 time=423.816 ms
64 bytes from 10.10.20.200: seq=1 ttl=63 time=350.039 ms
64 bytes from 10.10.20.200: seq=2 ttl=63 time=372.445 ms
64 bytes from 10.10.20.200: seq=3 ttl=63 time=275.377 ms
64 bytes from 10.10.20.200: seq=4 ttl=63 time=386.182 ms
^C
--- 10.10.20.200 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 275.377/361.571/423.816 ms
```
Связность есть, посмотрим маршруты на коммутаторе, заглянем в таблицы WAN и default-switch:
```
WAN.evpn.0: 2 destinations, 3 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.3:100::0::10.10.10.0::24/248               
                   *[EVPN/170] 00:12:20
                      Indirect
5:10.200.0.4:100::0::10.10.20.0::24/248               
                   *[BGP/170] 00:11:44, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0
                    [BGP/170] 00:11:27, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0

default-switch.evpn.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2:10.200.0.3:1::100::00:00:5e:00:53:01/304 MAC/IP        
                   *[EVPN/170] 00:14:46
                      Indirect
2:10.200.0.3:1::100::50:9d:51:00:09:00/304 MAC/IP        
                   *[EVPN/170] 00:12:00
                      Indirect
2:10.200.0.3:1::100::50:c0:14:00:09:00/304 MAC/IP        
                   *[EVPN/170] 00:01:21
                      Indirect
2:10.200.0.3:1::100::00:00:5e:00:53:01::10.10.10.1/304 MAC/IP        
                   *[EVPN/170] 00:14:46
                      Indirect
2:10.200.0.3:1::100::50:c0:14:00:09:00::10.10.10.200/304 MAC/IP        
                   *[EVPN/170] 00:00:46
                      Indirect
3:10.200.0.3:1::100::10.200.0.3/248 IM            
                   *[EVPN/170] 00:12:19
                      Indirect
```

Мы видим type 5 маршруты для локальной и удаленной сети, что говорит нам о том что используется симметричная модель irb.\
Также теперь мы видим ip в маршрутах 2ого типа, потому что теперь у нас создан irb и arp suppression включен.\
Заглянем в evpn:
```
root@LEAF-1> show evpn database 
Instance: default-switch
VLAN  DomainId  MAC address        Active source                  Timestamp        IP address
     100        00:00:5e:00:53:01  irb.100                        Apr 15 20:40:00  10.10.10.1
     100        50:c0:14:00:09:00  xe-0/0/2.0                     Apr 15 20:51:32  10.10.10.200
```
Теперь мы видим адреса из arp запросов.
Связность есть, все работает.
