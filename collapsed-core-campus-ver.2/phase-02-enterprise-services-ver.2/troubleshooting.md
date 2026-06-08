# Phase 02 — Enterprise Services Ver.2 Troubleshooting

## 1. Purpose

This document records the troubleshooting notes, symptoms, causes, checks, and fixes observed during **Phase 02 — Enterprise Services Ver.2** of the Collapsed Core Campus lab.

Phase 02 introduced centralized enterprise services through **MGMT-SRV1**.

MGMT-SRV1 is not a simple standalone server.
MGMT-SRV1 functions as the **Enterprise Management Server** for the campus network.

Services provided by MGMT-SRV1:

* DHCP Server
* DNS Resolver
* NTP Server
* Syslog Server
* SNMP Monitoring Station
* Config Backup Server

---

## 2. Troubleshooting Method Used in This Phase

The troubleshooting approach used in this phase was:

```text
1. Verify Phase 01 baseline first
2. Verify device IP addressing
3. Verify routing and reachability
4. Verify service process status
5. Verify listening ports
6. Verify client-side behavior
7. Verify logs
8. Save only after successful validation
```

The most important rule was:

```text
Do not start new Enterprise Services configuration until the Phase 01 foundation is confirmed stable.
```

---

## 3. Phase 01 Baseline Validation Before Phase 02

### Symptom

Before starting Phase 02, there was a concern that Phase 01 data may have been lost after CML restart or lab reload.

### Checks

Cisco devices were checked first:

```cisco
show ip interface brief
show vlan brief
show interfaces trunk
show spanning-tree summary
show standby brief
show ip route
show running-config | include ip route
```

Linux nodes were checked with:

```bash
ip -br addr
ip route
ping -c 4 <target-ip>
```

### Result

The following were confirmed:

```text
CORE1 / CORE2 SVI configuration retained
HSRP retained
STP root alignment retained
Trunks retained
EDGE1 routed links retained
ISP1 / ISP2 routed links retained
Static routing retained
SERVER1 static IP retained
MGMT-SRV1 static IP retained
INTERNET-SRV reachability retained
```

### Lesson

Always validate the previous phase before adding new services.

---

## 4. ISP2 Static Route Next-Hop Error

### Symptom

ISP2 had static routes pointing to the wrong next-hop.

Incorrect next-hop:

```text
100.64.1.2
```

This is EDGE1's ISP1-facing address, not the ISP2-facing address.

### Correct Next-Hop

ISP2 should use EDGE1's ISP2-facing address:

```text
100.64.2.2
```

### Fix

On ISP2:

```cisco
conf t
no ip route 172.16.0.0 255.255.255.252 100.64.1.2
no ip route 172.16.0.4 255.255.255.252 100.64.1.2
ip route 172.16.0.0 255.255.255.252 100.64.2.2
ip route 172.16.0.4 255.255.255.252 100.64.2.2
end
write memory
```

### Verification

```cisco
show running-config | include ip route
show ip route
ping 172.16.0.5
ping 172.16.0.1
```

### Result

ISP2 routed link reachability was restored.

---

## 5. External Connector DHCP Address Verification

### Symptom

External Connector interfaces on ISP1 and ISP2 needed to receive DHCP addresses before Internet reachability could work.

### Checks

On ISP1 / ISP2:

```cisco
show ip interface brief
show dhcp lease
show ip route
```

### Expected Result

ISP1 and ISP2 should receive addresses from the CML External Connector network:

```text
192.168.255.x/24
Default gateway: 192.168.255.1
```

### Verification

```cisco
ping 192.168.255.1
ping 8.8.8.8 source GigabitEthernet0/2
ping 1.1.1.1 source GigabitEthernet0/2
```

### Result

ISP1 and ISP2 successfully reached real Internet IP addresses through External Connector.

---

## 6. NAT/PAT Required for Internal Internet Access

### Symptom

ISP1 and ISP2 could reach the Internet, but MGMT-SRV1 initially could not reach external IPs.

### Cause

MGMT-SRV1 used an internal address:

```text
10.10.40.10
```

This private/internal address needed NAT/PAT before reaching the external network.

### Design

Two layers of NAT were used:

