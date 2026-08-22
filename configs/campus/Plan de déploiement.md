
| Phase |     |     |     |     |
| ----- | --- | --- | --- | --- |
|       |     |     |     |     |
# Template ACC-1
```
!=========================================================
! ACC-1 — Phase 1 : socle fonctionnel minimal
! Rôle : switch d'accès L2 pur (bâtiment A)
! Périmètre : VLAN + trunks + SVI management + socle SSH
! HORS périmètre (phase 2) : 802.1X, DHCP snooping, AAA, SNMP, syslog
!=========================================================

hostname ACC-1
!
! --- Base système ---
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
! --- Spanning-Tree ---
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
! --- Déclaration des VLAN (bâtiment A) ---
vlan 10
 name USERS-A
vlan 20
 name VOICE-A
vlan 30
 name IOT-A
vlan 40
 name WIFI-CORP-A
vlan 41
 name WIFI-GUEST-A
vlan 42
 name WIFI-BYOD-A
vlan 50
 name PRINTERS-A
vlan 60
 name PROV-WIRED-A
vlan 61
 name PROV-WIFI-A
vlan 62
 name QUARANTINE-A
vlan 90
 name LAB-DEV-A
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-A
!
! --- SVI de management (accès admin de l'équipement) ---
interface Vlan999
 description MGMT - Administration ACC-1
 ip address 10.254.1.11 255.255.255.0
 no shutdown
!
! Passerelle par défaut du switch (VIP VRRP portée par les distributions)
ip default-gateway 10.254.1.254
!
! --- Ports d'accès (vers postes/clients) ---
interface GigabitEthernet0/2
 description ACCESS - PC1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet3/0
 description ACCESS - Client Linux
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet3/1
 description ACCESS - Client Windows
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet3/2
 description ACCESS - Client IP
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
! --- Ports trunk (vers distribution) ---
interface GigabitEthernet0/0
 description TRUNK - vers DIST-1 Gi1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet0/1
 description TRUNK - vers DIST-2 Gi1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
! --- Ports inutilisés : parking + désactivés ---
interface range GigabitEthernet0/3, GigabitEthernet1/0-3, GigabitEthernet2/0-3, GigabitEthernet3/3
 description UNUSED
 switchport mode access
 switchport access vlan 998
 shutdown
!
! --- Accès SSH (management) ---
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
! --- Durcissement minimal ---
no ip http server
no ip http secure-server
!
end
```

# Template ACC-2
```
hostname ACC-2
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
vlan 10
 name USERS-A
vlan 20
 name VOICE-A
vlan 30
 name IOT-A
vlan 40
 name WIFI-CORP-A
vlan 41
 name WIFI-GUEST-A
vlan 42
 name WIFI-BYOD-A
vlan 50
 name PRINTERS-A
vlan 60
 name PROV-WIRED-A
vlan 61
 name PROV-WIFI-A
vlan 62
 name QUARANTINE-A
vlan 90
 name LAB-DEV-A
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-A
!
interface Vlan999
 description MGMT - Administration ACC-2
 ip address 10.254.1.12 255.255.255.0
 no shutdown
!
ip default-gateway 10.254.1.254
!
interface GigabitEthernet0/2
 description ACCESS - W-CLIENT
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet0/0
 description TRUNK - vers DIST-1 Gi1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet0/1
 description TRUNK - vers DIST-2 Gi1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface range GigabitEthernet0/3, GigabitEthernet1/0-3, GigabitEthernet2/0-3, GigabitEthernet3/0-3
 description UNUSED
 switchport mode access
 switchport access vlan 998
 shutdown
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```
# Template ACC-3 
```
hostname ACC-3
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
vlan 10
 name USERS-B
vlan 20
 name VOICE-B
vlan 30
 name IOT-B
vlan 40
 name WIFI-CORP-B
vlan 41
 name WIFI-GUEST-B
vlan 42
 name WIFI-BYOD-B
vlan 50
 name PRINTERS-B
vlan 60
 name PROV-WIRED-B
vlan 61
 name PROV-WIFI-B
vlan 62
 name QUARANTINE-B
vlan 90
 name LAB-DEV-B
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-B
!
interface Vlan999
 description MGMT - Administration ACC-3
 ip address 10.254.2.11 255.255.255.0
 no shutdown
!
ip default-gateway 10.254.2.254
!
interface GigabitEthernet0/2
 description ACCESS - L-CLIENT
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet0/0
 description TRUNK - vers DIST-3 Gi1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet0/1
 description TRUNK - vers DIST-4 Gi1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface range GigabitEthernet0/3, GigabitEthernet1/0-3, GigabitEthernet2/0-3, GigabitEthernet3/0-3
 description UNUSED
 switchport mode access
 switchport access vlan 998
 shutdown
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```

