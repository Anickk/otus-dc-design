# VxLAN. EVPN L2
---
## Конфигурация Overlay

В качестве протокола underlay был выбран IS-IS и настроен ранее.\
Пример конфигурации приводится для LEAF-1 и SPINE-1, на парных железках конфигурация схожа.
---
### Leaf-1:

Для начала настроим параметры для балансировки и построения bgp соседства, включим ECMP в VXLAN и Ingress replication для EVPN:
```
policy-options {
    policy-statement load-balancing-policy {
        then {
            load-balance per-packet;
        }
    }
}
routing-options {
    router-id 10.200.0.3;
    autonomous-system 65000;
    forwarding-table {
        export load-balancing-policy;
        dynamic-list-next-hop;
        chained-composite-next-hop {
            ingress {
                evpn;
            }
        }                               
```
Настроим RD и укажем VTEP интерфейс:
```
switch-options {
    vtep-source-interface lo0.0;
    route-distinguisher 10.200.0.3:1;
    vrf-target {
        target:1:1;
        auto;
    }                                   
}
```
Тут стоит отметить, что target:1:1 не нужно задавать при исполтьзовании ibgp в overlay, но для ebgp оно понадобится, т.к. AS будут разные а таргет формируется так - target:<local-AS>:<VNI>. Я буду использовать ibgp, но для наглядности оставлю команду.

Настройка evpn, укажем инкапсуляцию, режим работы, и разрешим инкапсуляцию для всех vni, чтобы не задавать их в ручную:
```
protocols {
    evpn {                              
            encapsulation vxlan;
            multicast-mode ingress-replication;
            extended-vni-list all;
        }
}
```

Настройка bgp overlay:
```
protocols {
    bgp {
        group OVERLAY {
            type internal;
            local-address 10.200.0.3;
            family evpn {
                signaling;
            }
            neighbor 10.200.0.1 {
                description SPINE-1;
            }
            neighbor 10.200.0.2 {
                description SPINE-2;
            }
        }
    }
```
---
### Spine-1:
Тут мы настраиваем только параметры для bgp, балансировку и сам overlay:
```
policy-options {
    policy-statement load-balancing-policy {
        then {
            load-balance per-packet;
        }
    }
}
routing-options {
    router-id 10.200.0.1;
    autonomous-system 65000;
    forwarding-table {
        export load-balancing-policy;   
    }
}
protocols {
    bgp {
        group OVERLAY {
            type internal;
            local-address 10.200.0.1;
            family evpn {
                signaling;
            }
            cluster 10.200.0.1;
            neighbor 10.200.0.3 {
                description LEAF-1;
            }
            neighbor 10.200.0.4 {
                description LEAF-2;
            }
        }
    }
```
cluster 10.200.0.1 - включает RR. на другом spine другой кластер id.
