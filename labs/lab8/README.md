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
