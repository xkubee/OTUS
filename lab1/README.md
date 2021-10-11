# ЗАДАНИЕ
Проектирование адресного пространства.
# РЕШЕНИЕ
#### Spine - CISCO NX 9k
####  Leaf - CISCO NX 9k
#### Clients - CISCO IOL
#### Switch - CISCO IOL

|    UNIT       |   INTERFACE     | ADDRESS |
| :------------ |:---------------:| -------:|
|    LEAF1      |      lo         | 1.1.1.1 |
|    LEAF2      |      lo         | 2.2.2.2 |
|    LEAF3      |      lo         | 3.3.3.3 |
|    LEAF4      |      lo         | 4.4.4.4 |

|    UNIT        |   INTERFACE     | ADDRESS     |
| :------------  |:---------------:|    --------:|
|    SPINE1      |      lo         | 10.10.10.10 |
|    SPINE2      |      lo         | 20.20.20.20 |
|    SPINE3      |      lo         | 30.30.30.30 |
