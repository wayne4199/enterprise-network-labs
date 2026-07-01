# Phase 07 — Troubleshooting

## 1. Purpose

This document records the major troubleshooting cases encountered during Phase 07.

Phase 07 focused on building the Cisco Catalyst SD-WAN overlay foundation:

- Controller bring-up
- WAN Edge onboarding
- Root CA installation
- Device certificate installation
- Control connections
- OMP
- TLOC
- BFD overlay tunnels
- Service VPN 10
- Multi-VPN segmentation with VPN 20

This file does not record every failed attempt.  
It records only the meaningful issues, root causes, fixes, and final verification methods.

---

## 2. Final Working Baseline

Before troubleshooting individual issues, the final successful state was:

```text
HQ-SDWAN-EDGE
- System IP: 1.1.1.101
- Site ID: 100
- Transport 1: 100.64.11.2 / public-internet
- Transport 2: 100.64.12.2 / biz-internet
- Service VPN 10: 172.16.7.1/30 toward HQ service side
- Service VPN 20: Loopback20 10.10.20.1/32

BR-SDWAN-EDGE
- System IP: 1.1.1.102
- Site ID: 200
- Transport 1: 100.64.21.2 / public-internet
- Transport 2: 100.64.22.2 / biz-internet
- Service VPN 10: 10.20.10.254/24
- Service VPN 20: Loopback20 10.20.20.1/32

Controllers
- SDWAN-MGR:   1.1.1.10 / 203.0.113.20
- SDWAN-CTRL:  1.1.1.20 / 203.0.113.21
- SDWAN-VALID: 1.1.1.30 / 203.0.113.22
```

Final successful verification:

```text
Control connections: up
OMP peers: up
TLOCs: received and resolved
BFD sessions: up
VPN 10 route exchange: pass
VPN 20 route exchange: pass
Cross-VPN isolation: pass
```

---

## 3. Troubleshooting Method Used

The following order was used repeatedly.

```text
1. Check basic interface state
2. Check transport reachability
3. Check root CA installation
4. Check chassis number and token
5. Check vManage WAN Edge certificate state
6. Check vBond / Validator serial validity
7. Check control connections
8. Check OMP peers
9. Check OMP routes
10. Check TLOC status
11. Check BFD sessions
12. Check service VPN routing table
13. Check end-to-end ping
```

Useful commands:

```cisco
show ip interface brief
show ip route
show ip route vrf 10
show ip route vrf 20

show sdwan system
show sdwan control local-properties
show sdwan control connections
show sdwan control connection-history

show sdwan omp peers
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
show sdwan bfd sessions
```

Controller-side commands:

```cisco
show omp peers
show omp routes vpn 10
show omp routes vpn 20
show omp tlocs
```

Validator-side commands:

```cisco
show orchestrator connections
show orchestrator valid-vedges
```

---

## 4. Issue 1 — vManage GUI Access Path Was Misunderstood

### Symptom

vManage GUI access was expected to work through a desktop node, but the working access path was actually through the CML external connection path.

### Working Access Path

```text
Windows Browser
  -> https://192.168.203.129:8443
  -> CML VM TCP forwarder
  -> SDWAN-MGR / vManage GUI
```

### Resolution

Use the CML VM / EXT-CONN path for vManage GUI access.

Do not depend on `desktop0` for vManage GUI access in this lab.

### Verification

```text
vManage GUI login page opens successfully
Catalyst SD-WAN dashboard loads
Configuration > Devices page loads
Configuration > Certificates page loads
```

---

## 5. Issue 2 — Temporary CA HTTP Server Must Stay Running

### Symptom

WAN Edge devices needed to download the root CA certificate, but the HTTP service must remain active during the download.

### Root Cause

The root CA certificate was hosted temporarily from the CML Cockpit terminal.  
If the terminal is closed, the HTTP server stops.

### Working Command

Run this on the CML Cockpit terminal:

```bash
cd ~/phase07-sdwan-ca
python3 -m http.server 8000 --bind 203.0.113.254
```

### Important Note

