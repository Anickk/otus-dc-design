# Построение Underlay сети на OSPF
---
## Конфигурация

Конфигурация на всех коммутаторах идентичная:
```
protocols ospf {
    area 0.0.0.0 {
        interface xe-0/0/0.0;
        interface xe-0/0/1.0;
        interface lo0.0 {
            passive;
        }
    }
}
```
Router id, автоматически берется с адреса Lo0, вручную настраивать в данном сценарии не требуется.

---
## Проверка

Проверяем статус соседей:

LEAF-1:
```
root@LEAF-1> show ospf neighbor 
Address          Interface              State     ID               Pri  Dead
10.100.0.0       xe-0/0/0.0             Full      10.200.0.1       128    37
10.100.0.4       xe-0/0/1.0             Full      10.200.0.2       128    35
```
LEAF-2:
```
root@LEAF-2> show ospf neighbor 
Address          Interface              State     ID               Pri  Dead
10.100.0.2       xe-0/0/0.0             Full      10.200.0.1       128    39
10.100.0.6       xe-0/0/1.0             Full      10.200.0.2       128    37
```
Видим по два соседа в статусе Full на каждом LEAF, соотетственно со стороны SPINE проверять не имеет смысла.

Проверяем распрространение маршрутов lo интерфейсов:

LEAF-1:
```
root@LEAF-1> show route protocol ospf 

inet.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.2/31      *[OSPF/10] 00:15:02, metric 2
                    > to 10.100.0.0 via xe-0/0/0.0
10.100.0.6/31      *[OSPF/10] 00:15:02, metric 2
                    > to 10.100.0.4 via xe-0/0/1.0
10.200.0.1/32      *[OSPF/10] 00:15:02, metric 1
                    > to 10.100.0.0 via xe-0/0/0.0
10.200.0.2/32      *[OSPF/10] 00:15:02, metric 1
                    > to 10.100.0.4 via xe-0/0/1.0
10.200.0.4/32      *[OSPF/10] 00:15:02, metric 2
                    > to 10.100.0.0 via xe-0/0/0.0
                      to 10.100.0.4 via xe-0/0/1.0
224.0.0.5/32       *[OSPF/10] 00:15:12, metric 1
                      MultiRecv

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
```
LEAF-2:
```
root@LEAF-2> show route protocol ospf   

inet.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.0/31      *[OSPF/10] 00:16:08, metric 2
                    > to 10.100.0.2 via xe-0/0/0.0
10.100.0.4/31      *[OSPF/10] 00:17:26, metric 2
                    > to 10.100.0.6 via xe-0/0/1.0
10.200.0.1/32      *[OSPF/10] 00:16:08, metric 1
                    > to 10.100.0.2 via xe-0/0/0.0
10.200.0.2/32      *[OSPF/10] 00:17:26, metric 1
                    > to 10.100.0.6 via xe-0/0/1.0
10.200.0.3/32      *[OSPF/10] 00:14:58, metric 2
                    > to 10.100.0.2 via xe-0/0/0.0
                      to 10.100.0.6 via xe-0/0/1.0
224.0.0.5/32       *[OSPF/10] 00:18:47, metric 1
                      MultiRecv

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
```

Заглянем в ospf database:
```
root@LEAF-1> show ospf database    

    OSPF database, Area 0.0.0.0
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len 
Router   10.200.0.1       10.200.0.1       0x80000005  1675  0x22 0xbe16  60
Router   10.200.0.2       10.200.0.2       0x80000005  1674  0x22 0x9a27  60
Router  *10.200.0.3       10.200.0.3       0x80000002  1673  0x22 0x6765  60
Router   10.200.0.4       10.200.0.4       0x80000006  1748  0x22 0x8239  60
Network  10.100.0.0       10.200.0.1       0x80000001  1678  0x22 0x54fa  32
Network  10.100.0.3       10.200.0.4       0x80000001  1752  0x22 0x2622  32
Network  10.100.0.4       10.200.0.2       0x80000001  1674  0x22 0x3019  32
Network  10.100.0.7       10.200.0.4       0x80000002   650  0x22 0xa38   32
```
Мы идим LSA type 1 и 2 как и должно быть, * перед LSA означает, что она сгенерирована локально.

Как видно из выводов, маршруты до всех Lo интерфейсах есть в таблицах маршрутизации, можно приступать к настройке overlay.
