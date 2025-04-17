# VxLAN. Аналоги VPC
---

В данной лабе рассмотрим подключения с ESI multihoming, подключим PE маршрутизаторы к нашим LEAF с помощью LACP, сэмулируем подключения к ISP и выпустим VM в интернет через ISP.

В качестве PE - Juniper vMX.\
В качестве ISP - Cisco IOL.

## Схема:
![img_1.png](scheme.png)

## Адресация:

Для эмуляции белого адресного пространства, везде будут использованы сети из диапазона 100.64.0.0/10.
| Сеть               | Назначение                 | Устройство | IP-адрес |
|--------------------|----------------------------|------------|----------|
| 100.64.10.0/24     | Сеть VM1 (vlan 100)        | VM1        | .200     |
|                    |                            | LEAF-1     | .1       |
| 100.64.20.0/24     | Сеть VM2 (vlan 200)        | VM2        | .200     |
|                    |                            | LEAF-2     | .1       |
| 100.100.100.0/24   | LEAF ↔ PE1 (vlan 1000)     | PE1        | .1       |
|                    |                            | LEAF-1     | .2       |
|                    |                            | LEAF-2     | .3       |
| 100.100.200.0/24   | LEAF ↔ PE2 (vlan 2000)     | PE2        | .1       |
|                    |                            | LEAF-1     | .2       |
|                    |                            | LEAF-2     | .3       |
| 100.111.100.0/24   | PE1 ↔ ISP1                 | ISP1       | .1       |
|                    |                            | PE1        | .2       |
| 100.112.200.0/24   | PE2 ↔ ISP2                 | ISP2       | .1       |
|                    |                            | PE2        | .2       |

## Конфигурация:

Подготовим роутеры ISP, адрес на uplink они получают по dhcp, нам требуется настроить даунлинк, nat и маршрутизацию во внутренние сети:
### ISP-1
```
ip nat inside source list 100 interface Ethernet0/1 overload
interface Ethernet0/1
 ip address dhcp
 ip nat outside
interface Ethernet0/0
 description PE-1
 ip address 100.111.100.1 255.255.255.0
 ip nat inside
ip route 100.64.0.0 255.192.0.0 100.111.100.2
```
### ISP-2
```
ip nat inside source list 100 interface Ethernet0/1 overload
interface Ethernet0/1
 ip address dhcp
 ip nat outside
interface Ethernet0/0
 description PE-2
 ip address 100.112.200.1 255.255.255.0
 ip nat inside
ip route 100.64.0.0 255.192.0.0 100.112.200.2
```

Настроим PE маршрутизаторы. Настроим стык с ISP, статический default, подготовим агрегированный интерфейс в сторону лифов, и преднастроим bgp для экспорта дефолта и импорта внутренних сетей:
### PE-1/2

Пример приводится для PE-1, на PE-2 конфиг идентичный, за исключением адресации и AS.
```
chassis {
    aggregated-devices {
        ethernet {
            device-count 100;           
        }
    }
}
///Эта команда требуется для возможности использхования агрегатов на JunOS
interfaces {
    ge-0/0/0 {
        description LEAF-1;
        gigether-options {
            802.3ad ae0;
        }
    }
    ge-0/0/1 {
        description LEAF-2;
        gigether-options {
            802.3ad ae0;
        }
    }
    ge-0/0/2 {
        description ISP;
        unit 0 {
            family inet {
                address 100.111.100.2/24;
            }
        }                               
    }
    ae0 {
        description LEAF;
        aggregated-ether-options {
            lacp {
                active;
            }
        }
        unit 0 {
            family inet {
                address 100.100.100.1/24;
            }
        }
    }
}
routing-options {
    static {
        route 0.0.0.0/0 next-hop 100.111.100.1;
    }
    autonomous-system 65100;
}
protocols {
    bgp {                               
        group LEAF {
            type external;
            family inet {
                unicast;
            }
            export DEFAULT;
            multipath multiple-as;
            neighbor 100.100.100.2 {
                description LEAF-1;
                peer-as 65001;
            }
            neighbor 100.100.100.3 {
                description LEAF-2;
                peer-as 65002;
            }
        }
    }
}
///Испотзовать будем eBGP
policy-options {
    policy-statement DEFAULT {
        term DEFAUL {
            from {                      
                protocol static;
                route-filter 0.0.0.0/0 exact;
            }
            then accept;
        }
        then reject;
    }
}
///Экспортируется только default route, на импорт политика не требуется, по умолчанию принимается все.
```