Keep this terminal open while the WAN Edge devices download the certificate.

### Expected HTTP Server Output

```text
Serving HTTP on 203.0.113.254 port 8000
"GET /WAYNE-SDWAN-LAB-CA.crt HTTP/1.1" 200 -
```

### WAN Edge Download Command

```cisco
copy http://203.0.113.254:8000/WAYNE-SDWAN-LAB-CA.crt bootflash:WAYNE-SDWAN-LAB-CA.crt
```

---

## 6. Issue 3 — Branch WAN Edge Could Not Reach the Temporary CA HTTP Server

### Symptom

From BR-SDWAN-EDGE:

```cisco
ping 203.0.113.254 source GigabitEthernet1
ping 203.0.113.254 source GigabitEthernet2
```

Result:

```text
Success rate is 0 percent
```

However, the HTTP server was still running correctly on the CML host.

### Root Cause

The CML host did not have return routes to the Branch WAN transport subnets.

Branch transport subnets:

```text
100.64.21.0/30
100.64.22.0/30
```

### Fix

On the CML host:

```bash
sudo ip route replace 100.64.21.0/30 via 203.0.113.1
sudo ip route replace 100.64.22.0/30 via 203.0.113.2
```

### Verification on CML Host

```bash
ip route get 100.64.21.2
ip route get 100.64.22.2

ping -c 2 100.64.21.2
ping -c 2 100.64.22.2
```

Expected result:

```text
100.64.21.2 via 203.0.113.1
100.64.22.2 via 203.0.113.2
0% packet loss
```

### Verification on Branch WAN Edge

```cisco
ping 203.0.113.254 source GigabitEthernet1
ping 203.0.113.254 source GigabitEthernet2
```

Expected result:

```text
Success rate is 100 percent
```

---

## 7. Issue 4 — Root CA Install Command Was Incorrect

### Symptom

The following command failed:

```cisco
root-cert-chain install bootflash:WAYNE-SDWAN-LAB-CA.crt
```

Error:

```text
% Invalid input detected
```

### Root Cause

The command was not valid in the current Catalyst SD-WAN Edge CLI mode/version.

### Fix

Use the platform software SD-WAN request command:

```cisco
request platform software sdwan root-cert-chain install bootflash:WAYNE-SDWAN-LAB-CA.crt
```

### Verification

```cisco
show sdwan control local-properties
```

Expected result:

```text
root-ca-chain-status        Installed
```

### Note

The device displayed a warning that future releases may require the certificate file to be under `/bootflash/sdwan`.  
For this lab, installation from `bootflash:` worked successfully.

---

## 8. Issue 5 — Manual Tunnel Interface Configuration Failed

### Symptom

Manual tunnel interface configuration caused commit failure.

Example attempted direction:

```cisco
interface Tunnel1
 tunnel source GigabitEthernet1
 tunnel mode sdwan
```

Commit failed with a tunnel-related error.

### Root Cause

For this Catalyst SD-WAN Edge lab, SD-WAN tunnel behavior is built from the transport physical interfaces under the `sdwan` configuration.  
Manual `Tunnel1` and `Tunnel2` configuration was unnecessary and caused conflict.

### Correct Approach

Do not manually build Tunnel interfaces.

Configure tunnel-interface under the physical transport interfaces:

```cisco
sdwan
 interface GigabitEthernet1
  tunnel-interface
   encapsulation ipsec
   color public-internet
   allow-service all
   exit
  exit

 interface GigabitEthernet2
  tunnel-interface
   encapsulation ipsec
   color biz-internet
   allow-service all
   exit
  exit
 exit
```

### Verification

```cisco
show ip interface brief
show sdwan control local-properties
```

Expected result:

```text
GigabitEthernet1 up/up
GigabitEthernet2 up/up
number-active-wan-interfaces 2
```

The device may display SD-WAN tunnel-related interfaces automatically after the physical transport interfaces are configured correctly.

---

## 9. Issue 6 — Branch WAN Edge Had Reachability but No Complete Control Plane

### Symptom

BR-SDWAN-EDGE could ping controllers:

