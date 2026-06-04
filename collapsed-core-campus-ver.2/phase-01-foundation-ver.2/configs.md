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
 no shutdown

interface vlan11
 ip address 10.10.11.1 255.255.255.0
 no shutdown

interface vlan20
 ip address 10.10.20.1 255.255.255.0
 no shutdown

interface vlan21
 ip address 10.10.21.1 255.255.255.0
 no shutdown

interface vlan30
 ip address 10.10.30.1 255.255.255.0
 no shutdown

interface vlan31
 ip address 10.10.31.1 255.255.255.0
 no shutdown

interface vlan40
 ip address 10.10.40.1 255.255.255.0
 no shutdown

interface vlan50
 ip address 10.10.50.1 255.255.255.0
 no shutdown

interface vlan60
 ip address 10.10.60.1 255.255.255.0
 no shutdown

interface vlan70
 ip address 10.10.70.1 255.255.255.0
 no shutdown
```

### CORE2

```cisco
ip routing

interface vlan10
 ip address 10.10.10.2 255.255.255.0
 no shutdown

interface vlan11
 ip address 10.10.11.2 255.255.255.0
 no shutdown

interface vlan20
 ip address 10.10.20.2 255.255.255.0
 no shutdown

interface vlan21
 ip address 10.10.21.2 255.255.255.0
 no shutdown

interface vlan30
 ip address 10.10.30.2 255.255.255.0
 no shutdown

interface vlan31
 ip address 10.10.31.2 255.255.255.0
 no shutdown

interface vlan40
 ip address 10.10.40.2 255.255.255.0
 no shutdown

interface vlan50
 ip address 10.10.50.2 255.255.255.0
 no shutdown

interface vlan60
 ip address 10.10.60.2 255.255.255.0
 no shutdown

interface vlan70
 ip address 10.10.70.2 255.255.255.0
 no shutdown
```

---

## SECTION 9 — HSRP Configuration

### CORE1

```cisco
interface vlan10
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt

interface vlan11
 standby 11 ip 10.10.11.254
 standby 11 priority 110
 standby 11 preempt

interface vlan20
 standby 20 ip 10.10.20.254

interface vlan21
 standby 21 ip 10.10.21.254

interface vlan30
 standby 30 ip 10.10.30.254
 standby 30 priority 110
 standby 30 preempt

interface vlan31
 standby 31 ip 10.10.31.254

interface vlan40
 standby 40 ip 10.10.40.254
 standby 40 priority 110
 standby 40 preempt

interface vlan50
 standby 50 ip 10.10.50.254

interface vlan60
 standby 60 ip 10.10.60.254
 standby 60 priority 110
 standby 60 preempt

interface vlan70
 standby 70 ip 10.10.70.254
```

### CORE2

```cisco
interface vlan10
 standby 10 ip 10.10.10.254

interface vlan11
 standby 11 ip 10.10.11.254

interface vlan20
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt

interface vlan21
 standby 21 ip 10.10.21.254
 standby 21 priority 110
 standby 21 preempt

interface vlan30
 standby 30 ip 10.10.30.254

interface vlan31
 standby 31 ip 10.10.31.254
 standby 31 priority 110
 standby 31 preempt

interface vlan40
 standby 40 ip 10.10.40.254

interface vlan50
 standby 50 ip 10.10.50.254
 standby 50 priority 110
 standby 50 preempt

interface vlan60
 standby 60 ip 10.10.60.254

interface vlan70
 standby 70 ip 10.10.70.254
 standby 70 priority 110
 standby 70 preempt
```

---

## SECTION 10 — Routed Links

### EDGE1

```cisco
interface g0/0
 description TO-ISP1
 ip address 100.64.1.2 255.255.255.252
 no shutdown

interface g0/1
 description TO-ISP2
 ip address 100.64.2.2 255.255.255.252
 no shutdown

interface g0/2
 description TO-CORE1
 ip address 172.16.0.1 255.255.255.252
 no shutdown

interface g0/3
 description TO-CORE2
 ip address 172.16.0.5 255.255.255.252
 no shutdown
