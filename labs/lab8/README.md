# Оптимизация таблиц маршрутизации

В данной работе поместим VM каждую в свой VRF и настроим маршрутизацию между ними через внешний маршрутизатор.\
Чтобы не ломать прошлые наработки добавим еще один LEAF и ISP.

## Схема:
![img_1.png](scheme1.png)

Не стану расписывать подключение к SPINE для LEAF-3, возьму следующие подсети из адресного плана первой лабы, отмечу только что адрес для vtep будет 10.200.0.5.\
Адресацию для стыков будем использовать следующую:
|VRF|LEAF-3|ISP-3|
|-----|------|-----|
|1|100.100.1.2/24|100.100.1.1/24|
|2|100.100.2.2/24|100.100.2.1/24|

## Кофигурация:

Выполним базовую конфигурацию ISP для выхода в интернет и маршрутизации между VRF, использовать будем статику и саб интерфейсы:

### ISP-3:
```
interface Ethernet0/0
 no ip address

interface Ethernet0/0.300
 description VRF-1
 encapsulation dot1Q 300
 ip address 100.100.1.1 255.255.255.0
 ip nat inside
 ip virtual-reassembly in

interface Ethernet0/0.400
 description VRF-2
 encapsulation dot1Q 400
 ip address 100.100.2.1 255.255.255.0
 ip nat inside
 ip virtual-reassembly in

interface Ethernet0/1
 ip address dhcp
 ip nat outside

ip route 100.64.10.0 255.255.255.0 100.100.1.2
ip route 100.64.20.0 255.255.255.0 100.100.2.2

access-list 100 permit ip any any

ip nat inside source list 100 interface ethernet 0/1 overload

```

Создадим VRF на LEAF-1 и LEAF-2 и переместим в них соответствующие irb интерфейсы из WAN vrf:

### Leaf-1:
```
routing-instances {
    VRF-1 {
        instance-type vrf;
        interface irb.100;
        route-distinguisher 10.200.0.3:101;
        vrf-target target:65002:1;
        vrf-table-label;
        routing-options {
            multipath {
                vpn-unequal-cost equal-external-internal;
            }
            auto-export;
        }
        protocols {
            evpn {
                ip-prefix-routes {      
                    advertise direct-nexthop;
                    encapsulation vxlan;
                    vni 65002;
                }
            }
        }
    }
```

### Leaf-2:
```
routing-instances {
    VRF-2 {
        instance-type vrf;
        interface irb.200;
        route-distinguisher 10.200.0.4:102;
        vrf-target target:65003:1;
        vrf-table-label;
        routing-options {
            multipath {
                vpn-unequal-cost equal-external-internal;
            }
            auto-export;
        }
        protocols {
            evpn {
                ip-prefix-routes {      
                    advertise direct-nexthop;
                    encapsulation vxlan;
                    vni 65003;
                }
            }
        }
    }
```

Создадим на Leaf-3 оба VRF и пропишем статические default маршруты в каждой VRF:

### Leaf-3:
```
routing-instances {
    VRF-1 {
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 100.100.1.1;
            }
            multipath {
                vpn-unequal-cost equal-external-internal;
            }
            auto-export;
        }
        protocols {
            evpn {
                ip-prefix-routes {
                    advertise direct-nexthop;
                    encapsulation vxlan;
                    vni 65002;
                }
            }
        }
        instance-type vrf;
        interface xe-0/0/2.300;
        route-distinguisher 10.200.0.5:101;
        vrf-target target:65002:1;
        vrf-table-label;
    }
    VRF-2 {
        routing-options {
            static {
                route 0.0.0.0/0 next-hop 100.100.2.1;
            }
            multipath {
                vpn-unequal-cost equal-external-internal;
            }
            auto-export;
        }
        protocols {
            evpn {
                ip-prefix-routes {      
                    advertise direct-nexthop;
                    encapsulation vxlan;
                    vni 65003;
                }
            }
        }
        instance-type vrf;
        interface xe-0/0/2.400;
        route-distinguisher 10.200.0.5:102;
        vrf-target target:65003:1;
        vrf-table-label;
    }
}
```

l2 vni мы при этом не создаем, будем использовать только type 5 маршруты.

Проверяем таблицу маршрутизации на Leaf-1:
```
root@LEAF-1> show route
***
VRF-1.inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[EVPN/170] 00:42:08 
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0
100.64.10.0/24     *[Direct/0] 00:42:08
                    > via irb.100
100.64.10.1/32     *[Local/0] 00:42:08
                      Local via irb.100
100.64.10.200/32   *[EVPN/7] 00:42:04
                    > via irb.100
100.100.1.0/24     *[EVPN/170] 00:42:08
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0

VRF-1.evpn.0: 3 destinations, 5 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

5:10.200.0.3:101::0::100.64.10.0::24/248               
                   *[EVPN/170] 00:42:08
                      Indirect
5:10.200.0.5:101::0::0.0.0.0::0/248               
                   *[BGP/170] 00:42:08, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0
                    [BGP/170] 00:42:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0
5:10.200.0.5:101::0::100.100.1.0::24/248               
                   *[BGP/170] 00:42:08, localpref 100, from 10.200.0.1
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0
                    [BGP/170] 00:42:07, localpref 100, from 10.200.0.2
                      AS path: I, validation-state: unverified
                      to 10.100.0.0 via xe-0/0/0.0
                    > to 10.100.0.4 via xe-0/0/1.0
```

Как видим, маршруты имеются. На Leaf-2 ситуация аналогична для своей VRF. Проверим свзность VM-1 с внешним миром и VM-2.

```

```