```cisco
ping 203.0.113.20
ping 203.0.113.21
ping 203.0.113.22
```

Result:

```text
Success rate is 100 percent
```

But control connections and OMP were not fully established.

### Root Cause

IP reachability alone is not enough for SD-WAN control plane establishment.

The following also had to be correct:

```text
Root CA installed
Organization name matched
System IP configured
Site ID configured
vBond configured
Chassis number/token applied
Device certificate installed
WAN Edge serial list pushed to controllers
WAN Edge validated
Clock synchronized
```

### Fix Checklist

On WAN Edge:

```cisco
show sdwan control local-properties
```

Required values:

```text
organization-name            WAYNE-SDWAN-LAB
root-ca-chain-status         Installed
certificate-status           Installed
certificate-validity         Valid
system-ip                    1.1.1.102
site-id                      200
number-active-wan-interfaces 2
number-vbond-peers           1
```

In vManage GUI:

```text
Configuration > Certificates > WAN Edge List
1. Install Certificate
2. Send to Controllers
3. Validate Device
```

### Verification

```cisco
show sdwan control connections
show sdwan omp peers
```

Expected result:

```text
vBond up
vManage up
vSmart up
OMP peer to vSmart up
```

---

## 10. Issue 7 — Certificate Install Failed Because CSR Was Pasted Instead of Certificate

### Symptom

vManage certificate install showed:

```text
Failed to decrypt common name from certificate
```

The pasted text started with:

```text
-----BEGIN CERTIFICATE REQUEST-----
```

### Root Cause

The CSR was pasted into the certificate install window.

vManage requires the signed certificate, not the CSR.

### Correct Text

The installed certificate must start with:

```text
-----BEGIN CERTIFICATE-----
```

And end with:

```text
-----END CERTIFICATE-----
```

### Fix

Sign the CSR first:

```bash
openssl x509 -req \
  -in BR-SDWAN-EDGE.csr \
  -CA WAYNE-SDWAN-LAB-CA.crt \
  -CAkey WAYNE-SDWAN-LAB-CA.key \
  -CAcreateserial \
  -out BR-SDWAN-EDGE.crt \
  -days 3650 \
  -sha256
```

Then paste the content of:

```bash
cat BR-SDWAN-EDGE.crt
```

into vManage:

```text
Configuration > Certificates > WAN Edge List > Install Certificate
```

---

## 11. Issue 8 — Certificate Install Failed Because of Clock Mismatch

### Symptom

vManage certificate installation failed even after using a signed certificate.

Log showed clock mismatch:

```text
system clock and vmanage clock are off
```

### Root Cause

The CML host time, vManage time, and WAN Edge time were not aligned.

The certificate validity period was generated based on the CML host clock.  
If vManage or WAN Edge time is behind the certificate `notBefore` time, the certificate can fail validation.

### Verification

On WAN Edge:

```cisco
show clock
show sdwan system | include Current time
```

On SDWAN-MGR:

```cisco
show clock
```

On CML host:

```bash
date -u
```

### Fix Used in Lab

The CML host clock was aligned manually with the SD-WAN lab clock.

```bash
sudo timedatectl set-ntp false
sudo date -u -s "2026-06-27 22:25:00"
date -u
```

Then the Branch WAN Edge certificate was regenerated:

```bash
mv BR-SDWAN-EDGE.crt BR-SDWAN-EDGE.crt.bad-time

openssl x509 -req \
  -in BR-SDWAN-EDGE.csr \
  -CA WAYNE-SDWAN-LAB-CA.crt \
  -CAkey WAYNE-SDWAN-LAB-CA.key \
  -CAcreateserial \
  -out BR-SDWAN-EDGE.crt \
  -days 3650 \
  -sha256
```

### Verification

```bash
openssl x509 -in BR-SDWAN-EDGE.crt -noout -subject -issuer -serial -dates
```

Expected result:

```text
notBefore is not in the future relative to vManage/WAN Edge
notAfter is valid
issuer is WAYNE-SDWAN-LAB-ROOT-CA
```

---