```

### CORE1

```cisco
interface g0/0
 description TO-EDGE1
 no switchport
 ip address 172.16.0.2 255.255.255.252
 no shutdown
```

### CORE2

```cisco
interface g0/0
 description TO-EDGE1
 no switchport
 ip address 172.16.0.6 255.255.255.252
 no shutdown
```

### ISP1

```cisco
interface g0/1
 description TO-EDGE1
 ip address 100.64.1.1 255.255.255.252
 no shutdown

interface g0/0
 description TO-INTERNET-SW
 ip address 203.0.113.1 255.255.255.0
 no shutdown
```

### ISP2

```cisco
interface g0/1
 description TO-EDGE1
 ip address 100.64.2.1 255.255.255.252
 no shutdown

interface g0/0
 description TO-INTERNET-SW
 ip address 203.0.113.2 255.255.255.0
 no shutdown
```

---

## SECTION 11 — Static Routing

### CORE1

```cisco
ip route 0.0.0.0 0.0.0.0 172.16.0.1
```

### CORE2

```cisco
ip route 0.0.0.0 0.0.0.0 172.16.0.5
```

### EDGE1

```cisco
ip route 10.10.0.0 255.255.0.0 172.16.0.2
ip route 10.10.0.0 255.255.0.0 172.16.0.6 10

ip route 0.0.0.0 0.0.0.0 100.64.1.1
ip route 0.0.0.0 0.0.0.0 100.64.2.1 10
```

### ISP1

```cisco
ip route 10.10.0.0 255.255.0.0 100.64.1.2

ip route 172.16.0.0 255.255.255.252 100.64.1.2
ip route 172.16.0.4 255.255.255.252 100.64.1.2
```

### ISP2

```cisco
ip route 10.10.0.0 255.255.0.0 100.64.2.2

ip route 172.16.0.0 255.255.255.252 100.64.2.2
ip route 172.16.0.4 255.255.255.252 100.64.2.2
```

---

## SECTION 12 — Temporary Linux Desktop Endpoint Addressing

These settings were used for PHASE 01 verification only.

They are not persistently saved because endpoint DHCP will be introduced in PHASE 02.

### EXEC

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip route add default via 10.10.10.254
```

### BIZ-ADMIN

```bash
sudo ip addr add 10.10.11.10/24 dev eth0
sudo ip route add default via 10.10.11.254
```

### SALES

```bash
sudo ip addr add 10.10.20.10/24 dev eth0
sudo ip route add default via 10.10.20.254
```

### MKT

```bash
sudo ip addr add 10.10.21.10/24 dev eth0
sudo ip route add default via 10.10.21.254
```

### IT-OPS

```bash
sudo ip addr add 10.10.30.10/24 dev eth0
sudo ip route add default via 10.10.30.254
```

### IT-SEC

```bash
sudo ip addr add 10.10.31.10/24 dev eth0
sudo ip route add default via 10.10.31.254
```

### PRINTER1

```bash
sudo ip addr add 10.10.60.10/24 dev eth0
sudo ip route add default via 10.10.60.254
```

---

## SECTION 13 — SERVER1 Persistent IP/Gateway Configuration

SERVER1 uses Ubuntu Netplan.

File:

```text
/etc/netplan/50-cloud-init.yaml
```

Final configuration:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.70.10/24
      routes:
        - to: default
          via: 10.10.70.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

Verification:

```bash
ip addr show ens2
ip route
ping 10.10.70.254
ping 10.10.40.10
ping 203.0.113.10
```

---

## SECTION 14 — MGMT-SRV1 Persistent IP/Gateway Configuration

MGMT-SRV1 uses Ubuntu Netplan.

File:

```text
/etc/netplan/50-cloud-init.yaml
```

Final configuration:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.40.10/24
      routes:
        - to: default
          via: 10.10.40.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

Verification:

```bash
ip addr show ens2
ip route
ping 10.10.40.254
ping 10.10.70.10
ping 203.0.113.10
```

---

## SECTION 15 — INTERNET-SRV Persistent IP/Gateway Configuration

INTERNET-SRV uses Tiny Core Linux.

