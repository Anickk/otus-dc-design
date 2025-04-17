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
### PE-1