# Template ACC-4 
```
hostname ACC-4
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
spanning-tree mode rapid-pvst
spanning-tree portfast default
spanning-tree portfast bpduguard default
!
vlan 10
 name USERS-B
vlan 20
 name VOICE-B
vlan 30
 name IOT-B
vlan 40
 name WIFI-CORP-B
vlan 41
 name WIFI-GUEST-B
vlan 42
 name WIFI-BYOD-B
vlan 50
 name PRINTERS-B
vlan 60
 name PROV-WIRED-B
vlan 61
 name PROV-WIFI-B
vlan 62
 name QUARANTINE-B
vlan 90
 name LAB-DEV-B
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-B
!
interface Vlan999
 description MGMT - Administration ACC-4
 ip address 10.254.2.12 255.255.255.0
 no shutdown
!
ip default-gateway 10.254.2.254
!
interface GigabitEthernet0/2
 description ACCESS - PC4
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet0/0
 description TRUNK - vers DIST-3 Gi1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet0/1
 description TRUNK - vers DIST-4 Gi1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface range GigabitEthernet0/3, GigabitEthernet1/0-3, GigabitEthernet2/0-3, GigabitEthernet3/0-3
 description UNUSED
 switchport mode access
 switchport access vlan 998
 shutdown
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```
# Template DIST-1

