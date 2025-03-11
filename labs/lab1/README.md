# Распределение адресного пространства
---
## Схема:

Лабораторная работа строится на Juniper QFX, поэтому на схеме присутствуют также PFE.\
Так же на схеме присутствуют PE маршрутизаторы и эмуляции подключения к интернету через ISP, на данном этапе на них можно не обращать внимание.

![img_1.png](scheme.png)
---
В качестве сетей для распределения выбраны сети:\
P2P - 10.100.0.0/24\
Loopback - 10.200.0.0/24

## Таблица для P2P-линков

| Устройство 1 | Интерфейс | Устройство 2 | Интерфейс | IP-адрес устройства 1 | IP-адрес устройства 2 | Подсеть         |
|--------------|-----------|--------------|-----------|-----------------------|-----------------------|-----------------|
| Spine 1      | xe-0/0/0      | Leaf 1       | xe-0/0/0      | 10.100.0.0/31         | 10.100.0.1/31         | 10.100.0.0/31   |
| Spine 1      | xe-0/0/1      | Leaf 2       | xe-0/0/0      | 10.100.0.2/31         | 10.100.0.3/31         | 10.100.0.2/31   |
| Spine 2      | xe-0/0/0      | Leaf 1       | xe-0/0/1      | 10.100.0.4/31         | 10.100.0.5/31         | 10.100.0.4/31   |
| Spine 2      | xe-0/0/1      | Leaf 2       | xe-0/0/1      | 10.100.0.6/31         | 10.100.0.7/31         | 10.100.0.6/31   |



## Таблица для Loopback-интерфейсов

| Устройство   | Интерфейс | IP-адрес       | Подсеть         |
|--------------|-----------|----------------|-----------------|
| Spine 1      | Loopback0 | 10.200.0.1/32  | 10.200.0.0/24   |
| Spine 2      | Loopback0 | 10.200.0.2/32  | 10.200.0.0/24   |
| Leaf 1       | Loopback0 | 10.200.0.3/32  | 10.200.0.0/24   |
| Leaf 2       | Loopback0 | 10.200.0.4/32  | 10.200.0.0/24   |

## Конфигурация
### SPINE-1
```
root@SPINE-1# show interfaces 
xe-0/0/0 {
    description LEAF-1;
    unit 0 {
        family inet {
            address 10.100.0.0/31;
        }
    }
}
xe-0/0/1 {
    description LEAF-2;
    unit 0 {
        family inet {
            address 10.100.0.2/31;
        }
    }
}
lo0 {                                   
    unit 0 {
        family inet {
            address 10.200.0.1/32;
        }
    }
}

{master:0}[edit]
```
### SPINE-2
```
root@SPINE-2# show interfaces 
xe-0/0/0 {
    description LEAF-1;
    unit 0 {
        family inet {
            address 10.100.0.4/31;
        }
    }
}
xe-0/0/1 {
    description LEAF-2;
    unit 0 {
        family inet {
            address 10.100.0.6/31;
        }
    }
}
lo0 {                                   
    unit 0 {
        family inet {
            address 10.200.0.2/32;
        }
    }
}

{master:0}[edit]
```
### LEAF-1
```
root@LEAF-1# show interfaces 
xe-0/0/0 {
    description SPINE-1;
    unit 0 {
        family inet {
            address 10.100.0.1/31;
        }
    }
}
xe-0/0/1 {
    description SPINE-2;
    unit 0 {
        family inet {
            address 10.100.0.5/31;
        }
    }
}
lo0 {                                   
    unit 0 {
        family inet {
            address 10.200.0.3/32;
        }
    }
}

{master:0}[edit]
```
### LEAF-2
```
root@LEAF-2# show interfaces    
xe-0/0/0 {
    description SPINE-1;
    unit 0 {
        family inet {
            address 10.100.0.3/31;
        }
    }
}
xe-0/0/1 {
    description SPINE-2;
    unit 0 {
        family inet {
            address 10.100.0.7/31;
        }
    }
}
lo0 {                                   
    unit 0 {
        family inet {
            address 10.200.0.4/32;
        }
    }
}

{master:0}[edit]
```
