# Phase 02 — Enterprise Services Verification

This document summarizes the validation procedures and successful verification results for the Enterprise Services phase.

---

# 1. Internet Connectivity Verification

## Objective

Verify that internal devices can successfully reach external Internet resources through EDGE1 NAT/PAT.

---

## MGMT-SRV1 Verification

### Verify External IP Reachability

```bash
ping 8.8.8.8
```

Expected Result:

```bash
64 bytes from 8.8.8.8
```

---

### Verify DNS Resolution

```bash
ping google.com
```

Expected Result:

```bash
PING google.com
```

Successful hostname resolution confirmed:

- DNS functionality
- Internet connectivity
- Proper default routing

---

# 2. DHCP Service Verification

## Objective

Verify centralized DHCP operation using dnsmasq on MGMT-SRV1.

---

## Client Verification

### Request DHCP Address

```bash
ip dhcp
```

### Verify Assigned Address

```bash
show ip
```

Expected Results:

- Dynamic IP address assigned
- Correct subnet mask
- Correct default gateway
- DNS server assignment

---

## DHCP Lease Verification on MGMT-SRV1

```bash
cat /var/lib/misc/dnsmasq.leases
```

Expected Result:

Multiple active DHCP lease entries.

---

# 3. NAT/PAT Verification

## Objective

Verify successful address translation through EDGE1.

---

## Client Internet Reachability

```bash
ping 8.8.8.8
```

Expected Result:

Successful ICMP replies from external Internet destinations.

---

## DNS-Based Connectivity

```bash
ping google.com
```

Expected Result:

Successful hostname resolution and Internet reachability.

---

# 4. SSH Remote Management Verification

## Objective

Verify secure SSH-only remote management access.

---

## SSH Status Verification

```bash
show ip ssh
```

Expected Result:

```bash
SSH Enabled - version 2.0
```

---

## VTY Configuration Verification

```bash
show run | section line vty
```

Expected Result:

```bash
login local
transport input ssh
```

---

## SSH Login Verification

```bash
ssh admin@10.x.x.x
```

Expected Result:

Successful encrypted SSH login.

---

# 5. SNMP Monitoring Verification

## Objective

Verify SNMPv2c monitoring functionality from MGMT-SRV1.

---

## Verify SNMP Polling

```bash
snmpwalk -v2c -c NMS-RO 10.255.255.1 1.3.6.1.2.1.1
```

Expected Results:

Successful retrieval of:

- System description
- Hostname
- Contact information
- Device location
- System uptime

---

## Example Successful Output

```bash
iso.3.6.1.2.1.1.5.0 = STRING: "EDGE1.lab.local"
```

---

# 6. Syslog Verification

## Objective

Verify centralized Syslog message delivery to MGMT-SRV1.

---

## Generate Log Event

Example interface shutdown/no shutdown:

```bash
interface g0/1
 shutdown
 no shutdown
```

---

## Verify Syslog Reception

```bash
sudo tail -f /var/log/syslog
```

Expected Result:

Cisco log messages successfully received from network devices.

---

# 7. NTP Synchronization Verification

## Objective

Verify centralized time synchronization using CORE1 as the NTP master.

---

## Verify NTP Associations

```bash
show ntp associations
```

Expected Result:

Devices successfully synchronized with CORE1.

---

## Verify System Clock

```bash
show clock
```

Expected Result:

Consistent timestamps across devices.

---

# 8. Management VLAN Verification

## Objective

Verify VLAN40 management segmentation and remote management reachability.

---

## Verify VLAN Status

```bash
show vlan brief
```

Expected Result:

```bash
VLAN 40 active
```

---

## Verify SVI Status

```bash
show ip interface brief
```

Expected Result:

```bash
Vlan40 up/up
```

---

## Verify Management Reachability

```bash
ping 10.40.40.x
```

Expected Result:

Successful reachability between management devices.

---

# 9. Trunk Verification

## Objective

Verify proper VLAN forwarding across trunk links.

---

## Verify Trunk Status

```bash
show interfaces trunk
```

Expected Results:

- Trunking enabled
- Native VLAN 999
- VLANs allowed and active
- VLAN forwarding operational

---

# 10. Routing Verification

## Objective

Verify Layer 3 routed connectivity between CORE and EDGE devices.

---

## Verify Routing Table

```bash
show ip route
```

Expected Result:

Presence of:

- Connected routes
- Static routes
- Default route

---

## Verify Point-to-Point Reachability

### CORE1 to EDGE1

```bash
ping 10.255.255.1
```

### CORE2 to EDGE1

```bash
ping 10.255.255.5
```

Expected Result:

Successful ICMP replies.

---

# 11. Enterprise Services Validation Summary

The following services were successfully validated:

| Service | Status |
|---|---|
| Internet Connectivity | Successful |
| DHCP Services | Successful |
| DNS Resolution | Successful |
| NAT/PAT | Successful |
| SSH Management | Successful |
| SNMP Monitoring | Successful |
| Syslog Logging | Successful |
| NTP Synchronization | Successful |
| Management VLAN | Successful |
| Routed Core-Edge Design | Successful |

---

# 12. Final Operational Outcome

Phase 02 successfully implemented a centralized enterprise management architecture including:

- Linux-based management services
- Secure remote administration
- Centralized monitoring
- Centralized logging
- Centralized DHCP services
- Management VLAN segmentation
- Routed enterprise core-edge connectivity

This phase simulated realistic enterprise infrastructure operations and troubleshooting workflows.

---
