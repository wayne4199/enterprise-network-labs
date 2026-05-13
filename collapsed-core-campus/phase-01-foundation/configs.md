# Phase 01 - Foundation Configurations

# Build Flow

1. Base VLAN Configuration
2. Trunk Configuration
3. STP Root Placement
4. SVI Configuration
5. HSRP Configuration
6. End Host Static IP Configuration

# STEP 1 — Base Device Configuration

## CORE1

hostname CORE1

no ip domain-lookup

spanning-tree mode rapid-pvst

## CORE2

hostname CORE2

no ip domain-lookup

spanning-tree mode rapid-pvst

## ACC1

hostname ACC1

no ip domain-lookup

spanning-tree mode rapid-pvst

## ACC2

hostname ACC2

no ip domain-lookup

spanning-tree mode rapid-pvst

## ACC3

hostname ACC3

no ip domain-lookup

spanning-tree mode rapid-pvst


# STEP 2 — VLAN Configuration

## CORE1 / CORE2 / ACC1 / ACC2 / ACC3

vlan 10

name USERS

vlan 20

name SERVERS

vlan 30

name VOICE

vlan 40

name MGMT

vlan 50

name GUEST

vlan 999

name NATIVE-BLACKHOLE


# STEP 3 — Trunk Configuration

## CORE1

interface range g0/1-3,g1/0

switchport trunk encapsulation dot1q

switchport mode trunk

switchport trunk native vlan 999

no shutdown

interface g0/1

description TO-CORE2

interface g0/2

description TO-ACC1

interface g0/3

description TO-ACC2

interface g1/0

description TO-ACC3

## CORE2

interface range g0/1-3,g1/0

switchport trunk encapsulation dot1q

switchport mode trunk

switchport trunk native vlan 999

no shutdown

interface g0/1

description TO-CORE1

interface g0/2

description TO-ACC1

interface g0/3

description TO-ACC2

interface g1/0

description TO-ACC3

## ACC1 / ACC2 / ACC3

interface range g0/0-1

switchport trunk encapsulation dot1q

switchport mode trunk

switchport trunk native vlan 999

no shutdown

interface g0/0

description TO-CORE1

interface g0/1

description TO-CORE2


# STEP 4 — Access Port Configuration

## ACC1

interface range g0/2-3

switchport mode access

switchport access vlan 10

spanning-tree portfast

no shutdown

interface g0/2

description TO-USER1

interface g0/3

description TO-USER2

## ACC2

interface g0/2

description TO-SERVER1

switchport mode access

switchport access vlan 20

spanning-tree portfast

no shutdown

## ACC3
 
interface g0/2

description TO-MGMT1

switchport mode access

switchport access vlan 40

spanning-tree portfast

no shutdown


# STEP 5 — STP Root Placement

## CORE1

spanning-tree vlan 10 root primary

spanning-tree vlan 20 root secondary

## CORE2

spanning-tree vlan 10 root secondary

spanning-tree vlan 20 root primary


# STEP 6 — Inter-VLAN Routing Configuration

## CORE1

ip routing

interface vlan 10

description USERS-GW

ip address 10.10.10.1 255.255.255.0

no shutdown

interface vlan 20

description SERVERS-GW

ip address 10.20.20.1 255.255.255.0

no shutdown

interface vlan 40

description MGMT-GW

ip address 10.40.40.1 255.255.255.0

no shutdown

## CORE2

ip routing

interface vlan 10

description USERS-GW

ip address 10.10.10.2 255.255.255.0

no shutdown

interface vlan 20

description SERVERS-GW

ip address 10.20.20.2 255.255.255.0

no shutdown

interface vlan 40

description MGMT-GW

ip address 10.40.40.2 255.255.255.0

no shutdown


# STEP 7 — HSRP Configuration

## CORE1

interface vlan 10

standby 10 ip 10.10.10.254

standby 10 priority 110

standby 10 preempt

interface vlan 20

standby 20 ip 10.20.20.254

standby 20 priority 90

standby 20 preempt

interface vlan 40

standby 40 ip 10.40.40.254

standby 40 priority 110

standby 40 preempt

## CORE2

interface vlan 10

standby 10 ip 10.10.10.254

standby 10 priority 90

standby 10 preempt

interface vlan 20

standby 20 ip 10.20.20.254

standby 20 priority 110

standby 20 preempt

interface vlan 40

standby 40 ip 10.40.40.254

standby 40 priority 90

standby 40 preempt


# STEP 8 — End Host Static IP Configuration

## USER1

ip addr add 10.10.10.11/24 dev eth0

ip route add default via 10.10.10.254

## USER2

ip addr add 10.10.10.12/24 dev eth0

ip route add default via 10.10.10.254

## SERVER1

ip addr add 10.20.20.11/24 dev eth0

ip route add default via 10.20.20.254

## MGMT1

ip addr add 10.40.40.11/24 dev eth0

ip route add default via 10.40.40.254