## 12. Issue 9 — WAN Edge Stayed in Staging State

### Symptom

After certificate installation, the WAN Edge still showed:

```text
Validate: staging
```

### Root Cause

The certificate was installed, but the device still needed to be validated and the updated WAN Edge list needed to be pushed to the controllers.

### Fix

In vManage GUI:

```text
Configuration > Certificates > WAN Edge List
```

Actions:

```text
1. Send to Controllers
2. Actions > Validate Device
3. Refresh the page
```

### Verification on SDWAN-VALID

```cisco
show orchestrator valid-vedges
```

Expected result:

```text
validity       valid
org            WAYNE-SDWAN-LAB
```

### Final Working State

```text
HQ-SDWAN-EDGE validity: valid
BR-SDWAN-EDGE validity: valid
```

---

## 13. Issue 10 — Wrong Validator Verification Command

### Symptom

The following commands failed:

```cisco
show orchestrator valid-edges
show orchestrator vedges
```

Error:

```text
syntax error: unknown argument
```

### Root Cause

The command syntax was incorrect for this SD-WAN Validator CLI.

### Correct Command

```cisco
show orchestrator valid-vedges
```

### Related Useful Command

```cisco
show orchestrator connections
```

### Expected Output

```text
vEdge / WAN Edge entries appear with:
- chassis number
- serial number
- validity
- organization
```

---

## 14. Issue 11 — vBond Challenge Response / Serial Not Present

### Symptom

WAN Edge control connection history showed challenge response failures.

Example indicators:

```text
challenge_resp
SERONTPRES - Serial Number not present
RXTRDWN
```

### Root Cause

The Validator did not yet fully trust the WAN Edge serial/certificate state.

Common causes:

```text
WAN Edge not validated
Serial list not sent to controllers
Certificate not installed
Certificate time invalid
Chassis number/token mismatch
```

### Fix

Check vManage:

```text
Configuration > Certificates > WAN Edge List
```

Then perform:

```text
1. Confirm chassis number/token
2. Install signed certificate
3. Send to Controllers
4. Validate Device
```

Check Validator:

```cisco
show orchestrator valid-vedges
```

Expected result:

```text
validity valid
```

Check WAN Edge:

```cisco
show sdwan control connections
```

Expected result:

```text
vBond state up
vManage state up
vSmart state up
```

---

## 15. Issue 12 — OMP Advertise Commands Were Not Accepted

### Symptom

The following commands failed under SD-WAN OMP configuration:

```cisco
sdwan
 omp
  advertise connected
  advertise static
```

Error:

```text
syntax error: unknown command
```

Also, global mode did not accept:

```cisco
omp
```

### Root Cause

In this Catalyst SD-WAN Edge software version and mode, those OMP advertise commands were not valid in the attempted location.

### Resolution

Do not use those commands in this lab.

Instead, configure the service VPN routes/interfaces and verify that OMP learns and advertises the routes.

Examples:

HQ VPN 10:

```cisco
vrf definition 10
 address-family ipv4
 exit-address-family
 exit

interface GigabitEthernet3
 vrf forwarding 10
 ip address 172.16.7.1 255.255.255.252
 no shutdown
 exit

ip route vrf 10 10.10.0.0 255.255.0.0 172.16.7.2
```

Branch VPN 10:

```cisco
vrf definition 10
 address-family ipv4
 exit-address-family
 exit

interface GigabitEthernet3
 vrf forwarding 10
 ip address 10.20.10.254 255.255.255.0
 no shutdown
 exit
```

VPN 20 loopbacks:

```cisco
interface Loopback20
 vrf forwarding 20
 ip address 10.10.20.1 255.255.255.255
 no shutdown
```

```cisco
interface Loopback20
 vrf forwarding 20
 ip address 10.20.20.1 255.255.255.255
 no shutdown
```

### Verification

```cisco
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
```

Expected result:

```text
origin-proto static
origin-proto connected
status C,R
```

On WAN Edge routing table:

```cisco
show ip route vrf 10
show ip route vrf 20
```

Expected result:

```text
m routes installed via Sdwan-system-intf
```

