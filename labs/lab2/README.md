# Построение Underlay сети на OSPF
---
## Конфигурация
### SPINE-1
```
root@SPINE-1> show configuration protocols ospf 
area 0.0.0.0 {
    interface xe-0/0/0.0;
    interface xe-0/0/1.0;
    interface lo0.0 {
        passive;
    }
}

{master:0}
```
