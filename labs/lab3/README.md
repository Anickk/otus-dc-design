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

inet.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.100.0.4/31      *[IS-IS/18] 00:27:27, metric 20
                    > to 10.100.0.1 via xe-0/0/0.0
10.100.0.6/31      *[IS-IS/18] 00:27:23, metric 20
                    > to 10.100.0.3 via xe-0/0/1.0
10.200.0.2/32      *[IS-IS/18] 00:27:23, metric 20
                    > to 10.100.0.1 via xe-0/0/0.0
                      to 10.100.0.3 via xe-0/0/1.0
10.200.0.3/32      *[IS-IS/18] 00:00:19, metric 10
                    > to 10.100.0.1 via xe-0/0/0.0
10.200.0.4/32      *[IS-IS/18] 00:27:23, metric 10
                    > to 10.100.0.3 via xe-0/0/1.0

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
```

Посмотрим в lsdb и проверим NET всех узлов:
```
root@SPINE-1> show isis database detail                                     
IS-IS level 1 link-state database:

IS-IS level 2 link-state database:

SPINE-1.00-00 Sequence: 0x13, Checksum: 0xd1c1, Lifetime: 736 secs
   IS neighbor: LEAF-1.00                     Metric:       10
   IS neighbor: LEAF-2.00                     Metric:       10
   IP prefix: 10.100.0.0/31                   Metric:       10 Internal Up
   IP prefix: 10.100.0.2/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.1/32                   Metric:        0 Internal Up

SPINE-2.00-00 Sequence: 0x12, Checksum: 0x5f47, Lifetime: 716 secs
   IS neighbor: LEAF-1.00                     Metric:       10
   IS neighbor: LEAF-2.00                     Metric:       10
   IP prefix: 10.100.0.4/31                   Metric:       10 Internal Up
   IP prefix: 10.100.0.6/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.2/32                   Metric:        0 Internal Up

LEAF-1.00-00 Sequence: 0x10, Checksum: 0xe7a7, Lifetime: 718 secs
   IS neighbor: SPINE-1.00                    Metric:       10
   IS neighbor: SPINE-2.00                    Metric:       10
   IP prefix: 10.100.0.0/31                   Metric:       10 Internal Up
   IP prefix: 10.100.0.4/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.3/32                   Metric:        0 Internal Up

LEAF-2.00-00 Sequence: 0x13, Checksum: 0x653b, Lifetime: 734 secs
   IS neighbor: SPINE-1.00                    Metric:       10
   IS neighbor: SPINE-2.00                    Metric:       10
   IP prefix: 10.100.0.2/31                   Metric:       10 Internal Up
   IP prefix: 10.100.0.6/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.4/32                   Metric:        0 Internal Up

{master:0}
root@SPINE-1> show isis hostname           
IS-IS hostname database:
System ID      Hostname                                         Type
0000.0000.0001 SPINE-1                                          Static  
0000.0000.0002 SPINE-2                                          Dynamic 
0000.0000.0003 LEAF-1                                           Dynamic 
0000.0000.0004 LEAF-2                                           Dynamic 

{master:0}
```

Проверяем доступность узлов:
```
root@SPINE-1> ping 10.200.0.2 
PING 10.200.0.2 (10.200.0.2): 56 data bytes
64 bytes from 10.200.0.2: icmp_seq=1 ttl=63 time=227.520 ms
64 bytes from 10.200.0.2: icmp_seq=2 ttl=63 time=340.996 ms
64 bytes from 10.200.0.2: icmp_seq=3 ttl=63 time=358.726 ms
^C
--- 10.200.0.2 ping statistics ---
4 packets transmitted, 3 packets received, 25% packet loss
round-trip min/avg/max/stddev = 227.520/309.081/358.726/58.125 ms

{master:0}
root@SPINE-1> ping 10.200.0.3    
PING 10.200.0.3 (10.200.0.3): 56 data bytes
64 bytes from 10.200.0.3: icmp_seq=0 ttl=64 time=211.128 ms
64 bytes from 10.200.0.3: icmp_seq=1 ttl=64 time=206.241 ms
64 bytes from 10.200.0.3: icmp_seq=2 ttl=64 time=114.520 ms
^C
--- 10.200.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 114.520/177.296/211.128/44.434 ms

{master:0}
root@SPINE-1> ping 10.200.0.4    
PING 10.200.0.4 (10.200.0.4): 56 data bytes
64 bytes from 10.200.0.4: icmp_seq=0 ttl=64 time=219.067 ms
64 bytes from 10.200.0.4: icmp_seq=1 ttl=64 time=232.134 ms
64 bytes from 10.200.0.4: icmp_seq=2 ttl=64 time=210.195 ms
^C
--- 10.200.0.4 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 210.195/220.465/232.134/9.011 ms

{master:0}
```

Все адреса лупбеков доступны, можно приступать к конфигураии overlay
