# ЗАДАНИЕ
Необходимо настроить работу vxlan в топологии Clos.
Конкретно, необходимо осуществить объединение при помощи VXLAN EVPN устройств CLIENT1 и CLIENT2, находящихся в разных подсетях.
# ПРЕДУСЛОВИЯ
В underlay-сети настроен протокол маршрутизации OSPF.
В топологии все Leaf и Spine устройства имеют информацию о доступности и loopback-интерфейсе каждого из соседей.
P.S. NVE-интерфейсы также работают через loopback-интерфейсы (именно данный кейс представляет интерес).

Топология включает в себя 7 устройств:

1. CLIENT1 - оконечное устройство
2. LEAF2   - leaf-устройство
3. SPINE2  - spine-устройство
4. SUPERSPINE  - superspine-устройство
5. SPINE3  - spine-устройство, соединяющее второй ЦОД с первым
6. LEAF4   - leaf-устройство
7. CLIENT2 - оконечное устройство

# РЕШЕНИЕ

## Схема сети

![evpn vxlan](https://user-images.githubusercontent.com/55625869/147421089-25426026-a294-4086-b6b9-0df5455a76e3.PNG)

## Адресное пространство


|    UNIT        |   INTERFACE       |        ADDRESS     |
| :------------- |:-----------------:| ------------------:|
|    CLIENT1     |        e1/1       | 192.168.1.1/24     |
|    LEAF2       |        lo         |   2.2.2.2/32       |
|    SPINE2      |        lo         | 20.20.20.20/32     |
|    SSPINE      |        lo         | 100.100.100.100/32 |
|    SPINE3      |        lo         | 30.30.30.30/32     |
|    LEAF4       |        lo         |   4.4.4.4/32       |
|    CLIENT2     |        e1/1       | 192.168.2.1/24     |

## Краткая информация об underlay

### Пример таблицы маршрутизации на устройстве leaf2

      2.2.2.2/32, ubest/mbest: 2/0, attached      <--------------
          *via 2.2.2.2, Lo0, [0/0], 4d02h, local 
          *via 2.2.2.2, Lo0, [0/0], 4d02h, direct
      4.4.4.4/32, ubest/mbest: 1/0                <--------------
          *via 10.0.0.21, Eth1/2, [110/161], 00:29:55, ospf-1, inter
      10.0.0.20/30, ubest/mbest: 1/0, attached
          *via 10.0.0.22, Eth1/2, [0/0], 4d02h, direct
      10.0.0.22/32, ubest/mbest: 1/0, attached
          *via 10.0.0.22, Eth1/2, [0/0], 4d02h, local
      10.0.0.24/30, ubest/mbest: 1/0
          *via 10.0.0.21, Eth1/2, [110/80], 4d02h, ospf-1, intra
      10.0.0.28/30, ubest/mbest: 1/0
          *via 10.0.0.21, Eth1/2, [110/80], 4d02h, ospf-1, inter
      10.0.0.32/30, ubest/mbest: 1/0
          *via 10.0.0.21, Eth1/2, [110/160], 4d02h, ospf-1, inter
      10.0.0.36/30, ubest/mbest: 1/0
          *via 10.0.0.21, Eth1/2, [110/120], 4d02h, ospf-1, inter
      20.20.20.20/32, ubest/mbest: 1/0          <--------------
          *via 10.0.0.21, Eth1/2, [110/41], 4d02h, ospf-1, inter
      30.30.30.30/32, ubest/mbest: 1/0          <--------------
          *via 10.0.0.21, Eth1/2, [110/121], 4d02h, ospf-1, inter
      100.100.100.100/32, ubest/mbest: 1/0      <--------------
          *via 10.0.0.21, Eth1/2, [110/81], 4d02h, ospf-1, inter

## Описание настроки LEAF-устройств (LEAF2)

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

      router bgp 65000        <--------------- настройка mBGP (настройка overlay)
        neighbor 20.20.20.20
          remote-as 65000
          update-source loopback0
          address-family l2vpn evpn 
            send-community
            send-community extended
            route-reflector-client

## Описание настроки SPINE-устройств (SPINE2)

      router bgp 65000
        neighbor 2.2.2.2
          remote-as 65000
          update-source loopback0
          address-family l2vpn evpn
            send-community
            send-community extended
            route-reflector-client
        neighbor 100.100.100.100
          remote-as 65000
          update-source loopback0
          address-family l2vpn evpn
            send-community
            send-community extended
            route-reflector-client

## Проверка работоспособности стенда

### LEAF4

      leaf4# show bgp l2vpn evpn
      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 112, Local Router ID is 4.4.4.4
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

      Route Distinguisher: 4.4.4.4:32777    (L2VNI 20000)
      *>l[2]:[0]:[0]:[48]:[5000.0006.0007]:[0]:[0.0.0.0]/216
                            4.4.4.4                           100      32768 i
      *>l[2]:[0]:[0]:[48]:[5000.0006.0007]:[32]:[192.168.2.1]/272
                            4.4.4.4                           100      32768 i
      *>l[3]:[0]:[32]:[4.4.4.4]/88
                            4.4.4.4                           100      32768 i

### LEAF2

      leaf2# show bgp l2vpn evpn
      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 106, Local Router ID is 2.2.2.2
      Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
      Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
      njected
      Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
      est2

         Network            Next Hop            Metric     LocPrf     Weight Path
      Route Distinguisher: 2.2.2.2:32777    (L2VNI 10000)
      *>l[2]:[0]:[0]:[48]:[5000.0001.0007]:[0]:[0.0.0.0]/216
                            2.2.2.2                           100      32768 i
      *>l[2]:[0]:[0]:[48]:[5000.0001.0007]:[32]:[192.168.1.1]/272
                            2.2.2.2                           100      32768 i
      *>l[3]:[0]:[32]:[2.2.2.2]/88
                            2.2.2.2                           100      32768 i

      Route Distinguisher: 4.4.4.4:32777
      *>i[2]:[0]:[0]:[48]:[5000.0006.0007]:[0]:[0.0.0.0]/216
                            4.4.4.4                           100          0 i
      *>i[2]:[0]:[0]:[48]:[5000.0006.0007]:[32]:[192.168.2.1]/272
                            4.4.4.4                           100          0 i
      *>i[3]:[0]:[32]:[4.4.4.4]/88
                            4.4.4.4                           100          0 i

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

      client2# ping 192.168.2.1
      PING 192.168.1.1 (192.168.1.1): 56 data bytes
      64 bytes from 192.168.1.1: icmp_seq=0 ttl=254 time=47.973 ms
      64 bytes from 192.168.1.1: icmp_seq=1 ttl=254 time=55.707 ms
      64 bytes from 192.168.1.1: icmp_seq=2 ttl=254 time=41.055 ms
      64 bytes from 192.168.1.1: icmp_seq=3 ttl=254 time=47.054 ms
      64 bytes from 192.168.1.1: icmp_seq=4 ttl=254 time=51.129 ms

      --- 192.168.1.1 ping statistics ---
      5 packets transmitted, 5 packets received, 0.00% packet loss
      round-trip min/avg/max = 41.055/48.583/55.707 ms