Runtime configuration:

```bash
sudo ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
sudo route add default gw 203.0.113.1
```

Persistent startup file:

```text
/opt/bootlocal.sh
```

The following lines were added to the end of `/opt/bootlocal.sh`:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

Tiny Core Linux persistence save:

```bash
filetool.sh -b
```

Verification:

```bash
cat /opt/bootlocal.sh
ifconfig eth0
route -n
ping 203.0.113.1
```

---

## SECTION 16 — Internet Simulation Reachability Validation

### ISP1

```cisco
ping 203.0.113.10
```

### ISP2

```cisco
ping 203.0.113.10
```

### EDGE1

```cisco
ping 203.0.113.10
```

### CORE1

```cisco
ping 203.0.113.10
```

### CORE2

```cisco
ping 203.0.113.10
```

### EXEC

```bash
ping 203.0.113.10
```

Successful EXEC-to-INTERNET-SRV reachability validates the complete PHASE 01 Ver.2 path:

```text
EXEC
→ HSRP VIP
→ CORE
→ EDGE1
→ ISP
→ INTERNET-SRV
```

---

## SECTION 17 — CML Config Preservation Strategy

CML `EXTRACT CONFIGS` should be used for Cisco network devices only.

Before using `EXTRACT CONFIGS`, save the configuration on each Cisco node.

### Cisco IOSv / IOSvL2 Nodes

Apply on:

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
write memory
```

Then select only the Cisco nodes in CML and click:

```text
EXTRACT CONFIGS
```

Expected result:

```text
Cisco IOSv / IOSvL2 startup configurations are preserved in the CML lab file.
```

---

## SECTION 18 — Linux-Based Node Recovery After Export / Import

CML `EXTRACT CONFIGS` does not fully preserve Linux OS-level configuration.

The following Linux-based node settings must be documented and verified manually after CML export, download, import, or rebuild:

```text
SERVER1
MGMT-SRV1
INTERNET-SRV
```

---

### SERVER1 Recovery

Restore `/etc/netplan/50-cloud-init.yaml`:

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.70.10/24
      routes:
        - to: default
          via: 10.10.70.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

Verify:

```bash
ip addr show ens2
ip route
ping 10.10.70.254
ping 10.10.40.10
ping 203.0.113.10
```

---

### MGMT-SRV1 Recovery

Restore `/etc/netplan/50-cloud-init.yaml`:

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: no
      addresses:
        - 10.10.40.10/24
      routes:
        - to: default
          via: 10.10.40.254
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
```

Apply:

```bash
sudo netplan generate
sudo netplan apply
```

Verify:

```bash
ip addr show ens2
ip route
ping 10.10.40.254
ping 10.10.70.10
ping 203.0.113.10
```

---

### INTERNET-SRV Recovery

Restore `/opt/bootlocal.sh`:

```bash
sudo vi /opt/bootlocal.sh
```

Add the following lines to the end of the file:

```bash
ifconfig eth0 203.0.113.10 netmask 255.255.255.0 up
route add default gw 203.0.113.1
```

Save Tiny Core persistence:

```bash
filetool.sh -b
```

Verify:

```bash
cat /opt/bootlocal.sh
ifconfig eth0
route -n
ping 203.0.113.1
```

---

## SECTION 19 — Linux Desktop Endpoint Recovery

Linux Desktop endpoint IP/Gateway settings are temporary and are not preserved by CML `EXTRACT CONFIGS`.

After lab restart, export/import, or rebuild, reapply endpoint addressing only when verification is required.

### EXEC Example

```bash
sudo ip addr add 10.10.10.10/24 dev eth0
sudo ip route add default via 10.10.10.254
```

### Verification

```bash
ip addr
ip route
ping 10.10.10.254
ping 203.0.113.10
```

Other Linux Desktop endpoints can be restored using the addressing values listed in SECTION 12.

---

## SECTION 20 — DNS Resolver

DNS Resolver configuration is intentionally not included in PHASE 01.

DNS Resolver will be configured in PHASE 02 after External Connector is added for real Internet access and `apt update`.
