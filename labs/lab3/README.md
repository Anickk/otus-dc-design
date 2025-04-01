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
        description LEAF-1;
        unit 0 {
            family iso;
        }
    }
    xe-0/0/1 {
        description LEAF-2;
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
