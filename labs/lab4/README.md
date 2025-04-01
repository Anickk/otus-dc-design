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

Настройка BGP, включим сразу bfd:
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
