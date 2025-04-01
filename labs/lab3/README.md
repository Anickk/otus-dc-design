# Построение Underlay сети на IS-IS
---
## Конфигурация

Конфигурация протокола на всех коммутаторах идентичная, включаем аторизацию с md5 и bfd:
```
protocols {
    isis {
        level 1 disable;
        level 2 {
            authentication-key "$9$RWlhlKWLxdwY8LjH"; ## SECRET-DATA
            authentication-type md5;
            wide-metrics-only;
        }
        interface xe-0/0/0.0 {
            point-to-point;
            bfd-liveness-detection {
                minimum-interval 300;
                multiplier 3;
            }
        }
        interface xe-0/0/1.0 {
            point-to-point;
            bfd-liveness-detection {
                minimum-interval 300;
                multiplier 3;
            }
        }
        interface lo0.0 {
            passive;
        }
    }                                   
}
```
Используем Single-Area L2 для возможности использования расшириных меток которые могут нам потребоваться для управления трафиком.

Конфигурация интерфейсов на примере SPINE-1, включаем на всех интерфейсах family iso:
```
interfaces {                            
    xe-0/0/0 {
        unit 0 {
            family iso;
        }
    }
    xe-0/0/1 {
        unit 0 {
            family iso;
        }
    }
    lo0 {
        unit 0 {
            family iso {
                address 49.0001.0000.0000.0001.00;
            }
        }
    }
}
```
Адреса NET уникальны для каждого коммутатора, зона общая, распределение представлено в следующей таблице:
| Устройство   | NET |
|--------------|-----------|
| Spine 1      | 49.0001.0000.0000.0001.00 |
| Spine 2      | 49.0001.0000.0000.0002.00 |
| Leaf 1       | 49.0001.0000.0000.0003.00 |
| Leaf 2       | 49.0001.0000.0000.0004.00 |


## Проверка

Все проверки будут выполнены на примере SPINE-1, на остальных коммутаторах считаем выводы аналогичными.

Проверяем сходимость is-is:
```
root@SPINE-1> show isis adjacency 
Interface             System         L State        Hold (secs) SNPA
xe-0/0/0.0            LEAF-1         2  Up                   20
xe-0/0/1.0            LEAF-2         2  Up                   23
```
Проверяем статус bfd:
```
root@SPINE-1> show bfd session 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.100.0.1               Up        xe-0/0/0.0     0.900     0.300        3   
10.100.0.3               Up        xe-0/0/1.0     0.900     0.300        3

2 sessions, 2 clients
Cumulative transmit rate 6.7 pps, cumulative receive rate 6.7 pps
```

Проверяем маршруты is-is:
```
root@SPINE-1> show route protocol isis 

inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.4/31      *[IS-IS/18] 00:14:58, metric 20
                    > to 10.100.0.1 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 00:14:54, metric 20
                    > to 10.100.0.3 via xe-0/0/1.0
10.200.0.2/32      *[IS-IS/18] 00:14:54, metric 20
                    > to 10.100.0.1 via xe-0/0/0.0
                      to 10.100.0.3 via xe-0/0/1.0
10.200.0.4/32      *[IS-IS/18] 00:14:54, metric 10
                    > to 10.100.0.3 via xe-0/0/1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
```
