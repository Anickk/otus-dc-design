# Построение Underlay сети на eBGP
---
## Конфигурация

Распределение AS:
| Устройство   | AS |
|--------------|-----------|
| Spine 1      | 65001 |
| Spine 2      | 65002 |
| Leaf 1       | 65011 |
| Leaf 2       | 65012 |


Настройки будут показаны на примере SPINE-1, конфигурация везде иденично за исключением адресов пиров, AS и router-id.

Настройка политики анонсов и балансировки:
```
policy-options {
    policy-statement lo0_in {
        term 1 {                        
            from {
                route-filter 10.200.0.0/24 upto /32;
            }
            then accept;
        }
    }
    policy-statement lo0_out {
        term 1 {
            from {
                protocol direct;
                route-filter 10.200.0.1/32 exact;
            }
            then accept;
        }
    }
    policy-statement load-balancing-policy {
        then {
            load-balance per-packet;
        }
```
Принимаем все адреса /32 входящие в сеть 10.200.0.0/24, включаем редистрибьюцию для коннектед сети lo0.

На Juniper нет мнимого DENY правила, поэтому правильнее было бы добавить команду set policy-options policy-statement lo0_in then reject, но мы экспортируем только адреса lo везде, поэтому политику импорта можно не использовать вовсе.


Настройка router-id, local AS и применение политику балансировки:
```
routing-options {
    router-id 10.200.0.1;
    autonomous-system 65001;
    forwarding-table {
        export load-balancing-policy;
    }
}
```

Настройка bgp, включим сразу multipath и bfd:
```
protocols {
    bgp {
        group UNDERLAY {                
            type external;
            import lo0_in;
            export lo0_out;
            multipath {
                multiple-as;
            }
            bfd-liveness-detection {
                minimum-interval 1000;
            }
            neighbor 10.100.0.1 {
                description LEAF-1;
                peer-as 65011;
            }
            neighbor 10.100.0.3 {
                description LEAF-2;
                peer-as 65012;
            }
        }
    }
}
```

## Проверка

Проверка bgp и bfd сессий:
```
root@SPINE-1> show bfd session 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.100.0.1               Up        xe-0/0/0.0     3.000     1.000        3   
10.100.0.3               Up        xe-0/0/1.0     3.000     1.000        3   

2 sessions, 2 clients
Cumulative transmit rate 2.0 pps, cumulative receive rate 2.0 pps

{master:0}
root@SPINE-1> show bgp summary    
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       6          4          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.100.0.1            65011        109        105       0       0       45:14 2/3/3/0              0/0/0/0
10.100.0.3            65012         42         43       0       1       15:42 2/3/3/0              0/0/0/0

{master:0}
```

Проверка отправляемых и получаемых анонсов на примере одного нейбора:
```
root@SPINE-1> show route advertising-protocol bgp 10.100.0.1 

inet.0: 10 destinations, 13 routes (10 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.200.0.1/32           Self                                    I
* 10.200.0.4/32           Self                                    65012 I

{master:0}
root@SPINE-1> show route receive-protocol bgp 10.100.0.1        

inet.0: 10 destinations, 13 routes (10 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.200.0.2/32           10.100.0.1                              65011 65002 I
* 10.200.0.3/32           10.100.0.1                              65011 I
  10.200.0.4/32           10.100.0.1                              65011 65002 65012 I

inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

{master:0}
```

Проверка таблицы маршрутизации bgp и доступности всех узлов:
```
root@SPINE-1> show route protocol bgp 

inet.0: 10 destinations, 13 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.200.0.2/32      *[BGP/170] 00:11:28, localpref 100
                      AS path: 65011 65002 I, validation-state: unverified
                    > to 10.100.0.1 via xe-0/0/0.0
                      to 10.100.0.3 via xe-0/0/1.0
                    [BGP/170] 00:18:40, localpref 100
                      AS path: 65012 65002 I, validation-state: unverified
                    > to 10.100.0.3 via xe-0/0/1.0
10.200.0.3/32      *[BGP/170] 00:44:18, localpref 100
                      AS path: 65011 I, validation-state: unverified
                    > to 10.100.0.1 via xe-0/0/0.0
                    [BGP/170] 00:18:40, localpref 100
                      AS path: 65012 65002 65011 I, validation-state: unverified
                    > to 10.100.0.3 via xe-0/0/1.0
10.200.0.4/32      *[BGP/170] 00:18:40, localpref 100
                      AS path: 65012 I, validation-state: unverified
                    > to 10.100.0.3 via xe-0/0/1.0
                    [BGP/170] 00:19:13, localpref 100
                      AS path: 65011 65002 65012 I, validation-state: unverified
                    > to 10.100.0.1 via xe-0/0/0.0
                                        
inet6.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)

{master:0}
root@SPINE-1> ping 10.200.0.2 
PING 10.200.0.2 (10.200.0.2): 56 data bytes
64 bytes from 10.200.0.2: icmp_seq=0 ttl=63 time=308.717 ms
64 bytes from 10.200.0.2: icmp_seq=1 ttl=63 time=109.659 ms
^C
--- 10.200.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 109.659/209.188/308.717/99.529 ms

{master:0}
root@SPINE-1> ping 10.200.0.3    
PING 10.200.0.3 (10.200.0.3): 56 data bytes
64 bytes from 10.200.0.3: icmp_seq=0 ttl=64 time=156.278 ms
64 bytes from 10.200.0.3: icmp_seq=1 ttl=64 time=270.472 ms
^C
--- 10.200.0.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 156.278/213.375/270.472/57.097 ms

{master:0}
root@SPINE-1> ping 10.200.0.4    
PING 10.200.0.4 (10.200.0.4): 56 data bytes
64 bytes from 10.200.0.4: icmp_seq=0 ttl=64 time=244.059 ms
64 bytes from 10.200.0.4: icmp_seq=1 ttl=64 time=199.181 ms
^C
--- 10.200.0.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 199.181/221.620/244.059/22.439 ms

{master:0}
```

Как видно из выводов, multipath работает, и все узлы доступны, можно приступать к настройке overlay.