```text
MGMT-SRV1 10.10.40.10
→ EDGE1 NAT/PAT
→ 100.64.1.2
→ ISP1 Lab NAT
→ 192.168.255.79
→ Internet
```

### Verification

MGMT-SRV1:

```bash
ping -c 4 8.8.8.8
ping -c 4 1.1.1.1
```

EDGE1:

```cisco
show ip nat translations
show ip nat statistics
```

ISP1:

```cisco
show ip nat translations
show ip nat statistics
```

### Result

Internal Internet reachability was restored after EDGE1 and ISP1 NAT/PAT were configured.

---

## 7. EDGE1 Dual PAT Statement Behavior

### Symptom

When configuring both of the following on EDGE1:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/1 overload
```

only one statement remained active at a time.

When G0/0 was configured, G0/1 disappeared.
When G0/1 was configured, G0/0 disappeared.

### Cause

With this simple dynamic PAT style, IOSv kept only one dynamic PAT mapping for the same ACL.

### Decision

For Phase 02, keep NAT/PAT on the primary ISP1 path:

```cisco
ip nat inside source list ENTERPRISE-NAT interface GigabitEthernet0/0 overload
```

### Reason

EDGE1's default route prefers ISP1:

```text
0.0.0.0/0 via 100.64.1.1
```

### Future Work

Dual-WAN NAT failover should be handled later using route-map based NAT or policy-based NAT.

---

## 8. DNS Resolver vs External DNS Query

### Symptom

MGMT-SRV1 could resolve `google.com`.

### Clarification

At first, this meant MGMT-SRV1 could use external DNS servers:

```text
8.8.8.8
1.1.1.1
```

This was not yet the same as MGMT-SRV1 acting as the internal DNS Resolver.

### Final Design

MGMT-SRV1 later became the internal DNS resolver through dnsmasq:

```text
Client → 10.10.40.10 → dnsmasq → upstream DNS
```

### Verification

```bash
dig @10.10.40.10 google.com A
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

---

## 9. dnsmasq Port 53 Conflict

### Symptom

dnsmasq failed to start.

Error:

```text
dnsmasq: failed to create listening socket for port 53: Address already in use
```

### Cause

`systemd-resolved` was already listening on local DNS stub addresses:

```text
127.0.0.53:53
127.0.0.54:53
```

### Check

```bash
sudo ss -lntup | grep ':53'
sudo ss -lnup | grep ':53'
```

### Fix

Do not disable `systemd-resolved`.

Instead, configure dnsmasq to listen on MGMT-SRV1's management IP:

```conf
interface=ens2
listen-address=10.10.40.10
bind-interfaces
```

### Verification

```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
systemctl status dnsmasq --no-pager
sudo ss -lnup | grep ':53'
```

Expected:

```text
10.10.40.10:53 dnsmasq
127.0.0.53:53 systemd-resolved
```

### Result

dnsmasq and systemd-resolved coexisted successfully.

---

## 10. dnsmasq `Failed to set DNS config` Warning

### Symptom

dnsmasq service showed warning messages such as:

```text
warning: ignoring resolv-set
Failed to set DNS configuration
```

### Cause

dnsmasq attempted to interact with the local resolver configuration, but that integration was not required for this lab.

### Impact

Not critical.

### Reason

The lab objective was not to make MGMT-SRV1 itself use dnsmasq as its local resolver.
The objective was to make dnsmasq listen on:

```text
10.10.40.10:53
```

and respond to internal clients.

### Validation

```bash
dig @10.10.40.10 google.com A
dig @10.10.40.10 server1.campus.lab A
```

If these queries work, dnsmasq is functioning for the lab.

---

## 11. dnsmasq Configuration File Conflict

### Symptom

dnsmasq failed with errors after editing the configuration.

Error:

```text
illegal repeated keyword
```

### Cause

An extra temporary or backup file was left inside:

```text
/etc/dnsmasq.d/
```

Observed file:

```text
campus-dn
```

dnsmasq read this extra file and caused configuration conflict.

### Check

```bash
ls -l /etc/dnsmasq.d/
```

### Fix

Move backup or temporary files out of `/etc/dnsmasq.d/`.

