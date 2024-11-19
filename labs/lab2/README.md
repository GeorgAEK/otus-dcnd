### План работы
1. Настройка OSPF в Underlay сети.
2. Проверка работы OSPF.

#### 1. Настройка OSPF в Underlay сети
Схема сети имеет вид:
![[1.2.Net-diagram.png]]

Учитывая выбранный шаблон назначения адресов, будем использовать следующие IP адреса:

| hostname     | lo0         | p2p sp1      | p2p sp2      |
| ------------ | ----------- | ------------ | ------------ |
| dc1-pod1-sp1 | 10.1.10.201 | -            | -            |
| dc1-pod1-sp2 | 10.1.10.202 | -            | -            |
| dc1-pod1-lf1 | 10.1.10.1   | 10.1.11.1/31 | 10.1.12.1/31 |
| dc1-pod1-lf2 | 10.1.10.2   | 10.1.11.3/31 | 10.1.12.3/31 |
| dc1-pod1-lf3 | 10.1.10.3   | 10.1.11.5/31 | 10.1.12.5/31 |

В качестве протокола динамической маршрутизации выбран OSPF. Так как мы ожидаем, что Underlay сеть потенциально может включать несколько подов, то каждый под поместим в свою area с типом NSSA. Соответственно, шаблон для номера area будет иметь вид:
**area {PODN},**
где
	{PODN} - номер пода (POD Number). Принимает значение от 1 до 16.

Backbone area (area 0) будет настроена на уровне spine--superspine.

Далее приведем конфигурации устройств.
Настройки spine на примере sp1:
```
! Command: show running-config
! device: dc1-pod1-sp1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname dc1-pod1-sp1
!
spanning-tree mode mstp
!
interface Ethernet1
   description lf1|Eth1
   no switchport
   ip address 10.1.11.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet2
   description lf2|Eth1
   no switchport
   ip address 10.1.11.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet3
   description lf3|Eth1
   no switchport
   ip address 10.1.11.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.1.10.201/32
   ip ospf area 0.0.0.1
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.1.10.201
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   area 0.0.0.1 nssa
   max-lsa 12000
!
end
```

Настройки leaf на примере lf3:
```
! Command: show running-config
! device: dc1-pod1-lf3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname dc1-pod1-lf3
!
spanning-tree mode mstp
!
interface Ethernet1
   description sp1|Eth3
   no switchport
   ip address 10.1.11.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet2
   description sp2|Eth3
   no switchport
   ip address 10.1.12.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.1
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.1.10.3/32
   ip ospf area 0.0.0.1
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.1.10.3
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   area 0.0.0.1 nssa
   max-lsa 12000
!
end
```


#### 2. Проверка работы OSPF

Проверка таблицы маршрутизации (на примере lf3):
```
dc1-pod1-lf3#show ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 O        10.1.10.1/32 [110/30] via 10.1.11.4, Ethernet1
                                via 10.1.12.4, Ethernet2
 O        10.1.10.2/32 [110/30] via 10.1.11.4, Ethernet1
                                via 10.1.12.4, Ethernet2
 C        10.1.10.3/32 is directly connected, Loopback0
 O        10.1.10.201/32 [110/20] via 10.1.11.4, Ethernet1
 O        10.1.10.202/32 [110/20] via 10.1.12.4, Ethernet2
 O        10.1.11.0/31 [110/20] via 10.1.11.4, Ethernet1
 O        10.1.11.2/31 [110/20] via 10.1.11.4, Ethernet1
 C        10.1.11.4/31 is directly connected, Ethernet1
 O        10.1.12.0/31 [110/20] via 10.1.12.4, Ethernet2
 O        10.1.12.2/31 [110/20] via 10.1.12.4, Ethernet2
 C        10.1.12.4/31 is directly connected, Ethernet2
```

Проверка связности адресов lo0 коммутаторов leaf (на примере lf3):
```
dc1-pod1-lf3#traceroute 10.1.10.1 source loopback 0
traceroute to 10.1.10.1 (10.1.10.1), 30 hops max, 60 byte packets
 1  10.1.11.4 (10.1.11.4)  8.998 ms  7.343 ms  7.594 ms
 2  10.1.10.1 (10.1.10.1)  18.846 ms  19.224 ms  42.170 ms
dc1-pod1-lf3#
dc1-pod1-lf3#traceroute 10.1.10.2 source loopback 0
traceroute to 10.1.10.2 (10.1.10.2), 30 hops max, 60 byte packets
 1  10.1.11.4 (10.1.11.4)  6.420 ms  6.630 ms  6.741 ms
 2  10.1.10.2 (10.1.10.2)  15.386 ms  14.524 ms  21.179 ms
```

Проверка OSPF database (на примере lf3):
```
dc1-pod1-lf3#show ip ospf database

            OSPF Router with ID(10.1.10.3) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.1)

Link ID         ADV Router      Age         Seq#         Checksum Link count
10.1.10.202     10.1.10.202     258         0x80000009   0x15f6   7
10.1.10.2       10.1.10.2       250         0x80000007   0x2929   5
10.1.10.1       10.1.10.1       254         0x80000008   0xc50    5
10.1.10.201     10.1.10.201     281         0x8000000a   0xd93a   7
10.1.10.3       10.1.10.3       245         0x80000008   0x4204   5
```

Проверка BFD (на примере sp1):
```
dc1-pod1-sp1#show bfd peers
VRF name: default
-----------------
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp
---------- ----------- ----------- -------------------- ------- ---------------
10.1.11.1  2114312902    47509848        Ethernet1(13)  normal  11/19/24 08:07
10.1.11.3  3580667884  2414719360        Ethernet2(14)  normal  11/19/24 08:11
10.1.11.5  4183777481  2014550263        Ethernet3(15)  normal  11/19/24 08:11

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
```
