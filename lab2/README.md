# ЗАДАНИЕ
Настройка протокола OSPF для underlay-маршрутизации.
# РЕШЕНИЕ

## Схема сети

![дз2](https://user-images.githubusercontent.com/55625869/138177093-5081662d-8a42-45e2-8928-ff3f3e1b8148.PNG)


## Настройка сетевых интерфейсов

Цветной прямоугольник на интерфейсе указывает на принадлежность к area.

Spine-устройства на интерфейсах, входящих в коммутатор, настроены на принадлежность к area 0.

Остальные интерфейсы spine-устройств настроены согласно схеме сети.
Так как best practices является принадлежность leaf к одной area, то каждые три интерфейса на каждом leaf-устройстве принадлежат одной area.

у меня с cisco ios в роли оконечных устройств eve-ng начинала зависать (скорее всего, из-за количества подключённых nxos), поэтому было создано 4 хоста.

Кратко об адресных пространствах

#### Clients

|    UNIT        |   INTERFACE       |     ADDRESS     |
| :------------- |:-----------------:| ---------------:|
|    Client1     |      e0/0         | 192.168.1.1/24  |
|    Client2     |      e0/0         | 192.168.2.1/24  |
|    Client3     |      e0/0         | 192.168.3.1/24  |
|    Client4     |      e0/0         | 192.168.4.1/24  |


|    UNIT        |   INTERFACE     |       ADDRESS      |
| :------------  |:---------------:| ------------------:|
|    SPINE1      |      lo         |    10.10.10.10/32  |
|    SPINE2      |      lo         |    20.20.20.20/32  |
|    SPINE3      |      lo         |    30.30.30.30/32  |
|    switch      |      lo         | 100.100.100.100/32 |

Кратко о маршрутах:

#### Spine1

            spine1# show ip ospf neighbors
             OSPF Process ID 1 VRF default
             Total number of neighbors: 4
             Neighbor ID     Pri State            Up Time  Address         Interface
             1.1.1.1           1 FULL/ -          00:00:51 10.0.0.2        Eth1/1
             2.2.2.2           1 FULL/ -          00:00:52 10.0.0.6        Eth1/2
             3.3.3.3           1 FULL/ -          00:00:51 10.0.0.10       Eth1/3
             100.100.100.100   1 FULL/ -          00:00:52 10.0.0.14       Eth1/4

            spine1# show ip ospf route
             OSPF Process ID 1 VRF default, Routing Table
              (D) denotes route is directly attached      (R) denotes route is in RIB
              (L) denotes route label is in ULIB          (NHR) denotes next-hop is in RIB
            10.0.0.0/30 (intra)(D) area 0.0.0.1
                 via 10.0.0.0/Eth1/1*  , cost 40 distance 110 (NHR)
            10.0.0.4/30 (intra)(D) area 0.0.0.2
                 via 10.0.0.4/Eth1/2*  , cost 40 distance 110 (NHR)
            10.0.0.8/30 (intra)(D) area 0.0.0.3
                 via 10.0.0.8/Eth1/3*  , cost 40 distance 110 (NHR)
            10.0.0.12/30 (intra)(D) area 0.0.0.0
                 via 10.0.0.12/Eth1/4*  , cost 40 distance 110 (NHR)
            10.0.0.16/30 (intra)(R) area 0.0.0.1
                 via 10.0.0.2/Eth1/1  , cost 80 distance 110 (NHR)
            10.0.0.20/30 (intra)(R) area 0.0.0.2
                 via 10.0.0.6/Eth1/2  , cost 80 distance 110 (NHR)
            10.0.0.24/30 (intra)(R) area 0.0.0.3
                 via 10.0.0.10/Eth1/3  , cost 80 distance 110 (NHR)
            10.0.0.28/30 (intra)(R) area 0.0.0.0
                 via 10.0.0.14/Eth1/4  , cost 80 distance 110 (NHR)
            10.0.0.36/30 (intra)(R) area 0.0.0.0
                 via 10.0.0.14/Eth1/4  , cost 80 distance 110 (NHR)
            10.10.10.10/32 (intra)(D) area 0.0.0.0
                 via 10.10.10.10/Lo0*  , cost 1 distance 110 (NHR)
            30.30.30.30/32 (intra)(R) area 0.0.0.0
                 via 10.0.0.14/Eth1/4  , cost 81 distance 110 (NHR)
            192.168.1.0/24 (intra)(R) area 0.0.0.1
                 via 10.0.0.2/Eth1/1  , cost 80 distance 110 (NHR)
            192.168.2.0/24 (intra)(R) area 0.0.0.2
                 via 10.0.0.6/Eth1/2  , cost 80 distance 110 (NHR)
            192.168.3.0/24 (intra)(R) area 0.0.0.3
                 via 10.0.0.10/Eth1/3  , cost 80 distance 110 (NHR)

#### Spine2

            spine2# show ip ospf neighbors
             OSPF Process ID 1 VRF default
             Total number of neighbors: 4
             Neighbor ID     Pri State            Up Time  Address         Interface
             1.1.1.1           1 FULL/ -          00:38:18 10.0.0.18       Eth1/1
             2.2.2.2           1 FULL/ -          00:58:04 10.0.0.22       Eth1/2
             3.3.3.3           1 FULL/ -          00:58:04 10.0.0.26       Eth1/3
             100.100.100.100   1 FULL/ -          00:49:29 10.0.0.30       Eth1/4

            spine2# show ip ospf route
             OSPF Process ID 1 VRF default, Routing Table
              (D) denotes route is directly attached      (R) denotes route is in RIB
              (L) denotes route label is in ULIB          (NHR) denotes next-hop is in RIB
            10.0.0.0/30 (intra)(R) area 0.0.0.1
                 via 10.0.0.18/Eth1/1  , cost 80 distance 110 (NHR)
            10.0.0.4/30 (intra)(R) area 0.0.0.2
                 via 10.0.0.22/Eth1/2  , cost 80 distance 110 (NHR)
            10.0.0.8/30 (intra)(R) area 0.0.0.3
                 via 10.0.0.26/Eth1/3  , cost 80 distance 110 (NHR)
            10.0.0.12/30 (intra)(R) area 0.0.0.0
                 via 10.0.0.30/Eth1/4  , cost 80 distance 110 (NHR)
            10.0.0.16/30 (intra)(D) area 0.0.0.1
                 via 10.0.0.16/Eth1/1*  , cost 40 distance 110 (NHR)
            10.0.0.20/30 (intra)(D) area 0.0.0.2
                 via 10.0.0.20/Eth1/2*  , cost 40 distance 110 (NHR)
            10.0.0.24/30 (intra)(D) area 0.0.0.3
                 via 10.0.0.24/Eth1/3*  , cost 40 distance 110 (NHR)
            10.0.0.28/30 (intra)(D) area 0.0.0.0
                 via 10.0.0.28/Eth1/4*  , cost 40 distance 110 (NHR)
            10.0.0.36/30 (intra)(R) area 0.0.0.0
                 via 10.0.0.30/Eth1/4  , cost 80 distance 110 (NHR)
            10.10.10.10/32 (intra)(R) area 0.0.0.0
                 via 10.0.0.30/Eth1/4  , cost 81 distance 110 (NHR)
            30.30.30.30/32 (intra)(D) area 0.0.0.0
                 via 30.30.30.30/Lo0*  , cost 1 distance 110 (NHR)
            192.168.1.0/24 (intra)(R) area 0.0.0.1
                 via 10.0.0.18/Eth1/1  , cost 80 distance 110 (NHR)
            192.168.2.0/24 (intra)(R) area 0.0.0.2
                 via 10.0.0.22/Eth1/2  , cost 80 distance 110 (NHR)
            192.168.3.0/24 (intra)(R) area 0.0.0.3
                 via 10.0.0.26/Eth1/3  , cost 80 distance 110 (NHR)

#### Leaf1

            leaf1# show ip ospf neighbors
             OSPF Process ID 1 VRF default
             Total number of neighbors: 2
             Neighbor ID     Pri State            Up Time  Address         Interface
             10.10.10.10       1 FULL/ -          00:04:22 10.0.0.1        Eth1/1
             20.20.20.20       1 FULL/ -          00:40:49 10.0.0.17       Eth1/2

            leaf1# show ip ospf route
             OSPF Process ID 1 VRF default, Routing Table
              (D) denotes route is directly attached      (R) denotes route is in RIB
              (L) denotes route label is in ULIB          (NHR) denotes next-hop is in RIB
            10.0.0.0/30 (intra)(D) area 0.0.0.1
                 via 10.0.0.0/Eth1/1*  , cost 40 distance 110 (NHR)
            10.0.0.4/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 80 distance 110 (NHR)
            10.0.0.8/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 80 distance 110 (NHR)
            10.0.0.12/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 80 distance 110 (NHR)
            10.0.0.16/30 (intra)(D) area 0.0.0.1
                 via 10.0.0.16/Eth1/2*  , cost 40 distance 110 (NHR)
            10.0.0.20/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.17/Eth1/2  , cost 80 distance 110 (NHR)
            10.0.0.24/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.17/Eth1/2  , cost 80 distance 110 (NHR)
            10.0.0.28/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.17/Eth1/2  , cost 80 distance 110 (NHR)
            10.0.0.36/30 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 120 distance 110 (NHR)
                 via 10.0.0.17/Eth1/2  , cost 120 distance 110 (NHR)
            10.10.10.10/32 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 41 distance 110 (NHR)
            30.30.30.30/32 (inter)(R) area 0.0.0.1
                 via 10.0.0.17/Eth1/2  , cost 41 distance 110 (NHR)
            192.168.1.0/24 (intra)(D) area 0.0.0.1
                 via 192.168.1.0/Eth1/5*  , cost 40 distance 110 (NHR)
            192.168.2.0/24 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 120 distance 110 (NHR)
                 via 10.0.0.17/Eth1/2  , cost 120 distance 110 (NHR)
            192.168.3.0/24 (inter)(R) area 0.0.0.1
                 via 10.0.0.1/Eth1/1  , cost 120 distance 110 (NHR)
                 via 10.0.0.17/Eth1/2  , cost 120 distance 110 (NHR)

#### Сетевая связность

            c1> ping 192.168.3.1

            84 bytes from 192.168.3.1 icmp_seq=1 ttl=61 time=60.835 ms
            84 bytes from 192.168.3.1 icmp_seq=2 ttl=61 time=39.891 ms
            84 bytes from 192.168.3.1 icmp_seq=3 ttl=61 time=26.413 ms
            ^C
            c1> ping 192.168.2.1

            192.168.2.1 icmp_seq=1 timeout
            84 bytes from 192.168.2.1 icmp_seq=2 ttl=61 time=30.148 ms
            84 bytes from 192.168.2.1 icmp_seq=3 ttl=61 time=24.415 ms
            ^C
            c1> ping 192.168.4.1

            84 bytes from 192.168.4.1 icmp_seq=1 ttl=59 time=34.574 ms
            84 bytes from 192.168.4.1 icmp_seq=2 ttl=59 time=45.661 ms
            84 bytes from 192.168.4.1 icmp_seq=3 ttl=59 time=20.289 ms
            84 bytes from 192.168.4.1 icmp_seq=4 ttl=59 time=33.119 ms
            ^C

