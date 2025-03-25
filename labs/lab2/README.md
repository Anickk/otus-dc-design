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
---
## Проверка

Проверяем статус соседей:
###LEAF-1
```
root@LEAF-1> show ospf neighbor 
Address          Interface              State     ID               Pri  Dead
10.100.0.0       xe-0/0/0.0             Full      10.200.0.1       128    37
10.100.0.4       xe-0/0/1.0             Full      10.200.0.2       128    35
```
