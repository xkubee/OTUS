# ЗАДАНИЕ
Необходимо настроить работу vxlan в топологии Clos.
Конкретно, необходимо осуществить объединение в один широковещательный домен устройств CLIENT1 и CLIENT2.
# ПРЕДУСЛОВИЯ
В underlay-сети настроен протокол маршрутизации OSPF.
В топологии все Leaf и Spine устройства имеют информацию о доступности и loopback-интерфейсе каждого из соседей.
P.S. NVE-интерфейсы также работают через loopback-интерфкейсы (именно данный кейс представляет интерес).

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

![evpn vxlan](https://user-images.githubusercontent.com/55625869/147418975-abf36d3e-ec02-45c8-b5fc-992ed920aa5e.PNG)

## Адресное пространство


|    UNIT        |   INTERFACE       |        ADDRESS     |
| :------------- |:-----------------:| ------------------:|
|    client2     |        eth0       | 192.168.2.1/24     |
|    leaf2       |        lo         |   2.2.2.2/32       |
|    spine2      |        lo         | 20.20.20.20/32     |
|    switch      |        lo         | 100.100.100.100/32 |
|    spine3      |        lo         | 30.30.30.30/32     |
|    leaf4       |        lo         | 192.168.4.1/24     |

## Краткая информация об underlay

### Пример таблицы маршрутизации на устройстве leaf2


      2.2.2.2/32, ubest/mbest: 2/0, attached               <-------- свой же loopback
          *via 2.2.2.2, Lo0, [0/0], 01:07:43, local
         *via 2.2.2.2, Lo0, [0/0], 01:07:43, direct
      20.20.20.20/32, ubest/mbest: 1/0                     <-------- loopback spine2
        *via 10.0.0.21, Eth1/2, [110/41], 01:06:31, ospf-1, inter 
      30.30.30.30/32, ubest/mbest: 1/0                     <-------- loopback spine3
         *via 10.0.0.21, Eth1/2, [110/121], 01:06:31, ospf-1, inter
      100.100.100.100/32, ubest/mbest: 1/0                 <-------- loopback switch
         *via 10.0.0.21, Eth1/2, [110/81], 01:06:31, ospf-1, inter
      192.168.2.0/24, ubest/mbest: 1/0, attached           <-------- сеть multicast-сервера
         *via 192.168.2.10, Eth1/5, [0/0], 01:06:38, direct
      192.168.2.10/32, ubest/mbest: 1/0, attached          <-------- адрес multicast-сервера
         *via 192.168.2.10, Eth1/5, [0/0], 01:06:38, local

## Описание настроки оконечных устройств (CLIENT1)

 #### 1. Включить интерфейс VLAN:
      feature interface-vlan

 #### 2. Настроить интерфейс VLAN10:
      
      interface Vlan10
        no shutdown
        ip address 192.168.1.1/24

 #### 3. Настроить интерфейс Ethernet1/1:
      
      interface Ethernet1/1 
        switchport mode trunk
        switchport access vlan 10

 #### По-идее, этого должно быть достаточно, но здесь возникают проблемы: 1. Почему-то отправляется только broadcast-сообщение, а сам трафик не идет. 2. Хочется уметь передавать не только тегированный трафик, но и обычный.

## Описание настроки LEAF-устройств (LEAF2)

### Настройка vn-сегмента

      vlan 10
        vn-segment 10000

### Настройка NVE-интерфейса

 #### 1. Перейти в меню настройки NVE-интерфейса:
      interface nve1
        
 #### 2. Включить его:
      host-reachability protocol bgp
      
 #### 3. Использовать в качестве source-интерфейса loopback-интерфейс:
      source-interface loopback0
      
 #### 4. Прописать, какой vn-сегмент будет настроен в данном nve-интерфейсе:
      member vni 10000
        ingress-replication protocol bgp
 #### 5. Включить возможность передавать информацию о vn-сегменте через EVPN:
       feature nv overlay
        

### Настройка физического интерфейса, направленного в сегмент клиента:
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

      leaf2# show bgp l2vpn evpn
      BGP routing table information for VRF default, address family L2VPN EVPN
      BGP table version is 11, Local Router ID is 2.2.2.2
      Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
      Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
      njected
      Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
      est2

         Network            Next Hop            Metric     LocPrf     Weight Path
      Route Distinguisher: 2.2.2.2:32777    (L2VNI 10000)
      *>l[2]:[0]:[0]:[48]:[5000.0001.0007]:[0]:[0.0.0.0]/216
                            2.2.2.2                           100      32768 i
      *>l[3]:[0]:[32]:[2.2.2.2]/88
                            2.2.2.2                           100      32768 i