---

## 16. Issue 13 — `addr-family ipv4` Typo

### Symptom

The following command failed:

```cisco
vrf definition 10
 addr-family ipv4
```

Error:

```text
syntax error: unknown command
```

### Root Cause

The command was typed incorrectly.

### Correct Command

```cisco
vrf definition 10
 address-family ipv4
 exit-address-family
 exit
```

### Verification

```cisco
show ip route vrf 10
```

Expected result:

```text
Routing Table: 10
```

---

## 17. Issue 14 — BFD Sessions Were Empty at First

### Symptom

The following command returned no useful BFD session output:

```cisco
show sdwan bfd sessions
```

### Root Cause

BFD overlay sessions are created after the SD-WAN control plane and TLOC resolution are working correctly.

BFD does not come up only because the WAN transport IPs can ping each other.

Required conditions:

```text
WAN Edge certificate valid
Control connections up
OMP peer up
TLOCs received and resolved
Remote TLOCs valid
```

### Verification Order

First check control:

```cisco
show sdwan control connections
show sdwan omp peers
```

Then check TLOCs on SDWAN-CTRL:

```cisco
show omp tlocs
```

Expected TLOC status:

```text
C,I,R
```

Then check BFD again:

```cisco
show sdwan bfd sessions
```

### Final Expected BFD State

On HQ:

```text
Remote System IP: 1.1.1.102
Remote Site ID: 200
State: up

public-internet to public-internet
public-internet to biz-internet
biz-internet to public-internet
biz-internet to biz-internet
```

On Branch:

```text
Remote System IP: 1.1.1.101
Remote Site ID: 100
State: up

public-internet to public-internet
public-internet to biz-internet
biz-internet to public-internet
biz-internet to biz-internet
```

---

## 18. Issue 15 — Controller Pipe Include Syntax Failed

### Symptom

The following command failed:

```cisco
show omp tlocs | include 1.1.1.101 1.1.1.102 public-internet|biz-internet|status|received
```

Error:

```text
syntax error: expecting
```

### Root Cause

The CLI did not accept the complex include expression.

### Fix

Use simpler commands.

Option 1 — show full output:

```cisco
show omp tlocs
```

Option 2 — filter one item at a time:

```cisco
show omp tlocs | include 1.1.1.101
show omp tlocs | include 1.1.1.102
show omp tlocs | include public-internet
show omp tlocs | include biz-internet
```

### Best Lab Practice

For Phase 07 documentation, use full output from:

```cisco
show omp tlocs
```

Then manually confirm the four expected TLOCs:

```text
1.1.1.101 public-internet ipsec
1.1.1.101 biz-internet ipsec
1.1.1.102 public-internet ipsec
1.1.1.102 biz-internet ipsec
```

---

## 19. Issue 16 — TLOC Status Was Rejected / Invalid / Staging

### Symptom

SDWAN-CTRL showed TLOC entries with states such as:

```text
Rej
Inv
U
Stg
```

### Root Cause

The WAN Edge was not fully validated yet, or the controller had not fully accepted the WAN Edge serial/certificate state.

### Fix

In vManage:

```text
Configuration > Certificates > WAN Edge List
```

Perform:

```text
1. Install signed WAN Edge certificate
2. Send to Controllers
3. Validate Device
4. Refresh
```

On SDWAN-VALID:

```cisco
show orchestrator valid-vedges
```

Expected result:

```text
validity valid
```

On SDWAN-CTRL:

```cisco
show omp tlocs
```

Expected final TLOC status:

```text
C,I,R
```

Meaning:

```text
C = chosen
I = installed
R = resolved
```

---

## 20. Issue 17 — VPN 10 Route Was Present but Needed Service-Side Static Route

### Symptom

HQ VPN 10 needed to advertise the HQ service-side summary route:

```text
10.10.0.0/16
```

### Root Cause

HQ-SDWAN-EDGE required a VRF 10 static route toward the HQ service-side next-hop.

### Fix

```cisco
config-transaction

ip route vrf 10 10.10.0.0 255.255.0.0 172.16.7.2

commit
end
```