```bash
sudo mkdir -p /root/dnsmasq-backups
sudo mv /etc/dnsmasq.d/campus-dn /root/dnsmasq-backups/
```

Expected valid directory:

```text
/etc/dnsmasq.d/
├── README
└── campus-dns.conf
```

### Verification

```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
systemctl status dnsmasq --no-pager
```

### Lesson

Do not keep backup files inside `/etc/dnsmasq.d/`.

Good backup locations:

```text
/root/dnsmasq-backups/
/home/cisco/backups/
```

---

## 12. `.local` Domain Issue

### Symptom

`campus.local` generated warnings and inconsistent client behavior.

Example warning:

```text
.local is reserved for Multicast DNS
```

### Cause

`.local` is commonly used by mDNS and can cause resolver confusion on Linux clients.

### Fix

The lab domain was changed from:

```text
campus.local
```

to:

```text
campus.lab
```

### Verification

```bash
grep 'campus.local' /etc/dnsmasq.d/campus-dns.conf
```

Expected:

```text
No output
```

Then verify:

```bash
dig @10.10.40.10 server1.campus.lab A
dig @10.10.40.10 mgmt-srv1.campus.lab A
dig @10.10.40.10 internet-srv.campus.lab A
```

### Result

The lab standardized on:

```text
campus.lab
```

---

## 13. Alpine Desktop / BusyBox DNS Behavior

### Symptom

CML Desktop nodes based on Alpine Linux could show inconsistent DNS behavior.

Example:

```text
ping: bad address 'server1.campus.lab'
```

But `nslookup` returned the correct A record.

### Cause

Alpine Desktop nodes use BusyBox-style tools such as:

```text
udhcpc
nslookup
ping
```

BusyBox tools may show confusing output when additional query types or resolver behavior are involved.

### Validation Method

Use explicit DNS server queries:

```bash
nslookup server1.campus.lab 10.10.40.10
nslookup mgmt-srv1.campus.lab 10.10.40.10
nslookup internet-srv.campus.lab 10.10.40.10
```

Expected A records:

```text
server1.campus.lab      → 10.10.70.10
mgmt-srv1.campus.lab    → 10.10.40.10
internet-srv.campus.lab → 203.0.113.10
```

For reachability, use IP ping:

```bash
ping -c 4 10.10.70.10
ping -c 4 10.10.40.10
ping -c 4 203.0.113.10
```

### Lesson

If `nslookup` returns the correct A record from `10.10.40.10`, DNS service is valid even if BusyBox `ping` behaves unexpectedly.

---

## 14. Alpine Desktop `/etc/resolv.conf` Missing

### Symptom

After DHCP renewal on an Alpine Desktop node:

```text
cat: can't open '/etc/resolv.conf': No such file or directory
```

### Cause

The Desktop/Alpine node did not automatically create or preserve `/etc/resolv.conf`.

### Impact

This does not mean DHCP reservation failed.

### DHCP Reservation Validation

```text
lease of 10.10.10.136 obtained from 10.10.40.10
```

### Optional Manual Fix

```bash
sudo sh -c 'echo "search campus.lab" > /etc/resolv.conf'
sudo sh -c 'echo "nameserver 10.10.40.10" >> /etc/resolv.conf'
cat /etc/resolv.conf
```

### DNS Direct Test

```bash
nslookup google.com 10.10.40.10
nslookup server1.campus.lab 10.10.40.10
```

---

## 15. DHCP Reservation Target Selection

### Symptom

Initial plan considered SERVER1 as a DHCP reservation target.

### Observation

SERVER1 had static IP:

```text
10.10.70.10/24
```

and was not present as a DHCP lease.

### Decision

Do not use SERVER1 as the DHCP reservation example.

Instead, use an actual DHCP client from the lease file:

```text
MAC: 52:54:00:71:87:ac
IP : 10.10.10.136
Name: exec
```

### dnsmasq Entry

```conf
dhcp-host=52:54:00:71:87:ac,10.10.10.136,exec,12h
```

### Verification

```text
lease of 10.10.10.136 obtained from 10.10.40.10
```

### Lesson

Use a real DHCP client for DHCP reservation validation.

---

## 16. SERVER1 Is Not the DNS/DHCP Server

### Symptom

