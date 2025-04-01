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

Настройки будут показаны на примере SPINE-1, конфигурация везде иденично за исключением адресов пиров, AS и router-id./
Настроим политики анонсов и балансировки:
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

Настроим router-id, local AS и применим политику балансировки:
```
routing-options {
    router-id 10.200.0.1;
    autonomous-system 65001;
    forwarding-table {
        export load-balancing-policy;
    }
}
```
