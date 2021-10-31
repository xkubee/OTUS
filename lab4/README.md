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
|    Client2     |      e0/0         | 192.168.1.2/24  |
|    Client3     |      e0/0         | 192.168.1.3/24  |


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
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 10.0.0.0/30
		network 10.0.0.4/30
		network 10.0.0.8/30
	  template peer LEAFS
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.2
		inherit peer LEAFS
		remote-as 65001
		description LEAF1
		update-source Ethernet1/1
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


## spine2

#### network settings 

	interface Ethernet1/1      <--- network leaf1
	  no switchport
	  ip address 10.0.0.17/30
	  no shutdown

	interface Ethernet1/2      <--- network leaf2
	  no switchport
	  ip address 10.0.0.21/30
	  no shutdown

	interface Ethernet1/3      <--- network leaf3
	  no switchport
	  ip address 10.0.0.25/30
	  no shutdown

	interface Ethernet1/4      <--- network SPINES
	  no switchport
	  ip address 172.16.1.2/29
	  no shutdown

#### bgp settings

	router bgp 65000
	  router-id 20.20.20.20
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 10.0.0.16/30
		network 10.0.0.20/30
		network 10.0.0.24/30
	  template peer LEAFS
		timers 3 9
		address-family ipv4 unicast
	  template peer SPINES
		remote-as 65000
		update-source Ethernet1/4
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.18
		inherit peer LEAFS
		remote-as 65001
		description LEAF1
		update-source Ethernet1/1
	  neighbor 10.0.0.22
		inherit peer LEAFS
		remote-as 65002
		description LEAF2
		update-source Ethernet1/2
	  neighbor 10.0.0.26
		inherit peer LEAFS
		remote-as 65003
		description LEAF3
		update-source Ethernet1/3
	  neighbor 172.16.1.1
		inherit peer SPINES
		description SPINE1
	  neighbor 172.16.1.3
		inherit peer SPINES
		description SPINE3


## spine3

#### network settings

	interface Ethernet1/1        <--- network leaf4
	  no switchport
	  ip address 10.0.0.33/30
	  no shutdown

	interface Ethernet1/2        <--- network SPINES
	  no switchport
	  ip address 172.16.1.3/29
	  no shutdown

#### bgp settings 

	router bgp 65000
	  router-id 30.30.30.30
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 10.0.0.32/30
	  template peer LEAFS
		timers 3 9
		address-family ipv4 unicast
	  template peer SPINES
		remote-as 65000
		update-source Ethernet1/2
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.34
		inherit peer LEAFS
		remote-as 65004
		description LEAF4
		update-source Ethernet1/1
	  neighbor 172.16.1.1
		inherit peer SPINES
		description SPINE1
	  neighbor 172.16.1.2
		inherit peer SPINES
		description SPINE2


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

## leaf2

#### network settings

	interface Ethernet1/1       <--- network spine1   
	  no switchport
	  ip address 10.0.0.6/30
	  no shutdown

	interface Ethernet1/2       <--- network spine2   
	  no switchport
	  ip address 10.0.0.22/30
	  no shutdown

	interface Ethernet1/5       <--- network client2  
	  no switchport
	  ip address 192.168.2.10/24
	  no shutdown

#### bgp settings 

	router bgp 65002
	  router-id 2.2.2.2
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 192.168.2.0/24
	  template peer SPINES
		remote-as 65000
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.5
		inherit peer SPINES
		description SPINE1
		update-source Ethernet1/1
	  neighbor 10.0.0.21
		inherit peer SPINES
		description SPINE2
		update-source Ethernet1/2

## leaf3

#### network settings

	interface Ethernet1/1       <--- network spine1
	  no switchport
	  ip address 10.0.0.10/30
	  no shutdown

	interface Ethernet1/2       <--- network spine2
	  no switchport
	  ip address 10.0.0.26/30
	  no shutdown

	interface Ethernet1/3       <--- network client3
	  no switchport
	  ip address 192.168.3.10/24
	  no shutdown

#### bgp settings 

	router bgp 65003
	  router-id 3.3.3.3
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 192.168.3.0/24
	  template peer SPINES
		remote-as 65000
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.9
		inherit peer SPINES
		description SPINE1
		update-source Ethernet1/1
	  neighbor 10.0.0.25
		inherit peer SPINES
		description SPINE2
		update-source Ethernet1/2


## leaf4

#### network settings

	interface Ethernet1/1        <--- network spine3
	  no switchport
	  ip address 10.0.0.34/30
	  no shutdown

	interface Ethernet1/2        <--- network client4
	  no switchport
	  ip address 192.168.4.10/24
	  no shutdown

#### bgp settings

	router bgp 65004
	  router-id 4.4.4.4
	  bestpath as-path multipath-relax
	  address-family ipv4 unicast
		network 192.168.4.0/24
	  template peer SPINES
		remote-as 65000
		timers 3 9
		address-family ipv4 unicast
	  neighbor 10.0.0.33
		inherit peer SPINES
		description SPINE4
		update-source Ethernet1/1