Commands were accidentally run on SERVER1 instead of MGMT-SRV1:

```bash
cat /etc/dnsmasq.d/campus-dns.conf
systemctl status dnsmasq
```

Result:

```text
No such file or directory
Unit dnsmasq.service could not be found
```

### Cause

SERVER1 is not the DNS/DHCP server.

### Correct Role

SERVER1:

```text
Static server node
IP: 10.10.70.10/24
DNS record: server1.campus.lab
```

MGMT-SRV1:

```text
Enterprise Management Server
DHCP/DNS/NTP/Syslog/SNMP/Config Backup
IP: 10.10.40.10/24
```

### Lesson

Always confirm which console is active before running service commands.

Useful command:

```bash
ip -br addr
hostname
```

---

## 17. NTP Access Switch Ping Failure

### Symptom

ACC1, ACC2, ACC3, and SRV-ACC1 could not ping MGMT-SRV1.

Error:

```text
% Unrecognized host or address, or protocol not running.
```

### Cause

The access switches were not fully prepared as management IP devices at this point.

Possible causes:

```text
No management SVI IP
No default gateway
Management SVI down
IP protocol not ready for management
```

### Decision

Do not force NTP/Syslog/SNMP on access switches in this section.

### Scope

NTP was verified on:

```text
CORE1
CORE2
EDGE1
```

### Lesson

Access switch management should be completed after management SVI/default-gateway cleanup.

---

## 18. chrony Package Installation Console Issue

### Symptom

During `apt install chrony`, the console appeared stuck.

### Cause

Ubuntu `needrestart` output and the CML console display became visually confusing.

### Fix

Use the following package install style:

```bash
sudo NEEDRESTART_MODE=a apt install -y chrony
```

If the console locks visually, try:

```text
Ctrl + Q
```

### Verification

```bash
which chronyd
chronyc --version
systemctl status chrony --no-pager
```

Expected:

```text
/usr/sbin/chronyd
chrony active (running)
```

---

## 19. chrony `Detected falseticker` Message

### Symptom

chrony logs showed messages such as:

```text
Detected falseticker
System clock wrong by ...
```

### Cause

Virtualized lab environments can temporarily produce time irregularities during boot or NTP source selection.

### Impact

Not critical if the current service state is active and tracking is valid.

### Verification

```bash
systemctl status chrony --no-pager
chronyc sources -v
chronyc tracking
```

Important valid state:

```text
Leap status: Normal
chrony active (running)
```

### Lesson

Treat past chrony log warnings as non-critical if current tracking state is valid.

---

## 20. Syslog File Named by IP Instead of Hostname

### Symptom

Syslog file was created as:

```text
/var/log/campus-network/10.10.40.1.log
```

instead of:

```text
CORE1.log
```

### Cause

rsyslog used source IP for the received message.

### Impact

Not a problem.

### Validation

```bash
sudo ls -l /var/log/campus-network/
sudo tail -n 30 /var/log/campus-network/*
```

Observed messages:

```text
%SYS-6-LOGGINGHOST_STARTSTOP
%SYS-5-CONFIG_I
```

### Lesson

IP-based log filenames are acceptable and may be more reliable in small labs.

---

## 21. SSH KEX Algorithm Failure from Ubuntu 24.04

### Symptom

SSH from MGMT-SRV1 to Cisco IOSv failed.

Error:

```text
no matching key exchange method found
Their offer:
diffie-hellman-group-exchange-sha1,
diffie-hellman-group14-sha1,
diffie-hellman-group1-sha1
```

### Cause

Ubuntu 24.04 OpenSSH rejects older SHA1-based algorithms by default.

### Fix

