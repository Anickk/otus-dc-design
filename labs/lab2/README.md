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
