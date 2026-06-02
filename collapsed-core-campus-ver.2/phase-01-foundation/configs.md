# PHASE 01 — configs.md

## SECTION 1 — Hostname Configuration

### ISP1

```cisco
hostname ISP1
```

### ISP2

```cisco
hostname ISP2
```

### EDGE1

```cisco
hostname EDGE1
```

### CORE1

```cisco
hostname CORE1
```

### CORE2

```cisco
hostname CORE2
```

### ACC1

```cisco
hostname ACC1
```

### ACC2

```cisco
hostname ACC2
```

### ACC3

```cisco
hostname ACC3
```

### SRV-ACC1

```cisco
hostname SRV-ACC1
```

---

## SECTION 2 — Base Device Configuration

Applied to:

* ISP1
* ISP2
* EDGE1
* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1

```cisco
no logging console
no ip domain-lookup
```

---

## SECTION 3 — VLAN Creation

Applied to:

* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1

```cisco
vlan 10
 name EXEC

vlan 11
 name BIZ-ADMIN

vlan 20
 name SALES

vlan 21
 name MKT

vlan 30
 name IT-OPS

vlan 31
 name IT-SEC

vlan 40
 name MGMT

vlan 50
 name GUEST

vlan 60
 name PRINTER

vlan 70
 name SERVER

vlan 999
 name NATIVE-BLACKHOLE
```

---

## SECTION 4 — Access Port Assignment

### ACC1

```cisco
interface g0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface g0/3
 switchport mode access
 switchport access vlan 11
 spanning-tree portfast

interface g1/0
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
```

### ACC2

```cisco
interface g0/2
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

interface g0/3
 switchport mode access
 switchport access vlan 21
 spanning-tree portfast
```

### ACC3

```cisco
interface g0/2
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast

interface g0/3
 switchport mode access
 switchport access vlan 31
 spanning-tree portfast
```

### SRV-ACC1

```cisco
interface g0/1
 switchport mode access
 switchport access vlan 70
 spanning-tree portfast

interface g0/2
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
```

---

## SECTION 5 — Trunk Configuration

### CORE1

```cisco
interface range g0/1 - 3, g1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

### CORE2

```cisco
interface range g0/1 - 3, g1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

### ACC1

```cisco
interface range g0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

### ACC2

```cisco
interface range g0/0 - 1, g1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

### ACC3

```cisco
interface range g0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

### SRV-ACC1

```cisco
interface g0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,11,20,21,30,31,40,50,60,70,999
```

---

## SECTION 6 — Rapid-PVST

Applied to:

* CORE1
* CORE2
* ACC1
* ACC2
* ACC3
* SRV-ACC1

```cisco
spanning-tree mode rapid-pvst
```

---

## SECTION 7 — STP Root Alignment

### CORE1

```cisco
spanning-tree vlan 10,11,30,40,60 root primary

spanning-tree vlan 20,21,31,50,70 root secondary
```

### CORE2

```cisco
spanning-tree vlan 20,21,31,50,70 root primary

spanning-tree vlan 10,11,30,40,60 root secondary
```

---

## SECTION 8 — SVI Configuration

### CORE1

```cisco
ip routing

interface vlan10
 ip address 10.10.10.1 255.255.255.0

interface vlan11
 ip address 10.10.11.1 255.255.255.0

interface vlan20
 ip address 10.10.20.1 255.255.255.0

interface vlan21
 ip address 10.10.21.1 255.255.255.0

interface vlan30
 ip address 10.10.30.1 255.255.255.0

interface vlan31
 ip address 10.10.31.1 255.255.255.0

interface vlan40
 ip address 10.10.40.1 255.255.255.0

interface vlan50
 ip address 10.10.50.1 255.255.255.0

interface vlan60
 ip address 10.10.60.1 255.255.255.0

interface vlan70
 ip address 10.10.70.1 255.255.255.0
```

### CORE2

```cisco
ip routing

interface vlan10
 ip address 10.10.10.2 255.255.255.0

interface vlan11
 ip address 10.10.11.2 255.255.255.0

interface vlan20
 ip address 10.10.20.2 255.255.255.0

interface vlan21
 ip address 10.10.21.2 255.255.255.0

interface vlan30
 ip address 10.10.30.2 255.255.255.0

interface vlan31
 ip address 10.10.31.2 255.255.255.0

interface vlan40
 ip address 10.10.40.2 255.255.255.0

interface vlan50
 ip address 10.10.50.2 255.255.255.0

interface vlan60
 ip address 10.10.60.2 255.255.255.0

interface vlan70
 ip address 10.10.70.2 255.255.255.0
```

---

## SECTION 9 — HSRP

### CORE1 Active

```cisco
interface vlan10
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

interface vlan11
 standby 11 ip 10.10.11.254
 standby 11 priority 110
 standby 11 preempt

interface vlan30
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt

interface vlan40
 standby 40 ip 10.10.40.254
 standby 40 priority 110
 standby 40 preempt

interface vlan60
 standby 60 ip 10.10.60.254
 standby 60 priority 110
 standby 60 preempt
```

### CORE2 Active

```cisco
interface vlan20
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt

interface vlan21
 standby 21 ip 10.10.21.254
 standby 21 priority 110
 standby 21 preempt

interface vlan31
 standby 31 ip 10.10.31.254
 standby 31 priority 110
 standby 31 preempt

interface vlan50
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt

interface vlan70
 standby 70 ip 10.10.70.254
 standby 70 priority 110
 standby 70 preempt
```