Use compatibility options:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@<device-ip>
```

Examples:

```bash
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.1

ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@10.10.40.2

ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@172.16.0.1
```

### Verification

Successful prompts:

```text
CORE1#
CORE2#
EDGE1#
```

---

## 22. SSH Password Prompt Repeated

### Symptom

During SSH to EDGE1, the password prompt appeared multiple times.

### Cause

Possible causes:

```text
Wrong password entered
Password not visible while typing
Authentication retry behavior
```

### Fix

Enter the configured local admin password carefully.

If still failing, reset the password from the Cisco console:

```cisco
conf t
username admin privilege 15 secret <ADMIN_PASSWORD>
end
```

### Verification

```text
EDGE1#
```

---

## 23. EDGE1 SSH Key Missing

### Symptom

EDGE1 initially had no SSH key.

Observed:

```text
IOS Keys in SECSH format: NONE
transport input none
```

### Cause

RSA key had not been generated and VTY access was disabled.

### Fix

```cisco
conf t
ip domain-name campus.lab
username admin privilege 15 secret <ADMIN_PASSWORD>
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
end
```

### Verification

```cisco
show ip ssh
show running-config | section line vty
```

Expected:

```text
SSH Enabled - version 2.0
transport input ssh
```

---

## 24. Telnet Not Tested Directly

### Observation

Telnet was not directly tested with a `telnet` client.

### Configuration-Based Validation

Telnet is blocked because VTY is configured as:

```cisco
transport input ssh
```

### Meaning

Only SSH is accepted on VTY lines.

### Optional Test

If telnet client is available:

```bash
telnet 10.10.40.1
telnet 10.10.40.2
telnet 172.16.0.1
```

Expected:

```text
Connection refused or failed
```

### Lesson

In this lab, Telnet blocking was validated by configuration.

---

## 25. SNMP MIB Name Resolution Failure

### Symptom

This failed:

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 sysName.0
```

Error:

```text
Unknown Object Identifier
Sub-id not found: (top) -> sysName
```

### Cause

Net-SNMP on MGMT-SRV1 did not resolve the MIB name.

### Fix

Use numeric OIDs.

sysName:

```text
1.3.6.1.2.1.1.5.0
```

sysDescr:

```text
1.3.6.1.2.1.1.1.0
```

ifDescr:

```text
1.3.6.1.2.1.2.2.1.2
```

### Working Commands

```bash
snmpwalk -v2c -c NMS-RO 10.10.40.1 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 10.10.40.2 1.3.6.1.2.1.1.5.0
snmpwalk -v2c -c NMS-RO 172.16.0.1 1.3.6.1.2.1.1.5.0
```

### Result

```text
CORE1.campus.lab
CORE2.campus.lab
EDGE1.campus.lab
```

### Lesson

Numeric OIDs are reliable for lab validation when MIB names are unavailable.

---

## 26. SNMP ACL Verification Before SNMP Test

### Symptom

SNMP community was configured, but ACL needed confirmation.

### Check

```cisco
show access-lists SNMP-MGMT
```

Expected:

```text
Standard IP access list SNMP-MGMT
    10 permit 10.10.40.10
    20 deny any
```

### Lesson

Always confirm SNMP ACL exists before running SNMP queries.

---

## 27. MGMT-SRV1 Console XOFF / Frozen Terminal

### Symptom

After package installation, the console did not respond to keys.

### Cause

The terminal likely entered XOFF state, often caused by `Ctrl + S`.

### Fix

Press:

```text
Ctrl + Q
```

### Result

The console resumed output/input.

### Lesson

If Linux console appears frozen, try `Ctrl + Q` before rebooting.

---

## 28. MGMT-SRV1 Console Empty After Reboot

### Symptom

After restarting MGMT-SRV1, the console screen was blank or showed only a cursor.

### Cause

The Ubuntu console may be booting, or the TTY prompt may not be drawn.

### Fix

Try:

```text
Enter
Wait 30–60 seconds
Close and reopen console
Type username even if prompt is not visible
```

If needed:

```text
MGMT-SRV1 Stop
Wait 10 seconds
Start
Wait 1–2 minutes
Open Console
```

### Lesson

Blank console does not always mean data loss.

---

## 29. Ubuntu `needrestart` Output Looks Like a Hang

### Symptom

After `apt install`, messages such as the following appeared and confused the console:

```text
outdated hypervisor (qemu) binaries on this host
```

### Cause

Ubuntu's `needrestart` tool prints post-install service/hypervisor notices.

### Impact

Usually not a failure.

### Prevention

Use:

```bash
sudo NEEDRESTART_MODE=a apt install -y <package>
```

Example:

```bash
sudo NEEDRESTART_MODE=a apt install -y snmp
```

### Lesson

