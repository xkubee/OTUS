# ЗАДАНИЕ
Настройка протокола ISIS для underlay-маршрутизации.
# РЕШЕНИЕ

## Схема сети

![дз3](https://user-images.githubusercontent.com/55625869/138610232-c1aef733-2cd1-46f7-9f98-a0d0aba2ef7f.PNG)

## Настройка сетевых интерфейсов

Цветной прямоугольник на интерфейсе указывает на принадлежность к area.

    ВАЖНО! При настройке важно было помнить о том, что устройство должно ПОЛНОСТЬЮ входить в area (в area должны входить все сетевые интерфейсы)

Кратко об адресных пространствах

#### ClIENTS

|    UNIT        |   INTERFACE       |     ADDRESS     |
| :------------- |:-----------------:| ---------------:|
|    Client1     |      e0/0         | 192.168.1.1/24  |
|    Client2     |      e0/0         | 192.168.2.1/24  |
|    Client3     |      e0/0         | 192.168.3.1/24  |
|    Client4     |      e0/0         | 192.168.4.1/24  |

#### ISIS net (zone)

|    UNIT        |          INTERFACE        |
| :------------  | -------------------------:|
|    switch      | 49.0111.0111.0111.0111.00 |
|    SPINE1      | 49.0111.0001.0001.0001.00 |
|    SPINE2      | 49.0111.0002.0002.0002.00 |
|    SPINE3      | 49.0111.0003.0003.0003.00 |
|    LEAF1       | 49.0001.0011.0011.0011.00 |
|    LEAF2       | 49.0001.0022.0022.0022.00 |
|    LEAF3       | 49.0001.0033.0033.0033.00 |
|    LEAF4       | 49.0002.0044.0044.0044.00 |

Кратко о маршрутах:

#### Leaf1

    leaf1# show isis route
    IS-IS process: 9001 VRF: default
    IS-IS IPv4 routing table

    10.0.0.0/30, L1, direct
       *via Ethernet1/1, metric 40, L1, direct
      via Ethernet1/1, metric 40, L2, direct
    10.0.0.4/30, L2
       *via 10.0.0.1, Ethernet1/1, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    10.0.0.8/30, L2
       *via 10.0.0.1, Ethernet1/1, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    10.0.0.12/30, L2
       *via 10.0.0.1, Ethernet1/1, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    10.0.0.16/30, L1, direct
       *via Ethernet1/2, metric 40, L1, direct
      via Ethernet1/2, metric 40, L2, direct
    10.0.0.20/30, L2
       *via 10.0.0.17, Ethernet1/2, metric 80, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
    10.0.0.24/30, L2
       *via 10.0.0.17, Ethernet1/2, metric 80, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
    10.0.0.28/30, L2
       *via 10.0.0.17, Ethernet1/2, metric 80, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
    10.0.0.32/30, L2
       *via 10.0.0.17, Ethernet1/2, metric 160, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    10.0.0.36/30, L2
       *via 10.0.0.17, Ethernet1/2, metric 120, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    192.168.1.0/24, L1, direct
       *via Ethernet1/5, metric 40, L1, direct
      via Ethernet1/5, metric 40, L2, direct
    192.168.2.0/24, L2
       *via 10.0.0.1, Ethernet1/1, metric 120, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.17, Ethernet1/2, metric 120, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    192.168.3.0/24, L2
       *via 10.0.0.1, Ethernet1/1, metric 120, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.17, Ethernet1/2, metric 120, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    192.168.4.0/24, L2
       *via 10.0.0.17, Ethernet1/2, metric 200, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115

#### Spine1

    spine1# show isis route
    IS-IS process: 9000 VRF: default
    IS-IS IPv4 routing table

    10.0.0.0/30, L1, direct
       *via Ethernet1/1, metric 40, L1, direct
      via Ethernet1/1, metric 40, L2, direct
    10.0.0.4/30, L1, direct
       *via Ethernet1/2, metric 40, L1, direct
      via Ethernet1/2, metric 40, L2, direct
    10.0.0.8/30, L1, direct
       *via Ethernet1/3, metric 40, L1, direct
      via Ethernet1/3, metric 40, L2, direct
    10.0.0.12/30, L1, direct
       *via Ethernet1/4, metric 40, L1, direct
      via Ethernet1/4, metric 40, L2, direct
    10.0.0.16/30, L2
       *via 10.0.0.2, Ethernet1/1, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    10.0.0.20/30, L2
       *via 10.0.0.6, Ethernet1/2, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    10.0.0.24/30, L2
       *via 10.0.0.10, Ethernet1/3, metric 80, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
    10.0.0.28/30, L2
       *via 10.0.0.2, Ethernet1/1, metric 120, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.6, Ethernet1/2, metric 120, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.10, Ethernet1/3, metric 120, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    10.0.0.32/30, L2
       *via 10.0.0.2, Ethernet1/1, metric 200, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.6, Ethernet1/2, metric 200, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.10, Ethernet1/3, metric 200, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    10.0.0.36/30, L2
       *via 10.0.0.2, Ethernet1/1, metric 160, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.6, Ethernet1/2, metric 160, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.10, Ethernet1/3, metric 160, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115
    192.168.1.0/24, L2
       *via 10.0.0.2, Ethernet1/1, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    192.168.2.0/24, L2
       *via 10.0.0.6, Ethernet1/2, metric 80, L2 (I,U), table-map-deny no-pol, admin
    -dist 115
    192.168.3.0/24, L2
       *via 10.0.0.10, Ethernet1/3, metric 80, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
    192.168.4.0/24, L2
       *via 10.0.0.2, Ethernet1/1, metric 240, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.6, Ethernet1/2, metric 240, L2 (I,U), table-map-deny no-pol, admi
    n-dist 115
       *via 10.0.0.10, Ethernet1/3, metric 240, L2 (I,U), table-map-deny no-pol, adm
    in-dist 115

#### Сетевая связность

    c1> ping 192.168.2.1

    84 bytes from 192.168.2.1 icmp_seq=1 ttl=61 time=37.283 ms
    84 bytes from 192.168.2.1 icmp_seq=2 ttl=61 time=18.327 ms
    84 bytes from 192.168.2.1 icmp_seq=3 ttl=61 time=12.547 ms
    ^C
    c1> ping 192.168.3.1

    192.168.3.1 icmp_seq=1 timeout
    84 bytes from 192.168.3.1 icmp_seq=2 ttl=61 time=35.999 ms
    84 bytes from 192.168.3.1 icmp_seq=3 ttl=61 time=15.767 ms
    84 bytes from 192.168.3.1 icmp_seq=4 ttl=61 time=17.009 ms
    ^C
    c1> ping 192.168.4.1

    84 bytes from 192.168.4.1 icmp_seq=1 ttl=59 time=32.380 ms
    84 bytes from 192.168.4.1 icmp_seq=2 ttl=59 time=24.445 ms
    84 bytes from 192.168.4.1 icmp_seq=3 ttl=59 time=21.818 ms
    ^C
    c1>