### Verification on HQ

```cisco
show ip route vrf 10
```

Expected result:

```text
S 10.10.0.0/16 via 172.16.7.2
m 10.20.10.0/24 via 1.1.1.102, Sdwan-system-intf
C 172.16.7.0/30 directly connected, GigabitEthernet3
```

### Verification on SDWAN-CTRL

```cisco
show omp routes vpn 10
```

Expected result:

```text
10.10.0.0/16 received from 1.1.1.101
origin-proto static
status C,R
```

---

## 21. Issue 18 — VPN 20 Was Intentionally Isolated from VPN 10

### Symptom

VPN 20 to VPN 10 ping failed.

From HQ:

```cisco
ping vrf 20 10.20.10.254 source Loopback20
```

Result:

```text
Success rate is 0 percent
```

From Branch:

```cisco
ping vrf 20 172.16.7.1 source Loopback20
```

Result:

```text
Success rate is 0 percent
```

### Root Cause

This was expected behavior.

VPN 10 and VPN 20 are separate service VPNs.  
No route leaking was configured between them.

### Correct Interpretation

This is not a failure.

It confirms SD-WAN service VPN segmentation.

### Positive VPN 20 Test

HQ to Branch VPN 20:

```cisco
ping vrf 20 10.20.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

Branch to HQ VPN 20:

```cisco
ping vrf 20 10.10.20.1 source Loopback20
```

Expected result:

```text
Success rate is 100 percent
```

---

## 22. Issue 19 — Direct WAN Ping Does Not Prove Overlay Health

### Symptom

Transport ping succeeded:

```cisco
ping 100.64.21.2 source GigabitEthernet1
ping 100.64.22.2 source GigabitEthernet2
```

But overlay routing and BFD still had to be checked separately.

### Root Cause

Transport reachability only confirms underlay connectivity.

SD-WAN overlay requires additional control-plane and data-plane validation:

```text
DTLS/TLS control connections
OMP peer
OMP route exchange
TLOC resolution
IPsec BFD sessions
Service VPN routes
```

### Correct Verification

Use this sequence:

```cisco
show sdwan control connections
show sdwan omp peers
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
show sdwan bfd sessions
show ip route vrf 10
show ip route vrf 20
```

---

## 23. Issue 20 — vManage WAN Edge Certificate Must Match Device CN

### Symptom

Certificate install or validation failed when the certificate did not match the expected device identity.

### Root Cause

The certificate Common Name must match the device identity generated from the WAN Edge CSR.

For BR-SDWAN-EDGE, the CSR subject contained:

```text
CN = vedge-C8K-PAYG-a28-7e91-401d-9b4f-9f969c250194-1.viptela.com
```

### Verification

```bash
openssl req -in BR-SDWAN-EDGE.csr -noout -subject
openssl x509 -in BR-SDWAN-EDGE.crt -noout -subject -issuer -serial -dates
```

Expected result:

```text
Certificate subject matches the CSR subject
Issuer is WAYNE-SDWAN-LAB-ROOT-CA
Certificate date is valid
```

---

## 24. Final Troubleshooting Checklist

Use this checklist if Phase 07 has to be rebuilt.

### Step 1 — Check Transport Interfaces

```cisco
show ip interface brief
```

Expected:

```text
GigabitEthernet1 up/up
GigabitEthernet2 up/up
number-active-wan-interfaces 2
```

### Step 2 — Check Underlay Reachability

```cisco
ping 203.0.113.20
ping 203.0.113.21
ping 203.0.113.22
```

Expected:

```text
Success rate is 100 percent
```

### Step 3 — Check Root CA

```cisco
show sdwan control local-properties
```

Expected:

```text
root-ca-chain-status Installed
```

### Step 4 — Check Device Certificate

```cisco
show sdwan control local-properties
```

Expected:

```text
certificate-status   Installed
certificate-validity Valid
```

### Step 5 — Check vBond Validity

On SDWAN-VALID:

```cisco
show orchestrator valid-vedges
```

Expected:

```text
validity valid
```

### Step 6 — Check Control Connections

```cisco
show sdwan control connections
```

Expected:

```text
vsmart up
vbond up
vmanage up
```

### Step 7 — Check OMP Peer

```cisco
show sdwan omp peers
```

Expected:

```text
vsmart peer up
```

On SDWAN-CTRL:

```cisco
show omp peers
```

Expected:

```text
1.1.1.101 up
1.1.1.102 up
```

### Step 8 — Check OMP Routes

```cisco
show sdwan omp routes vpn 10
show sdwan omp routes vpn 20
```

Expected:

```text
VPN 10 routes exchanged
VPN 20 routes exchanged
```

### Step 9 — Check TLOCs

On SDWAN-CTRL:

```cisco
show omp tlocs
```

Expected:

```text
1.1.1.101 public-internet ipsec C,I,R
1.1.1.101 biz-internet    ipsec C,I,R
1.1.1.102 public-internet ipsec C,I,R
1.1.1.102 biz-internet    ipsec C,I,R
```

### Step 10 — Check BFD

```cisco
show sdwan bfd sessions
```

Expected:

```text
Four BFD sessions between HQ and Branch are up
```

### Step 11 — Check Service VPN Routes

```cisco
show ip route vrf 10
show ip route vrf 20
```

Expected:

```text
Remote routes installed as m routes via Sdwan-system-intf
```

### Step 12 — Check Same-VPN Reachability

VPN 10:

```cisco
ping vrf 10 10.20.10.254
ping vrf 10 172.16.7.1
```

VPN 20:

```cisco
ping vrf 20 10.20.20.1 source Loopback20
ping vrf 20 10.10.20.1 source Loopback20
```

Expected:

```text
Success rate is 100 percent
```

### Step 13 — Check Cross-VPN Isolation

```cisco
ping vrf 20 10.20.10.254 source Loopback20
ping vrf 20 172.16.7.1 source Loopback20
```

Expected:

```text
Success rate is 0 percent
```

This confirms VPN segmentation.

---

## 25. Lessons Learned

### 1. SD-WAN onboarding is not only IP reachability

Even if WAN Edge can ping vManage, vSmart, and vBond, the SD-WAN control plane will not fully form unless the certificate, serial list, token, organization name, and validation state are correct.

### 2. Certificate time matters

If the CML host signs a certificate using a time that is ahead of vManage or WAN Edge, certificate installation can fail.

Always check:

```bash
date -u
```

and:

```cisco
show clock
```

before signing certificates.

### 3. Do not paste CSR into vManage certificate install

CSR:

```text
-----BEGIN CERTIFICATE REQUEST-----
```

is not valid for certificate installation.

Signed certificate:

```text
-----BEGIN CERTIFICATE-----
```

is required.

### 4. Send to Controllers and Validate Device are both important

Installing the certificate alone is not enough.

The final workflow is:

```text
Install Certificate
Send to Controllers
Validate Device
Refresh
```

### 5. TLOC state must be C,I,R

TLOC entries must be chosen, installed, and resolved before BFD and service VPN forwarding can be considered fully healthy.

### 6. BFD verifies the overlay data plane

Control connections and OMP verify the control plane.  
BFD verifies the IPsec overlay data plane between WAN Edges.

### 7. Service VPN segmentation worked correctly

VPN 10 and VPN 20 successfully exchanged routes within the same VPN.

Cross-VPN communication failed as expected because route leaking was not configured.

---

## 26. Final Result

All Phase 07 troubleshooting items were resolved.

Final confirmed working state:

```text
SDWAN-MGR GUI access: working
SDWAN-CTRL OMP control plane: working
SDWAN-VALID orchestration: working
HQ-SDWAN-EDGE onboarding: complete
BR-SDWAN-EDGE onboarding: complete
Root CA installation: complete
Device certificate installation: complete
WAN Edge validation: valid
Control connections: up
OMP peers: up
TLOCs: C,I,R
BFD sessions: up
VPN 10 communication: pass
VPN 20 communication: pass
Cross-VPN isolation: pass
```