Use `NEEDRESTART_MODE=a` during package installation in this lab.

---

## 30. Config Backup Using SSH and Terminal Length

### Issue

When capturing `show running-config` over SSH, output paging can interrupt the backup.

### Fix

Send `terminal length 0` before `show running-config`.

### Working Pattern

```bash
printf "terminal length 0\nshow running-config\nexit\n" | \
ssh -tt -oKexAlgorithms=+diffie-hellman-group14-sha1 \
-oHostKeyAlgorithms=+ssh-rsa \
-oPubkeyAcceptedAlgorithms=+ssh-rsa \
admin@<device-ip> > <backup-file>
```

### Verification

```bash
grep -H "^hostname" /srv/config-backups/*/*running-config-$(date +%F).txt
```

Expected:

```text
hostname CORE1
hostname CORE2
hostname EDGE1
```

---

## 31. Difference Between Cisco Config Backup and MGMT-SRV1 Backup

### Question

Why back up MGMT-SRV1 after doing Config Backup?

### Answer

They are different backup targets.

Cisco Config Backup:

```text
MGMT-SRV1 backs up CORE1 / CORE2 / EDGE1 running-configs.
```

MGMT-SRV1 Enterprise Management Server Backup:

```text
Backs up MGMT-SRV1's own service configuration.
```

### MGMT-SRV1 Backup Includes

```text
/etc/dnsmasq.d/campus-dns.conf
/etc/chrony/chrony.conf
/etc/rsyslog.d/10-campus-network.conf
/srv/config-backups/
```

### Summary

```text
Section 11 = MGMT-SRV1 backs up Cisco devices
MGMT-SRV1 backup = MGMT-SRV1 backs up itself
```

Both are required.

---

## 32. tar Warning During MGMT-SRV1 Backup

### Symptom

During backup archive creation:

```text
tar: Removing leading '/' from member names
```

### Cause

tar removes the leading slash from absolute paths for safety.

### Impact

Normal and safe.

### Reason

This prevents accidental extraction directly into system root paths.

### Lesson

This warning does not indicate backup failure.

---

## 33. Final Service Health Checks

After troubleshooting or rebooting MGMT-SRV1, verify:

```bash
ip -br addr
ip route
systemctl status dnsmasq --no-pager
systemctl status chrony --no-pager
systemctl status rsyslog --no-pager
which snmpwalk
which snmpget
```

Expected:

```text
ens2 = 10.10.40.10/24
default via 10.10.40.254
dnsmasq active (running)
chrony active (running)
rsyslog active (running)
snmpwalk exists
snmpget exists
```

---

## 34. Final Troubleshooting Summary

Major issues encountered and resolved:

| Issue                        | Resolution                                    |
| ---------------------------- | --------------------------------------------- |
| ISP2 wrong route next-hop    | Corrected next-hop to 100.64.2.2              |
| MGMT-SRV1 no external access | Configured EDGE1 and ISP1 NAT/PAT             |
| EDGE1 dual PAT limitation    | Kept primary G0/0 PAT                         |
| dnsmasq port 53 conflict     | Bound dnsmasq to 10.10.40.10                  |
| dnsmasq extra file conflict  | Removed temporary file from `/etc/dnsmasq.d/` |
| `.local` domain issue        | Changed domain to `campus.lab`                |
| Alpine DNS client behavior   | Validated with explicit `nslookup`            |
| chrony console confusion     | Used service checks and `NEEDRESTART_MODE=a`  |
| SSH KEX failure              | Used OpenSSH compatibility options            |
| SNMP MIB name failure        | Used numeric OIDs                             |
| Console frozen               | Used `Ctrl + Q`                               |
| Config backup paging         | Used `terminal length 0`                      |
| MGMT-SRV1 backup tar warning | Confirmed as normal                           |

---

## 35. Completion Status

All major Phase 02 Enterprise Services troubleshooting items were resolved or documented.

Final confirmed services:

```text
DHCP
DNS Resolver
NTP
Syslog
SSH
SNMP
AAA Local
Management ACL
Config Backup
DHCP Reservation
DNS Static Records
MGMT-SRV1 Enterprise Management Server Backup
```

Phase 02 troubleshooting is complete.