```
hostname DIST-1
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40,41,42,50,60,61,62,90,999 priority 24576
!
vlan 10
 name USERS-A
vlan 20
 name VOICE-A
vlan 30
 name IOT-A
vlan 40
 name WIFI-CORP-A
vlan 41
 name WIFI-GUEST-A
vlan 42
 name WIFI-BYOD-A
vlan 50
 name PRINTERS-A
vlan 60
 name PROV-WIRED-A
vlan 61
 name PROV-WIFI-A
vlan 62
 name QUARANTINE-A
vlan 90
 name LAB-DEV-A
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-A
!
interface Loopback0
 description MGMT/ROUTER-ID - DIST-1
 ip address 10.0.0.11 255.255.255.255
!
interface Port-channel1
 description LAG-L2 - inter-distribution vers DIST-2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
!
interface GigabitEthernet0/2
 description LAG member - vers DIST-2 Gi0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers DIST-2 Gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers CORE-1 Gi1/0
 no switchport
 ip address 10.0.40.1 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers CORE-2 Gi1/0
 no switchport
 ip address 10.0.40.9 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/0
 description TRUNK - vers ACC-1 Gi0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet1/1
 description TRUNK - vers ACC-2 Gi0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface Vlan10
 description GW USERS-A
 ip address 10.1.10.252 255.255.255.0
 vrrp 10 ip 10.1.10.254
 vrrp 10 priority 110
 vrrp 10 preempt
 no shutdown
!
interface Vlan20
 description GW VOICE-A
 ip address 10.1.20.252 255.255.255.0
 vrrp 20 ip 10.1.20.254
 vrrp 20 priority 110
 vrrp 20 preempt
 no shutdown
!
interface Vlan30
 description GW IOT-A
 ip address 10.1.30.252 255.255.255.0
 vrrp 30 ip 10.1.30.254
 vrrp 30 priority 110
 vrrp 30 preempt
 no shutdown
!
interface Vlan40
 description GW WIFI-CORP-A
 ip address 10.1.40.252 255.255.255.0
 vrrp 40 ip 10.1.40.254
 vrrp 40 priority 110
 vrrp 40 preempt
 no shutdown
!
interface Vlan41
 description GW WIFI-GUEST-A
 ip address 10.1.41.252 255.255.255.0
 vrrp 41 ip 10.1.41.254
 vrrp 41 priority 110
 vrrp 41 preempt
 no shutdown
!
interface Vlan42
 description GW WIFI-BYOD-A
 ip address 10.1.42.252 255.255.255.0
 vrrp 42 ip 10.1.42.254
 vrrp 42 priority 110
 vrrp 42 preempt
 no shutdown
!
interface Vlan50
 description GW PRINTERS-A
 ip address 10.1.50.252 255.255.255.0
 vrrp 50 ip 10.1.50.254
 vrrp 50 priority 110
 vrrp 50 preempt
 no shutdown
!
interface Vlan60
 description GW PROV-WIRED-A
 ip address 10.1.60.252 255.255.255.0
 vrrp 60 ip 10.1.60.254
 vrrp 60 priority 110
 vrrp 60 preempt
 no shutdown
!
interface Vlan61
 description GW PROV-WIFI-A
 ip address 10.1.61.252 255.255.255.0
 vrrp 61 ip 10.1.61.254
 vrrp 61 priority 110
 vrrp 61 preempt
 no shutdown
!
interface Vlan62
 description GW QUARANTINE-A
 ip address 10.1.62.252 255.255.255.0
 vrrp 62 ip 10.1.62.254
 vrrp 62 priority 110
 vrrp 62 preempt
 no shutdown
!
interface Vlan90
 description GW LAB-DEV-A
 ip address 10.1.90.252 255.255.255.0
 vrrp 90 ip 10.1.90.254
 vrrp 90 priority 110
 vrrp 90 preempt
 no shutdown
!
interface Vlan999
 description GW MGMT-LAN-A
 ip address 10.254.1.252 255.255.255.0
 vrrp 99 ip 10.254.1.254
 vrrp 99 priority 110
 vrrp 99 preempt
 no shutdown
!
router ospf 1
 router-id 10.0.0.11
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.11 0.0.0.0 area 0
 network 10.0.40.0 0.0.0.1 area 0
 network 10.0.40.8 0.0.0.1 area 0
 network 10.1.10.0 0.0.0.255 area 0
 network 10.1.20.0 0.0.0.255 area 0
 network 10.1.30.0 0.0.0.255 area 0
 network 10.1.40.0 0.0.0.255 area 0
 network 10.1.41.0 0.0.0.255 area 0
 network 10.1.42.0 0.0.0.255 area 0
 network 10.1.50.0 0.0.0.255 area 0
 network 10.1.60.0 0.0.0.255 area 0
 network 10.1.61.0 0.0.0.255 area 0
 network 10.1.62.0 0.0.0.255 area 0
 network 10.1.90.0 0.0.0.255 area 0
 network 10.254.1.0 0.0.0.255 area 0
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```

