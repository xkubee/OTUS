# ЗАДАНИЕ
Настройка протокола BGP для underlay-маршрутизации.
# РЕШЕНИЕ

## Схема сети

![bgp_рис](https://user-images.githubusercontent.com/55625869/139600484-55838415-dee1-4987-830a-ccc8fc97e504.PNG)

## Были учтены следующие рекомендации:

#### as-path multipath-relax   		

Главная "фича" BGP - балансировка трафика по всем линкам.

#### keepalive (3), hold timer (9), advertisement interval (0)

Для ЦОД данные таймеры критичны, поэтому такие "частые" значения.

#### Использование

BFD (На NX OS реализован на уровне железа) и Templates (удобство).


#### Модель ASN 

2-ух уровневаня архитектура: все Spine - в одну ASN, каждый LEAF в свою ASN

3-ух уровневаня архитектура: Core - в одну ASN, каждый Spine - в свою ASN, каждый LEAF - в свою ASN.

## Корректировка адресного пространства

#### Switching

|    UNIT        |              INTERFACES          |
| :------------- |:--------------------------------:|
|    Switch      |             switchports          |

#### Distribution layer

|    UNIT        |   INTERFACE       |    NETWORK   |    ADDRESS    |
| :------------- |:-----------------:|:------------:| -------------:|
|    SPINE1      |      e1/1         | 10.0.0.0/30  | 10.0.0.1/30   |
|                |      e1/2         | 10.0.0.4/30  | 10.0.0.5/30   |
|                |      e1/3         | 10.0.0.8/30  | 10.0.0.9/30   |
|                |      e1/4         | 172.16.1.0/29| 172.16.1.1/29 |
|                |       lo          |      /32     | 10.10.10.10/32|
|    SPINE2      |      e1/1         | 10.0.0.16/30 | 10.0.0.17/30  |
|                |      e1/2         | 10.0.0.20/30 | 10.0.0.21/30  |
|                |      e1/3         | 10.0.0.24/30 | 10.0.0.25/30  |
|                |      e1/4         | 172.16.1.0/29| 172.16.1.2/29 |
|                |       lo          |      /32     | 20.20.20.20/32|
|    SPINE3      |      e1/1         | 10.0.0.32/30 | 10.0.0.33/30  |
|                |      e1/2         | 172.16.1.0/29| 172.16.1.3/29 |
|                |       lo          |      /32     | 30.30.30.30/32|

#### Access layer

|    UNIT       |   INTERFACE       |    NETWORK   |    ADDRESS   |
| :------------ |:-----------------:|:------------:| ------------:|
|    LEAF1      |      e1/1         | 10.0.0.0/30  | 10.0.0.2/30  |
|               |      e1/2         | 10.0.0.16/30 | 10.0.0.18/30 |
|               |       lo          |      /32     |  1.1.1.1/32  |
|    LEAF2      |      e1/1         | 10.0.0.4/30  | 10.0.0.6/30  |
|               |      e1/2         | 10.0.0.20/30 | 10.0.0.22/30 |
|               |       lo          |      /32     |  2.2.2.2/32  |
|    LEAF3      |      e1/1         | 10.0.0.8/30  | 10.0.0.10/30 |
|               |      e1/2         | 10.0.0.24/30 | 10.0.0.26/30 |
|               |       lo          |      /32     |  3.3.3.3/32  |
|    LEAF4      |      e1/1         | 10.0.0.32/30 | 10.0.0.34/30 |
|               |       lo          |      /32     |  4.4.4.4/32  |

#### Clients

|    UNIT        |   INTERFACE       |     ADDRESS     |
| :------------- |:-----------------:| ---------------:|
|    Client1     |      e0/0         | 192.168.1.1/24  |
|    Client2     |      e0/0         | 192.168.2.1/24  |
|    Client3     |      e0/0         | 192.168.3.1/24  |
|    Client4     |      e0/0         | 192.168.4.1/24  |

#### Прямоугольниками разных цветов отмечены сети, которые будут транслироваться устройствами своим "пирам".

Так, например, в группе address-family ipv4 unicast устройством Leaf1 будет транслироваться сеть, находящаяся за интерфейсом e1/5.
Spine1 будет транслировать сети, которые находятся за интерфейсами e1/1, e1/2, e1/3 в сеть за интерфейсом e1/4.

## Важные моменты в конфигурировании:

## spine1

#### network settings 

	interface Ethernet1/1        <--- network leaf1
	  no switchport
	  ip address 10.0.0.1/30
	  no shutdown

	interface Ethernet1/2        <--- network leaf2
	  no switchport
	  ip address 10.0.0.5/30
	  no shutdown

	interface Ethernet1/3        <--- network leaf3
	  no switchport
	  ip address 10.0.0.9/30
	  no shutdown

	interface Ethernet1/4        <--- network SPINES
	  no switchport
	  ip address 172.16.1.1/29
	  no shutdown

#### bgp settings

	router bgp 65000
	  router-id 10.10.10.10
	  bestpath as-path multipath-relax  <--- балансировка
	  address-family ipv4 unicast       <--- адреса сетей, которые необходимо транслировать в "wan" (или же, остальным spine)  
		network 10.0.0.0/30
		network 10.0.0.4/30
		network 10.0.0.8/30
	  template peer LEAFS               <--- создание шаблона для удобства настройки leaf-устройств
		timers 3 9                  <--- таймеры
		address-family ipv4 unicast
	  neighbor 10.0.0.2                 <--- адреса соседей, с которыми должен быть осуществлён "пиринг"
		inherit peer LEAFS      
		remote-as 65001             <--- номер автономной системы соседа
		description LEAF1
		update-source Ethernet1/1   <--- через данный интерфейс с соседом 10.0.0.2 будет осуществляться пиринг с последующей передачей всей доступной информации 
	  neighbor 10.0.0.6
		inherit peer LEAFS
		remote-as 65002
		description LEAF2
		update-source Ethernet1/2
	  neighbor 10.0.0.10
		inherit peer LEAFS
		remote-as 65003
		description LEAF3
		update-source Ethernet1/3
	  neighbor 172.16.1.2
		inherit peer LEAFS
		remote-as 65000
		description SPINE2
		update-source Ethernet1/4
	  neighbor 172.16.1.3
		inherit peer LEAFS
		remote-as 65000
		description SPINE3
		update-source Ethernet1/4

## leaf1

#### network settings

	interface Ethernet1/1       <--- network spine1   
	  no switchport
	  ip address 10.0.0.2/30
	  no shutdown

	interface Ethernet1/2       <--- network spine2 
	  no switchport
	  ip address 10.0.0.18/30
	  no shutdown

	interface Ethernet1/5       <--- network client1  
	  no switchport
	  ip address 192.168.1.10/24
	  no shutdown

#### bgp settings 

	router bgp 65001
	  router-id 1.1.1.1
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 192.168.1.0/24
	  template peer SPINES
		remote-as 65000
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.1
		inherit peer SPINES
		remote-as 65000
		description SPINE1
		update-source Ethernet1/1
	  neighbor 10.0.0.17
		inherit peer SPINES
		remote-as 65000
		description SPINE2
		update-source Ethernet1/2

### Настройки остальных устройств аналогичны и приведены в файлах settings_*.conf

## Маршрутная информация:

## spine1

	spine1# show ip route
	IP Route Table for VRF "default"
	'*' denotes best ucast next-hop
	'**' denotes best mcast next-hop
	'[x/y]' denotes [preference/metric]
	'%<string>' in via output denotes VRF <string>

	10.0.0.0/30, ubest/mbest: 1/0, attached
	    *via 10.0.0.1, Eth1/1, [0/0], 1d11h, direct
	10.0.0.1/32, ubest/mbest: 1/0, attached
	    *via 10.0.0.1, Eth1/1, [0/0], 1d11h, local
	10.0.0.4/30, ubest/mbest: 1/0, attached
	    *via 10.0.0.5, Eth1/2, [0/0], 1d11h, direct
	10.0.0.5/32, ubest/mbest: 1/0, attached
	    *via 10.0.0.5, Eth1/2, [0/0], 1d11h, local
	10.0.0.8/30, ubest/mbest: 1/0, attached
	    *via 10.0.0.9, Eth1/3, [0/0], 1d11h, direct
	10.0.0.9/32, ubest/mbest: 1/0, attached
	    *via 10.0.0.9, Eth1/3, [0/0], 1d11h, local
	10.0.0.16/30, ubest/mbest: 1/0
	    *via 172.16.1.2, [200/0], 00:59:33, bgp-65000, internal, tag 65000
	10.0.0.20/30, ubest/mbest: 1/0
	    *via 172.16.1.2, [200/0], 00:59:25, bgp-65000, internal, tag 65000
	10.0.0.24/30, ubest/mbest: 1/0
	    *via 172.16.1.2, [200/0], 00:59:14, bgp-65000, internal, tag 65000
	10.0.0.32/30, ubest/mbest: 1/0
	    *via 172.16.1.3, [200/0], 01:09:04, bgp-65000, internal, tag 65000
	10.10.10.10/32, ubest/mbest: 2/0, attached
	    *via 10.10.10.10, Lo0, [0/0], 1d11h, local
	    *via 10.10.10.10, Lo0, [0/0], 1d11h, direct
	172.16.1.0/29, ubest/mbest: 1/0, attached
	    *via 172.16.1.1, Eth1/4, [0/0], 01:40:43, direct
	172.16.1.1/32, ubest/mbest: 1/0, attached
	    *via 172.16.1.1, Eth1/4, [0/0], 01:40:43, local
	192.168.1.0/24, ubest/mbest: 1/0
	    *via 10.0.0.2, [20/0], 00:59:33, bgp-65000, external, tag 65001
	192.168.2.0/24, ubest/mbest: 1/0
	    *via 10.0.0.6, [20/0], 00:59:24, bgp-65000, external, tag 65002
	192.168.3.0/24, ubest/mbest: 1/0
	    *via 10.0.0.10, [20/0], 00:59:12, bgp-65000, external, tag 65003
	192.168.4.0/24, ubest/mbest: 1/0
	    *via 10.0.0.34, [200/0], 01:08:58, bgp-65000, internal, tag 65004

Знает обо всех нужных сетях.

## leaf4

	leaf4# show ip route
	IP Route Table for VRF "default"
	'*' denotes best ucast next-hop
	'**' denotes best mcast next-hop
	'[x/y]' denotes [preference/metric]
	'%<string>' in via output denotes VRF <string>

	4.4.4.4/32, ubest/mbest: 2/0, attached
	    *via 4.4.4.4, Lo0, [0/0], 1d10h, local
	    *via 4.4.4.4, Lo0, [0/0], 1d10h, direct
	10.0.0.0/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:33, bgp-65004, external, tag 65000
	10.0.0.4/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:24, bgp-65004, external, tag 65000
	10.0.0.8/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:15, bgp-65004, external, tag 65000
	10.0.0.16/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:01:03, bgp-65004, external, tag 65000
	10.0.0.20/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:00:54, bgp-65004, external, tag 65000
	10.0.0.24/30, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:00:44, bgp-65004, external, tag 65000
	10.0.0.32/30, ubest/mbest: 1/0, attached
	    *via 10.0.0.34, Eth1/1, [0/0], 1d10h, direct
	10.0.0.34/32, ubest/mbest: 1/0, attached
	    *via 10.0.0.34, Eth1/1, [0/0], 1d10h, local
	192.168.1.0/24, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:32, bgp-65004, external, tag 65000
	192.168.2.0/24, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:23, bgp-65004, external, tag 65000
	192.168.3.0/24, ubest/mbest: 1/0
	    *via 10.0.0.33, [20/0], 01:03:14, bgp-65004, external, tag 65000
	192.168.4.0/24, ubest/mbest: 1/0, attached
	    *via 192.168.4.10, Eth1/2, [0/0], 1d10h, direct
	192.168.4.10/32, ubest/mbest: 1/0, attached
	    *via 192.168.4.10, Eth1/2, [0/0], 1d10h, local

Знает обо всех нужных сетях.

## Сетевая связность оконечных устройств:

## client1

	c1> ping 192.168.4.1

	192.168.4.1 icmp_seq=1 timeout
	84 bytes from 192.168.4.1 icmp_seq=2 ttl=60 time=41.412 ms
	84 bytes from 192.168.4.1 icmp_seq=3 ttl=60 time=26.755 ms
	84 bytes from 192.168.4.1 icmp_seq=4 ttl=60 time=30.994 ms
	84 bytes from 192.168.4.1 icmp_seq=5 ttl=60 time=38.981 ms

Клиенты c1 и c2 находятся в сетевой доступности друг для друга.

