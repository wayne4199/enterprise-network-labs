# PHASE 06 — SD-WAN Underlay Enhancement

## EDGE1

### ACLs

```cisco
ip access-list extended GUEST-INTERNET
 permit ip 10.50.50.0 0.0.0.255 any

ip access-list extended VIDEO-INTERNET
 permit ip 10.20.20.0 0.0.0.255 any

ip access-list extended VOICE-INTERNET
 permit ip 10.30.30.0 0.0.0.255 any

ip access-list standard NAT-GUEST
 permit 10.50.50.0 0.0.0.255
```

---

### IP SLA

```cisco
ip sla 1
 icmp-echo 100.64.1.1 source-interface GigabitEthernet0/0
 frequency 5

ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 100.64.2.1 source-interface GigabitEthernet0/3
 frequency 5

ip sla schedule 2 life forever start-time now
```

---

### Tracking Objects

```cisco
track 1 ip sla 1 reachability
track 2 ip sla 2 reachability
```

---

### NAT

```cisco
ip nat inside source list 10 interface GigabitEthernet0/0 overload

ip nat inside source list NAT-GUEST interface GigabitEthernet0/3 overload
```

---

### Route Maps

#### Guest Traffic

```cisco
route-map GUEST-PBR permit 10
 match ip address GUEST-INTERNET

 set ip next-hop verify-availability 100.64.2.1 10 track 2
 set ip next-hop verify-availability 100.64.1.1 20 track 1
```

#### Voice Traffic

```cisco
route-map VOICE-PBR permit 10
 match ip address VOICE-INTERNET

 set ip next-hop verify-availability 100.64.1.1 10 track 1
```

#### Video Traffic

```cisco
route-map VIDEO-PBR permit 10
 match ip address VIDEO-INTERNET

 set ip next-hop verify-availability 100.64.1.1 10 track 1
```

---

### PBR Application

```cisco
interface GigabitEthernet0/1
 ip policy route-map VOICE-PBR

interface GigabitEthernet0/2
 ip policy route-map VIDEO-PBR

interface GigabitEthernet0/1
 ip policy route-map GUEST-PBR

interface GigabitEthernet0/2
 ip policy route-map GUEST-PBR
```

---

### WAN Interfaces

#### ISP1

```cisco
interface GigabitEthernet0/0
 description EDGE1-TO-ISP1

 ip address 100.64.1.2 255.255.255.252

 ip nat outside

 service-policy output WAN-SHAPE
```

#### CORE1

```cisco
interface GigabitEthernet0/1
 description EDGE1-TO-CORE1

 ip address 172.16.0.1 255.255.255.252

 ip nat inside

 ip policy route-map GUEST-PBR
```

#### CORE2

```cisco
interface GigabitEthernet0/2
 description EDGE1-TO-CORE2

 ip address 172.16.0.5 255.255.255.252

 ip nat inside

 ip policy route-map GUEST-PBR
```

#### ISP2

```cisco
interface GigabitEthernet0/3
 description EDGE1-TO-ISP2

 ip address 100.64.2.2 255.255.255.252

 ip nat outside
```

---

## CORE1

### VLAN Interfaces

```cisco
interface Vlan30
 ip address 10.30.30.2 255.255.255.0

 standby 30 ip 10.30.30.254
 standby 30 priority 110
 standby 30 preempt

 no shutdown
```

```cisco
interface Vlan50
 ip address 10.50.50.2 255.255.255.0

 standby 50 ip 10.50.50.254
 standby 50 priority 110
 standby 50 preempt

 ip access-group GUEST-SEGMENTATION in

 no shutdown
```

---

### Guest Segmentation Policy

```cisco
ip access-list extended GUEST-SEGMENTATION

 deny ip 10.50.50.0 0.0.0.255 10.10.10.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.20.20.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.30.30.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.40.40.0 0.0.0.255

 permit ip 10.50.50.0 0.0.0.255 any
```

---

## CORE2

### VLAN Interfaces

```cisco
interface Vlan30
 ip address 10.30.30.3 255.255.255.0

 standby 30 ip 10.30.30.254
 standby 30 priority 100
 standby 30 preempt

 no shutdown
```

```cisco
interface Vlan50
 ip address 10.50.50.3 255.255.255.0

 standby 50 ip 10.50.50.254
 standby 50 priority 100
 standby 50 preempt

 ip access-group GUEST-SEGMENTATION in

 no shutdown
```

---

### Guest Segmentation Policy

```cisco
ip access-list extended GUEST-SEGMENTATION

 deny ip 10.50.50.0 0.0.0.255 10.10.10.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.20.20.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.30.30.0 0.0.0.255

 deny ip 10.50.50.0 0.0.0.255 10.40.40.0 0.0.0.255

 permit ip 10.50.50.0 0.0.0.255 any
```

---

## ACC3

### Guest Access Port

```cisco
interface GigabitEthernet0/3

 switchport mode access

 switchport access vlan 50

 spanning-tree portfast edge
```

---

## INTERNET-SRV1

### IP Configuration

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up

route add default gw 203.0.113.1
```

### Static Routes

```bash
route add -net 100.64.1.0 netmask 255.255.255.252 gw 203.0.113.1

route add -net 100.64.2.0 netmask 255.255.255.252 gw 203.0.113.2

route add -net 10.50.50.0 netmask 255.255.255.0 gw 203.0.113.2
```

---

## ISP1

```cisco
interface GigabitEthernet0/2
 ip address 203.0.113.1 255.255.255.0
 no shutdown
```

---

## ISP2

```cisco
interface GigabitEthernet0/2
 ip address 203.0.113.2 255.255.255.0
 no shutdown
```