# Template DIST-2
```
hostname DIST-2
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40,41,42,50,60,61,62,90,999 priority 28672
!
vlan 10
 name USERS-A
vlan 20
 name VOICE-A
vlan 30
 name IOT-A
vlan 40
 name WIFI-CORP-A
vlan 41
 name WIFI-GUEST-A
vlan 42
 name WIFI-BYOD-A
vlan 50
 name PRINTERS-A
vlan 60
 name PROV-WIRED-A
vlan 61
 name PROV-WIFI-A
vlan 62
 name QUARANTINE-A
vlan 90
 name LAB-DEV-A
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-A
!
interface Loopback0
 description MGMT/ROUTER-ID - DIST-2
 ip address 10.0.0.12 255.255.255.255
!
interface Port-channel1
 description LAG-L2 - inter-distribution vers DIST-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
!
interface GigabitEthernet0/2
 description LAG member - vers DIST-1 Gi0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers DIST-1 Gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers CORE-1 Gi1/1
 no switchport
 ip address 10.0.40.3 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers CORE-2 Gi1/1
 no switchport
 ip address 10.0.40.11 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/0
 description TRUNK - vers ACC-1 Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet1/1
 description TRUNK - vers ACC-2 Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface Vlan10
 description GW USERS-A
 ip address 10.1.10.253 255.255.255.0
 vrrp 10 ip 10.1.10.254
 vrrp 10 priority 100
 vrrp 10 preempt
 no shutdown
!
interface Vlan20
 description GW VOICE-A
 ip address 10.1.20.253 255.255.255.0
 vrrp 20 ip 10.1.20.254
 vrrp 20 priority 100
 vrrp 20 preempt
 no shutdown
!
interface Vlan30
 description GW IOT-A
 ip address 10.1.30.253 255.255.255.0
 vrrp 30 ip 10.1.30.254
 vrrp 30 priority 100
 vrrp 30 preempt
 no shutdown
!
interface Vlan40
 description GW WIFI-CORP-A
 ip address 10.1.40.253 255.255.255.0
 vrrp 40 ip 10.1.40.254
 vrrp 40 priority 100
 vrrp 40 preempt
 no shutdown
!
interface Vlan41
 description GW WIFI-GUEST-A
 ip address 10.1.41.253 255.255.255.0
 vrrp 41 ip 10.1.41.254
 vrrp 41 priority 100
 vrrp 41 preempt
 no shutdown
!
interface Vlan42
 description GW WIFI-BYOD-A
 ip address 10.1.42.253 255.255.255.0
 vrrp 42 ip 10.1.42.254
 vrrp 42 priority 100
 vrrp 42 preempt
 no shutdown
!
interface Vlan50
 description GW PRINTERS-A
 ip address 10.1.50.253 255.255.255.0
 vrrp 50 ip 10.1.50.254
 vrrp 50 priority 100
 vrrp 50 preempt
 no shutdown
!
interface Vlan60
 description GW PROV-WIRED-A
 ip address 10.1.60.253 255.255.255.0
 vrrp 60 ip 10.1.60.254
 vrrp 60 priority 100
 vrrp 60 preempt
 no shutdown
!
interface Vlan61
 description GW PROV-WIFI-A
 ip address 10.1.61.253 255.255.255.0
 vrrp 61 ip 10.1.61.254
 vrrp 61 priority 100
 vrrp 61 preempt
 no shutdown
!
interface Vlan62
 description GW QUARANTINE-A
 ip address 10.1.62.253 255.255.255.0
 vrrp 62 ip 10.1.62.254
 vrrp 62 priority 100
 vrrp 62 preempt
 no shutdown
!
interface Vlan90
 description GW LAB-DEV-A
 ip address 10.1.90.253 255.255.255.0
 vrrp 90 ip 10.1.90.254
 vrrp 90 priority 100
 vrrp 90 preempt
 no shutdown
!
interface Vlan999
 description GW MGMT-LAN-A
 ip address 10.254.1.253 255.255.255.0
 vrrp 99 ip 10.254.1.254
 vrrp 99 priority 100
 vrrp 99 preempt
 no shutdown
!
router ospf 1
 router-id 10.0.0.12
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.12 0.0.0.0 area 0
 network 10.0.40.2 0.0.0.1 area 0
 network 10.0.40.10 0.0.0.1 area 0
 network 10.1.10.0 0.0.0.255 area 0
 network 10.1.20.0 0.0.0.255 area 0
 network 10.1.30.0 0.0.0.255 area 0
 network 10.1.40.0 0.0.0.255 area 0
 network 10.1.41.0 0.0.0.255 area 0
 network 10.1.42.0 0.0.0.255 area 0
 network 10.1.50.0 0.0.0.255 area 0
 network 10.1.60.0 0.0.0.255 area 0
 network 10.1.61.0 0.0.0.255 area 0
 network 10.1.62.0 0.0.0.255 area 0
 network 10.1.90.0 0.0.0.255 area 0
 network 10.254.1.0 0.0.0.255 area 0
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```
# Template DIST-3 
```
hostname DIST-3
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40,41,42,50,60,61,62,90,999 priority 24576
!
vlan 10
 name USERS-B
vlan 20
 name VOICE-B
vlan 30
 name IOT-B
vlan 40
 name WIFI-CORP-B
vlan 41
 name WIFI-GUEST-B
vlan 42
 name WIFI-BYOD-B
vlan 50
 name PRINTERS-B
vlan 60
 name PROV-WIRED-B
vlan 61
 name PROV-WIFI-B
vlan 62
 name QUARANTINE-B
vlan 90
 name LAB-DEV-B
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-B
!
interface Loopback0
 description MGMT/ROUTER-ID - DIST-3
 ip address 10.0.0.13 255.255.255.255
!
interface Port-channel1
 description LAG-L2 - inter-distribution vers DIST-4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
!
interface GigabitEthernet0/2
 description LAG member - vers DIST-4 Gi0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers DIST-4 Gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers CORE-1 Gi1/2
 no switchport
 ip address 10.0.40.5 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers CORE-2 Gi1/2
 no switchport
 ip address 10.0.40.13 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/0
 description TRUNK - vers ACC-3 Gi0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet1/1
 description TRUNK - vers ACC-4 Gi0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface Vlan10
 description GW USERS-B
 ip address 10.1.110.252 255.255.255.0
 vrrp 10 ip 10.1.110.254
 vrrp 10 priority 110
 vrrp 10 preempt
 no shutdown
!
interface Vlan20
 description GW VOICE-B
 ip address 10.1.120.252 255.255.255.0
 vrrp 20 ip 10.1.120.254
 vrrp 20 priority 110
 vrrp 20 preempt
 no shutdown
!
interface Vlan30
 description GW IOT-B
 ip address 10.1.130.252 255.255.255.0
 vrrp 30 ip 10.1.130.254
 vrrp 30 priority 110
 vrrp 30 preempt
 no shutdown
!
interface Vlan40
 description GW WIFI-CORP-B
 ip address 10.1.140.252 255.255.255.0
 vrrp 40 ip 10.1.140.254
 vrrp 40 priority 110
 vrrp 40 preempt
 no shutdown
!
interface Vlan41
 description GW WIFI-GUEST-B
 ip address 10.1.141.252 255.255.255.0
 vrrp 41 ip 10.1.141.254
 vrrp 41 priority 110
 vrrp 41 preempt
 no shutdown
!
interface Vlan42
 description GW WIFI-BYOD-B
 ip address 10.1.142.252 255.255.255.0
 vrrp 42 ip 10.1.142.254
 vrrp 42 priority 110
 vrrp 42 preempt
 no shutdown
!
interface Vlan50
 description GW PRINTERS-B
 ip address 10.1.150.252 255.255.255.0
 vrrp 50 ip 10.1.150.254
 vrrp 50 priority 110
 vrrp 50 preempt
 no shutdown
!
interface Vlan60
 description GW PROV-WIRED-B
 ip address 10.1.160.252 255.255.255.0
 vrrp 60 ip 10.1.160.254
 vrrp 60 priority 110
 vrrp 60 preempt
 no shutdown
!
interface Vlan61
 description GW PROV-WIFI-B
 ip address 10.1.161.252 255.255.255.0
 vrrp 61 ip 10.1.161.254
 vrrp 61 priority 110
 vrrp 61 preempt
 no shutdown
!
interface Vlan62
 description GW QUARANTINE-B
 ip address 10.1.162.252 255.255.255.0
 vrrp 62 ip 10.1.162.254
 vrrp 62 priority 110
 vrrp 62 preempt
 no shutdown
!
interface Vlan90
 description GW LAB-DEV-B
 ip address 10.1.190.252 255.255.255.0
 vrrp 90 ip 10.1.190.254
 vrrp 90 priority 110
 vrrp 90 preempt
 no shutdown
!
interface Vlan999
 description GW MGMT-LAN-B
 ip address 10.254.2.252 255.255.255.0
 vrrp 99 ip 10.254.2.254
 vrrp 99 priority 110
 vrrp 99 preempt
 no shutdown
!
router ospf 1
 router-id 10.0.0.13
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.13 0.0.0.0 area 0
 network 10.0.40.4 0.0.0.1 area 0
 network 10.0.40.12 0.0.0.1 area 0
 network 10.1.110.0 0.0.0.255 area 0
 network 10.1.120.0 0.0.0.255 area 0
 network 10.1.130.0 0.0.0.255 area 0
 network 10.1.140.0 0.0.0.255 area 0
 network 10.1.141.0 0.0.0.255 area 0
 network 10.1.142.0 0.0.0.255 area 0
 network 10.1.150.0 0.0.0.255 area 0
 network 10.1.160.0 0.0.0.255 area 0
 network 10.1.161.0 0.0.0.255 area 0
 network 10.1.162.0 0.0.0.255 area 0
 network 10.1.190.0 0.0.0.255 area 0
 network 10.254.2.0 0.0.0.255 area 0
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
```


