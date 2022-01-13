# ЗАДАНИЕ
Необходимо настроить работу VXLAN-MULTIPOD в топологии Clos.
Конкретно, необходимо осуществить объединение при помощи технологии VXLAN-MULTIPOD разные ЦОД'ы.
# ПРЕДУСЛОВИЯ
В underlay-сети настроен протокол маршрутизации OSPF.
В топологии все Leaf и Spine устройства имеют информацию о доступности и loopback-интерфейсе каждого из соседей.

Топология включает в себя 5 устройств:

1. LEAF2   - оконечное устройство
2. LEAF4   - оконечное устройство
3. SPINE2  - spine-устройство (устройство, на котором настроен EVPN VXLAN) - POD1
4. SPINE3  - spine-устройство (устройство, на котором настроен EVPN VXLAN) - POD2
7. LEAF4   - leaf-устройство
8. CLIENT2 - оконечное устройство

# РЕШЕНИЕ

## Схема сети

![VXLAN-MULTIPOD](https://user-images.githubusercontent.com/55625869/149390561-7ac9c1a5-34ac-4373-9bb8-96870d6b1697.PNG)


## Адресное пространство

|    UNIT        |   INTERFACE       |        ADDRESS     |
| :------------- |:-----------------:| ------------------:|
|    LEAF2       |        lo         |  192.168.1.1/24    |
|    SPINE2      |        lo         | 20.20.20.20/32     |
|    SSPINE      |        lo         | 100.100.100.100/32 |
|    SPINE3      |        lo         | 30.30.30.30/32     |
|    LEAF4       |        lo         |   192.168.2.1/24   |

## Описание настроки LEAF-устройств (SPINE2)

### Включение нужных сервисов

      nv overlay evpn
      feature ospf
      feature bgp
      feature interface-vlan
      feature vn-segment-vlan-based
      feature nv overlay

### Настройка виртуального mac-адреса

      fabric forwarding anycast-gateway-mac 0001.0001.0001

### Настройки VLAN-ов

      vlan 1,10,99
      vlan 10
        vn-segment 10000
      vlan 99
        vn-segment 9999

### Настройка vrf (vrf выступает в качестве связующего звена между интерфейсом vlan10 и интерфейсом vlan99)
      
      vrf context L3VNI
        vni 9999
        address-family ipv4 unicast
          route-target import 9999:9999     
          route-target import 9999:9999 evpn
          route-target export 9999:9999
          route-target export 9999:9999 evpn
          route-target both auto
          route-target both auto evpn

### Настройки интерфейсов (VLAN-интерфейсов, nve-интерфейса, физического интерфейса):

      interface Vlan10
        no shutdown
        vrf member L3VNI
        ip address 192.168.1.2/24
        fabric forwarding mode anycast-gateway

      interface Vlan99
        no shutdown
        vrf member L3VNI
        ip forward

      interface nve1
        no shutdown
        host-reachability protocol bgp
        source-interface loopback0
        member vni 9999 associate-vrf
        member vni 10000
          ingress-replication protocol bgp

      interface Ethernet1/5
        switchport mode trunk

### Настройка BGP

      router bgp 65001  <----- новая AS
        neighbor 100.100.100.100
          remote-as 65000
          update-source loopback0
          ebgp-multihop 5
          address-family l2vpn evpn
            send-community
            send-community extended

## Описание настроки устройства SUPERSPINE

### Настройка BGP

      router bgp 65000
        address-family l2vpn evpn
          retain route-target all
        neighbor 20.20.20.20
          remote-as 65001         <----- AS POD1
          update-source loopback0
          ebgp-multihop 5
          address-family l2vpn evpn
            send-community
            send-community extended
            route-map NONH out
        neighbor 30.30.30.30
          remote-as 65002        <----- AS POD2
          update-source loopback0
          ebgp-multihop 5
          address-family l2vpn evpn
            send-community
            send-community extended
            route-map NONH out

## Проверка работоспособности стенда

### SPINE2

      spine2# show bgp l2vpn evpn
      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 13, Local Router ID is 20.20.20.20
      Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
      Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
      Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

         Network            Next Hop            Metric     LocPrf     Weight Path
      Route Distinguisher: 20.20.20.20:32777    (L2VNI 10000)
      *>l[2]:[0]:[0]:[48]:[5000.0002.0007]:[0]:[0.0.0.0]/216
                            20.20.20.20                       100      32768 i
      *>l[2]:[0]:[0]:[48]:[5000.0002.0007]:[32]:[192.168.1.1]/272
                            20.20.20.20                       100      32768 i
      *>l[3]:[0]:[32]:[20.20.20.20]/88
                            20.20.20.20                       100      32768 i

      Route Distinguisher: 30.30.30.30:32777
      *>e[2]:[0]:[0]:[48]:[5000.000a.0007]:[32]:[192.168.2.1]/272
                            30.30.30.30                                    0 65000 65002 i

### SPINE3

      spine3# show bgp l2vpn evpn
      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 21, Local Router ID is 30.30.30.30
      Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
      Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
      Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

         Network            Next Hop            Metric     LocPrf     Weight Path
      Route Distinguisher: 20.20.20.20:32777
      *>e[2]:[0]:[0]:[48]:[5000.0002.0007]:[0]:[0.0.0.0]/216
                            20.20.20.20                                    0 65000 65001 i
      *>e[2]:[0]:[0]:[48]:[5000.0002.0007]:[32]:[192.168.1.1]/272
                            20.20.20.20                                    0 65000 65001 i
      *>e[3]:[0]:[32]:[20.20.20.20]/88
                            20.20.20.20                                    0 65000 65001 i

      Route Distinguisher: 30.30.30.30:32777    (L2VNI 20000)
      *>l[2]:[0]:[0]:[48]:[5000.000a.0007]:[0]:[0.0.0.0]/216
                            30.30.30.30                       100      32768 i
      *>l[2]:[0]:[0]:[48]:[5000.000a.0007]:[32]:[192.168.2.1]/272
                            30.30.30.30                       100      32768 i
      *>l[3]:[0]:[32]:[30.30.30.30]/88
                            30.30.30.30                       100      32768 i

### SUPERSPINE

      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 101, Local Router ID is 100.100.100.100
      Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
      Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
      njected
      Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
      est2

         Network            Next Hop            Metric     LocPrf     Weight Path
      Route Distinguisher: 2.2.2.2:32777
      *>i[2]:[0]:[0]:[48]:[5000.0001.0007]:[0]:[0.0.0.0]/216
                            2.2.2.2                           100          0 i
      *>i[2]:[0]:[0]:[48]:[5000.0001.0007]:[32]:[192.168.1.1]/272
                            2.2.2.2                           100          0 i
      *>i[3]:[0]:[32]:[2.2.2.2]/88
                            2.2.2.2                           100          0 i

      Route Distinguisher: 4.4.4.4:32777
      *>i[2]:[0]:[0]:[48]:[5000.0006.0007]:[0]:[0.0.0.0]/216
                            4.4.4.4                           100          0 i
      *>i[2]:[0]:[0]:[48]:[5000.0006.0007]:[32]:[192.168.2.1]/272
                            4.4.4.4                           100          0 i
      *>i[3]:[0]:[32]:[4.4.4.4]/88
                            4.4.4.4                           100          0 i

### Сетевая связность между CLIENT1 и CLIENT2

      client2# ping 192.168.1.1
      PING 192.168.1.1 (192.168.1.1): 56 data bytes
      64 bytes from 192.168.1.1: icmp_seq=0 ttl=254 time=42.778 ms
      64 bytes from 192.168.1.1: icmp_seq=1 ttl=254 time=59.705 ms
      64 bytes from 192.168.1.1: icmp_seq=2 ttl=254 time=11.055 ms
      64 bytes from 192.168.1.1: icmp_seq=3 ttl=254 time=17.014 ms
      64 bytes from 192.168.1.1: icmp_seq=4 ttl=254 time=11.229 ms

      --- 192.168.1.1 ping statistics ---
      5 packets transmitted, 5 packets received, 0.00% packet loss
      round-trip min/avg/max = 43.056/49.583/51.727 ms
