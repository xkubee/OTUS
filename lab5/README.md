# ЗАДАНИЕ
Необходимо настроить работу multicast-трафика в топологии Clos.
Конкретно, необходимо осуществить возможность принимать устройству Leaf4 (оно выступает в качестве клиента) multicast-трафик от устройства client 2 (оно выступает в качестве сервера).
# ПРЕДУСЛОВИЯ
В underlay-сети настроен протокол маршрутизации OSPF.
В топологии все Leaf и Spine устройства имеют информацию о доступности и loopback-интерфейсе каждого из соседей.
P.S. Именно к loopback-интерфейсам необходимо привязывать работу протоколов OSPF и PIM для правильности реализации и чёткого понимания принципов работы с multicast-трафиком.

Также, в силу нехватки ресурсов на сервере (в ближайшее время мне должны дать новый стенд), пришлось использовать только часть топологии.
Она включает в себя 6 устройств:

1. client2 - multicast-сервер
2. leaf2   - leaf-устройство
3. spine2  - spine-устройство
4. switch  - spine-устройство, выступающее в качестве RP в данной топологии
5. spine3  - spine-устройство, соединяющее второй ЦОД с первым
6. leaf4   - оконечное устройство, выступающее в качестве multicast-клиента (устройство подписано на группу 239.3.3.3)

# РЕШЕНИЕ

## Схема сети

![tempsnip](https://user-images.githubusercontent.com/55625869/145262722-6d4c3bda-a26c-4009-8723-3e3725ae4ca3.png)

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

### Описание настройки протокола PIM-SM на CISCO NX-OS

 1. Необходимо включить протокол PIM:
  feature pim
 2. Настроить использование loopback-интерфейса в качестве "обменника" PIM-сообщениями:
  ip pim register-source loopback 0
 3. Включить протокол PIM-SM на всех интерфейсах, которым нужно будет обмениваться PIM-сообщениями:
  ip pim sparse-mode
 4. !!! Включить протокол PIM-SM на loopback-интерфейсе!!!:
  interface loopback 0
   ip pim sparse-mode
 5. Указать адрес RP (устройству, которое будет строить "деревья", по которым будет передаваться multicast-трафик)

 Данных настроек необходимо и достаточно для того, чтобы протокол PIM-SM правильно заработал в сети.

### Пример конфигурации протокола PIM-SM на устройстве leaf2

			ip pim rp-address 100.100.100.100 group-list 224.0.0.0/4
			ip pim ssm range 232.0.0.0/8
			ip pim register-source loopback0

			interface Ethernet1/1
			  no switchport
			  ip address 10.0.0.6/30
			  ip ospf network point-to-point
			  no ip ospf passive-interface
			  ip router ospf 1 area 0.0.0.2
			  ip pim sparse-mode
			  no shutdown

			interface Ethernet1/2
			  no switchport
			  ip address 10.0.0.22/30
			  ip ospf network point-to-point
			  no ip ospf passive-interface
			  ip router ospf 1 area 0.0.0.2
			  ip pim sparse-mode
			  no shutdown

			interface Ethernet1/5
			  no switchport
			  ip address 192.168.2.10/24
			  ip ospf network point-to-point
			  no ip ospf passive-interface
			  ip router ospf 1 area 0.0.0.2
			  ip pim sparse-mode
			  no shutdown

			interface loopback0
			  ip address 2.2.2.2/32
			  ip ospf network point-to-point
			  ip router ospf 1 area 0.0.0.2
			  ip pim sparse-mode