# Template DIST-4 
```
hostname DIST-4
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,40,41,42,50,60,61,62,90,999 priority 28672
!
vlan 10
 name USERS-B
vlan 20
 name VOICE-B
vlan 30
 name IOT-B
vlan 40
 name WIFI-CORP-B
vlan 41
 name WIFI-GUEST-B
vlan 42
 name WIFI-BYOD-B
vlan 50
 name PRINTERS-B
vlan 60
 name PROV-WIRED-B
vlan 61
 name PROV-WIFI-B
vlan 62
 name QUARANTINE-B
vlan 90
 name LAB-DEV-B
vlan 997
 name NATIVE-UNUSED
vlan 998
 name PARKING
vlan 999
 name MGMT-LAN-B
!
interface Loopback0
 description MGMT/ROUTER-ID - DIST-4
 ip address 10.0.0.14 255.255.255.255
!
interface Port-channel1
 description LAG-L2 - inter-distribution vers DIST-3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
!
interface GigabitEthernet0/2
 description LAG member - vers DIST-3 Gi0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers DIST-3 Gi0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers CORE-1 Gi1/3
 no switchport
 ip address 10.0.40.7 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers CORE-2 Gi1/3
 no switchport
 ip address 10.0.40.15 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/0
 description TRUNK - vers ACC-3 Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface GigabitEthernet1/1
 description TRUNK - vers ACC-4 Gi0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 997
 switchport trunk allowed vlan 10,20,30,40,41,42,50,60,61,62,90,999
 no shutdown
!
interface Vlan10
 description GW USERS-B
 ip address 10.1.110.253 255.255.255.0
 vrrp 10 ip 10.1.110.254
 vrrp 10 priority 100
 vrrp 10 preempt
 no shutdown
!
interface Vlan20
 description GW VOICE-B
 ip address 10.1.120.253 255.255.255.0
 vrrp 20 ip 10.1.120.254
 vrrp 20 priority 100
 vrrp 20 preempt
 no shutdown
!
interface Vlan30
 description GW IOT-B
 ip address 10.1.130.253 255.255.255.0
 vrrp 30 ip 10.1.130.254
 vrrp 30 priority 100
 vrrp 30 preempt
 no shutdown
!
interface Vlan40
 description GW WIFI-CORP-B
 ip address 10.1.140.253 255.255.255.0
 vrrp 40 ip 10.1.140.254
 vrrp 40 priority 100
 vrrp 40 preempt
 no shutdown
!
interface Vlan41
 description GW WIFI-GUEST-B
 ip address 10.1.141.253 255.255.255.0
 vrrp 41 ip 10.1.141.254
 vrrp 41 priority 100
 vrrp 41 preempt
 no shutdown
!
interface Vlan42
 description GW WIFI-BYOD-B
 ip address 10.1.142.253 255.255.255.0
 vrrp 42 ip 10.1.142.254
 vrrp 42 priority 100
 vrrp 42 preempt
 no shutdown
!
interface Vlan50
 description GW PRINTERS-B
 ip address 10.1.150.253 255.255.255.0
 vrrp 50 ip 10.1.150.254
 vrrp 50 priority 100
 vrrp 50 preempt
 no shutdown
!
interface Vlan60
 description GW PROV-WIRED-B
 ip address 10.1.160.253 255.255.255.0
 vrrp 60 ip 10.1.160.254
 vrrp 60 priority 100
 vrrp 60 preempt
 no shutdown
!
interface Vlan61
 description GW PROV-WIFI-B
 ip address 10.1.161.253 255.255.255.0
 vrrp 61 ip 10.1.161.254
 vrrp 61 priority 100
 vrrp 61 preempt
 no shutdown
!
interface Vlan62
 description GW QUARANTINE-B
 ip address 10.1.162.253 255.255.255.0
 vrrp 62 ip 10.1.162.254
 vrrp 62 priority 100
 vrrp 62 preempt
 no shutdown
!
interface Vlan90
 description GW LAB-DEV-B
 ip address 10.1.190.253 255.255.255.0
 vrrp 90 ip 10.1.190.254
 vrrp 90 priority 100
 vrrp 90 preempt
 no shutdown
!
interface Vlan999
 description GW MGMT-LAN-B
 ip address 10.254.2.253 255.255.255.0
 vrrp 99 ip 10.254.2.254
 vrrp 99 priority 100
 vrrp 99 preempt
 no shutdown
!
router ospf 1
 router-id 10.0.0.14
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.14 0.0.0.0 area 0
 network 10.0.40.6 0.0.0.1 area 0
 network 10.0.40.14 0.0.0.1 area 0
 network 10.1.110.0 0.0.0.255 area 0
 network 10.1.120.0 0.0.0.255 area 0
 network 10.1.130.0 0.0.0.255 area 0
 network 10.1.140.0 0.0.0.255 area 0
 network 10.1.141.0 0.0.0.255 area 0
 network 10.1.142.0 0.0.0.255 area 0
 network 10.1.150.0 0.0.0.255 area 0
 network 10.1.160.0 0.0.0.255 area 0
 network 10.1.161.0 0.0.0.255 area 0
 network 10.1.162.0 0.0.0.255 area 0
 network 10.1.190.0 0.0.0.255 area 0
 network 10.254.2.0 0.0.0.255 area 0
!
username admin privilege 15 secret azerty
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```
# Template CORE-1 
```
hostname CORE-1
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
!
interface Loopback0
 description MGMT/ROUTER-ID - CORE-1
 ip address 10.0.0.1 255.255.255.255
!
interface Port-channel1
 description LAG-L3 - inter-core vers CORE-2
 no switchport
 ip address 10.0.40.16 255.255.255.254
 ip ospf network point-to-point
!
interface GigabitEthernet0/2
 description LAG member - vers CORE-2 Gi0/2
 no switchport
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers CORE-2 Gi0/3
 no switchport
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet1/0
 description L3-ROUTED - vers DIST-1 Gi0/0
 no switchport
 ip address 10.0.40.0 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/1
 description L3-ROUTED - vers DIST-2 Gi0/0
 no switchport
 ip address 10.0.40.2 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/2
 description L3-ROUTED - vers DIST-3 Gi0/0
 no switchport
 ip address 10.0.40.4 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/3
 description L3-ROUTED - vers DIST-4 Gi0/0
 no switchport
 ip address 10.0.40.6 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers FW-1 em4
 no switchport
 ip address 10.0.30.1 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers FW-2 em4
 no switchport
 ip address 10.0.30.5 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
router ospf 1
 router-id 10.0.0.1
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface GigabitEthernet1/0
 no passive-interface GigabitEthernet1/1
 no passive-interface GigabitEthernet1/2
 no passive-interface GigabitEthernet1/3
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.1 0.0.0.0 area 0
 network 10.0.40.16 0.0.0.1 area 0
 network 10.0.40.0 0.0.0.1 area 0
 network 10.0.40.2 0.0.0.1 area 0
 network 10.0.40.4 0.0.0.1 area 0
 network 10.0.40.6 0.0.0.1 area 0
 network 10.0.30.0 0.0.0.1 area 0
 network 10.0.30.4 0.0.0.1 area 0
!
username admin privilege 15 secret VOTRE_MDP_FORT
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
!
end
```

# Template CORE-2 
```
hostname CORE-2
!
no ip domain-lookup
ip domain-name nomai.lab
service timestamps log datetime msec
service timestamps debug datetime msec
service password-encryption
!
ip routing
!
spanning-tree mode rapid-pvst
!
interface Loopback0
 description MGMT/ROUTER-ID - CORE-2
 ip address 10.0.0.2 255.255.255.255
!
interface Port-channel1
 description LAG-L3 - inter-core vers CORE-1
 no switchport
 ip address 10.0.40.17 255.255.255.254
 ip ospf network point-to-point
!
interface GigabitEthernet0/2
 description LAG member - vers CORE-1 Gi0/2
 no switchport
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet0/3
 description LAG member - vers CORE-1 Gi0/3
 no switchport
 channel-group 1 mode on
 no shutdown
!
interface GigabitEthernet1/0
 description L3-ROUTED - vers DIST-1 Gi0/1
 no switchport
 ip address 10.0.40.8 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/1
 description L3-ROUTED - vers DIST-2 Gi0/1
 no switchport
 ip address 10.0.40.10 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/2
 description L3-ROUTED - vers DIST-3 Gi0/1
 no switchport
 ip address 10.0.40.12 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet1/3
 description L3-ROUTED - vers DIST-4 Gi0/1
 no switchport
 ip address 10.0.40.14 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/0
 description L3-ROUTED - vers FW-1 em5
 no switchport
 ip address 10.0.30.3 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
interface GigabitEthernet0/1
 description L3-ROUTED - vers FW-2 em5
 no switchport
 ip address 10.0.30.7 255.255.255.254
 ip ospf network point-to-point
 no shutdown
!
router ospf 1
 router-id 10.0.0.2
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface GigabitEthernet1/0
 no passive-interface GigabitEthernet1/1
 no passive-interface GigabitEthernet1/2
 no passive-interface GigabitEthernet1/3
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
 network 10.0.0.2 0.0.0.0 area 0
 network 10.0.40.16 0.0.0.1 area 0
 network 10.0.40.8 0.0.0.1 area 0
 network 10.0.40.10 0.0.0.1 area 0
 network 10.0.40.12 0.0.0.1 area 0
 network 10.0.40.14 0.0.0.1 area 0
 network 10.0.30.2 0.0.0.1 area 0
 network 10.0.30.6 0.0.0.1 area 0
!
username admin privilege 15 secret VOTRE_MDP_FORT
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
line vty 0 4
 transport input ssh
 login local
 exec-timeout 15 0
!
no ip http server
no ip http secure-server
```