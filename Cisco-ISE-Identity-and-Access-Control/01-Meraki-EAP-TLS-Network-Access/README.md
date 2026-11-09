# Cisco ISE and Meraki EAP-TLS Network Access

*WPA2-Enterprise, certificate authentication, approved-device authorization, and dynamic VLAN assignment*

This lab extended my Meraki wireless environment into certificate-based network access control. I deployed Cisco ISE on ESXi, enabled its internal certificate authority, created an EAP-TLS-only policy, and connected a Meraki MR36 SSID to ISE through RADIUS.

The final policy required two independent checks. A client needed a valid certificate created from the approved template, and its real wireless MAC address needed to belong to the correct ISE endpoint group.

**In one sentence:** I built a WPA2-Enterprise SSID where Cisco ISE authenticated Windows devices with EAP-TLS, authorized approved IT and HR endpoints into separate VLANs, and denied a valid certificate when the device MAC was not yet approved.

## What I Built

- Catalyst Layer 3 switching for VLANs 10, 20, 30, and 99
- Routed upstream access through a Cisco edge router
- Meraki MS130 switching with two MR36 access points
- Cisco ISE running as a virtual machine on an ESXi VLAN 30 port group
- ISE internal certificate authority and an EAP-TLS client template
- Certificate authentication with non-TLS fallback methods disabled
- One Meraki SSID with RADIUS VLAN override
- IT authorization to VLAN 10 and HR authorization to VLAN 20
- A negative test proving that a certificate alone did not grant access

## Logical Topology

```mermaid
flowchart TB
    classDef edge fill:#f8d7da,stroke:#b02a37,color:#111
    classDef core fill:#fff3cd,stroke:#997404,color:#111
    classDef meraki fill:#d1ecf1,stroke:#0c6f82,color:#111
    classDef ise fill:#e8dcf8,stroke:#6f42c1,color:#111
    classDef endpoint fill:#d4edda,stroke:#227a3b,color:#111

    ISP["Upstream network<br/>172.18.2.x"]:::edge --> R1["Cisco router<br/>NAT and edge routing"]:::edge
    R1 -->|"10.255.0.0/30"| SW["Catalyst 3560<br/>SVIs, DHCP and routing"]:::core
    SW -->|"Gi0/2 trunk<br/>VLANs 10, 20 and 99"| MS["Meraki MS130"]:::meraki
    SW -->|"Gi0/3 trunk<br/>VLAN 30 tagged"| ESXI["ESXi<br/>192.168.99.10"]:::core
    ESXI --> ISE["Cisco ISE<br/>ise01.lab.local<br/>192.168.30.6"]:::ise
    MS --> AP1["MR36 AP1"]:::meraki
    MS --> AP2["MR36 AP2"]:::meraki
    AP1 --> SSID["LAB-EAPTLS<br/>WPA2-Enterprise"]:::meraki
    AP2 --> SSID
    SSID --> IT["Approved IT endpoint<br/>VLAN 10"]:::endpoint
    SSID --> HR["Approved HR endpoint<br/>VLAN 20"]:::endpoint
    SSID -. "RADIUS 1812/1813" .-> ISE
```

## Addressing and VLAN Plan

| VLAN or link | Network | Gateway or address | Purpose |
|---|---|---|---|
| VLAN 10 | `192.168.10.0/24` | `192.168.10.1` | IT clients |
| VLAN 20 | `192.168.20.0/24` | `192.168.20.1` | HR clients |
| VLAN 30 | `192.168.30.0/24` | ISE `192.168.30.6` | Server network |
| VLAN 99 | `192.168.99.0/24` | ESXi `192.168.99.10` | Infrastructure management |
| Router transit | `10.255.0.0/30` | Router `.1`, Catalyst `.2` | Routed upstream path |

## Stage 1: Routing and Switching Foundation

The Catalyst VLAN database contains dedicated IT, HR, server, and management networks. VLAN 999 remains available as an unused parking VLAN.

![Catalyst VLAN database showing IT, HR, server, management, and unused VLANs](Screenshots/01-Catalyst-Switch-VLAN-Database.png)

The trunk view confirms that the Meraki and ESXi paths carry the required VLANs. The same capture also shows active DHCP bindings, including leases in VLAN 10 and VLAN 99.

![Catalyst trunks and DHCP bindings](Screenshots/02-Catalyst-Trunks-and-DHCP-Bindings.png)

The edge router used one upstream interface and one routed interface toward the Catalyst. Both operational interfaces were up when captured.

![Router IP interface status](Screenshots/03-Edge-Router-IP-Interface-Status.png)

The routing table shows a default route toward the upstream gateway, the connected `/30` transit network, and a route toward the internal `192.168.0.0/16` networks through the Catalyst.

![Router routing table with the upstream default and internal route](Screenshots/04-Edge-Router-Routing-Table.png)

NAT statistics and translations provide traffic-path evidence beyond static configuration. The router recorded active translations from an internal VLAN 10 client to external destinations.

![Router NAT statistics and active translations](Screenshots/05-Edge-Router-NAT-Statistics-and-Translations.png)

## Stage 2: Meraki Switching and Wireless Infrastructure

The Meraki MS130 port view shows the switch operating as the Layer 2 distribution point for the uplink and connected lab devices.

![Meraki MS130 trunk port configuration](Screenshots/06-Meraki-MS130-Trunk-Port-Configuration.png)

Meraki Dashboard discovered the MS130 and both MR36 access points in the Layer 2 topology.

![Meraki Dashboard Layer 2 topology](Screenshots/07-Meraki-Dashboard-Layer2-Topology.png)

Both MR36 access points were online and running the same firmware when the inventory was captured.

![Two Meraki MR36 access points online](Screenshots/08-Meraki-MR36-Access-Points-Online.png)

## Stage 3: ESXi and Cisco ISE Deployment

The ESXi management VMkernel interface remained on the management network, separate from the ISE server VLAN.

![ESXi management VMkernel interface](Screenshots/09-ESXi-Management-VMkernel-Interface.png)

I created an `ISE-VLAN30` port group and assigned VLAN ID 30. This allowed the ISE virtual machine to use the tagged server VLAN while ESXi management stayed on VLAN 99.

![ESXi VLAN 30 port group for Cisco ISE](Screenshots/10-ESXi-ISE-VLAN30-Port-Group.png)

The ISE virtual machine was provisioned with 8 vCPUs, 32 GB RAM, a 300 GB disk, and network adapters connected to the VLAN 30 port group.

![Cisco ISE virtual machine resources and network adapters](Screenshots/11-ESXi-Cisco-ISE-VM-Resources-and-NICs.png)

The installed node reported Cisco ISE 3.5 on the Application Deployment Engine platform.

![Cisco ISE version and build information](Screenshots/12-Cisco-ISE-Version-and-Build.png)

The application-status capture confirms that the database, application server, policy, certificate-authority, logging, API, and protocol services were running.

![Cisco ISE application services running](Screenshots/13-Cisco-ISE-Application-Services-Status.png)

The ISE dashboard provided the first operational view of endpoints, network devices, authentications, alarms, and node health.

![Cisco ISE dashboard summary](Screenshots/14-Cisco-ISE-Dashboard-Summary.png)

## Stage 4: Certificate Authority and EAP-TLS Identity

ISE held the system certificates required for its administrative, portal, EAP, SAML, pxGrid, and messaging roles.

![Cisco ISE system certificate inventory](Screenshots/15-Cisco-ISE-System-Certificates.png)

The dedicated `ISE-EAP-CERT` certificate was assigned to EAP authentication. This allowed wireless clients to validate the server during the TLS exchange.

![Cisco ISE EAP server certificate](Screenshots/16-Cisco-ISE-EAP-Server-Certificate.png)

I enabled the ISE internal certificate authority on the standalone node.

![Cisco ISE internal CA settings](Screenshots/17-Cisco-ISE-Internal-CA-Settings.png)

The resulting hierarchy included the root CA, node CA, endpoint subordinate CA, and OCSP responder.

![Cisco ISE internal CA certificate hierarchy](Screenshots/18-Cisco-ISE-CA-Certificate-Hierarchy.png)

The `Secure_WiFi_EAP_TLS` certificate template used RSA keys and the Client Authentication extended key usage. The template was designed for individual endpoint certificates rather than a shared credential.

![Cisco ISE EAP-TLS certificate template](Screenshots/19-Cisco-ISE-EAPTLS-Certificate-Template.png)

On Windows, the client certificate appeared in the Local Computer personal store with a valid path through the ISE endpoint subordinate CA and root hierarchy.

![Windows EAP-TLS client certificate chain](Screenshots/20-Windows-EAPTLS-Client-Certificate-Chain.png)

## Stage 5: ISE Network Device and Endpoint Authorization

The Meraki infrastructure was registered in ISE as a network device with RADIUS authentication enabled.

![Cisco ISE Meraki MR36 network-device entry](Screenshots/21-Cisco-ISE-MR36-Network-Device-Entry.png)

I created approved endpoint groups to add device-level authorization on top of certificate authentication. The captured IT group contains the authorized IT endpoint.

![Cisco ISE approved IT endpoint group](Screenshots/22-Cisco-ISE-Approved-IT-Endpoint-Group.png)

The certificate authentication profile `CAP_EAPTLS_DEVICE` allowed ISE to derive the endpoint identity from the certificate.

![Cisco ISE EAP-TLS certificate authentication profile](Screenshots/23-Cisco-ISE-EAPTLS-Certificate-Authentication-Profile.png)

The allowed-protocols policy enabled EAP-TLS and disabled password-based alternatives. This prevented the secure SSID from falling back to PEAP, MS-CHAPv2, PAP, or other weaker methods.

![Cisco ISE EAP-TLS-only allowed protocols](Screenshots/24-Cisco-ISE-EAPTLS-Only-Allowed-Protocols.png)

The `Meraki-EAPTLS-Secure` policy set matched wireless 802.1X traffic for `LAB-EAPTLS`. Its authentication rule used the certificate authentication profile and rejected failed or unknown identities.

![Cisco ISE EAP-TLS authentication policy](Screenshots/25-Cisco-ISE-EAPTLS-Authentication-Policy.png)

Authorization then checked the EAP method, certificate template, and approved endpoint group:

- `IT-EAPTLS-APPROVED` returned `PERMIT_VLAN10`
- `HR-EAPTLS-APPROVED` returned `PERMIT_VLAN20`
- The default rule returned `DenyAccess`

![Cisco ISE EAP-TLS authorization rules](Screenshots/26-Cisco-ISE-EAPTLS-Authorization-Policies.png)

The VLAN authorization profiles returned standard RADIUS tunnel attributes for VLAN 10 and VLAN 20.

![Cisco ISE VLAN 10 authorization profile](Screenshots/27-Cisco-ISE-VLAN10-Authorization-Profile.png)

![Cisco ISE VLAN 20 authorization profile](Screenshots/28-Cisco-ISE-VLAN20-Authorization-Profile.png)

## Stage 6: Meraki LAB-EAPTLS SSID

The `LAB-EAPTLS` SSID used Enterprise authentication with the configured RADIUS server.

![Meraki LAB-EAPTLS enterprise security configuration](Screenshots/29-Meraki-LAB-EAPTLS-SSID-Enterprise-Security.png)

The SSID used WPA2-Enterprise, with direct access after successful 802.1X authentication and no additional splash page.

![Meraki LAB-EAPTLS WPA2 and direct-access settings](Screenshots/30-Meraki-LAB-EAPTLS-WPA2-and-Direct-Access.png)

Meraki sent authentication to ISE on UDP 1812 and accounting on UDP 1813. The configured shared secret remained obscured in the captured image.

![Meraki LAB-EAPTLS RADIUS authentication and accounting servers](Screenshots/31-Meraki-LAB-EAPTLS-RADIUS-Servers.png)

The SSID used bridge mode with external DHCP. RADIUS VLAN override was enabled so ISE could return VLAN 10 or VLAN 20 for the same SSID.

![Meraki bridge mode and RADIUS VLAN override](Screenshots/32-Meraki-LAB-EAPTLS-Bridge-Mode-and-VLAN-Override.png)

## Stage 7: Windows EAP-TLS Supplicant Configuration

I created the Windows wireless profile manually so the authentication method could be controlled explicitly.

![Windows wireless profile setup starting point](Screenshots/33-Windows-Wireless-Profile-Setup-Start.png)

![Windows manual wireless-network selection](Screenshots/34-Windows-Manual-Wireless-Network-Selection.png)

The profile used SSID `LAB-EAPTLS`, WPA2-Enterprise, and AES encryption.

![Windows LAB-EAPTLS WPA2-Enterprise profile](Screenshots/35-Windows-LAB-EAPTLS-WPA2-Enterprise-Profile.png)

![Windows confirmation that the LAB-EAPTLS profile was created](Screenshots/36-Windows-LAB-EAPTLS-Profile-Created.png)

I selected **Microsoft: Smart Card or other certificate (EAP-TLS)** instead of PEAP.

![Windows EAP-TLS authentication method](Screenshots/37-Windows-EAPTLS-Authentication-Method.png)

Because the certificate was installed in the Local Computer store, the advanced 802.1X settings used computer authentication.

![Windows EAP-TLS computer authentication mode](Screenshots/38-Windows-EAPTLS-Computer-Authentication-Mode.png)

Windows displayed the first-connect certificate confirmation, then established the secured wireless session.

![Windows LAB-EAPTLS certificate trust prompt](Screenshots/39-Windows-LAB-EAPTLS-Certificate-Trust-Prompt.png)

![Windows LAB-EAPTLS connected and secured](Screenshots/40-Windows-LAB-EAPTLS-Connected.png)

## Stage 8: IT and HR Client Validation

The IT endpoint connected with EAP-TLS and received `192.168.10.25` with gateway `192.168.10.1`. Windows reported WPA2-Enterprise, the certificate sign-in method, and a 5 GHz association.

![IT client EAP-TLS VLAN 10 network details](Screenshots/41-IT-Client-EAPTLS-VLAN10-Network-Details.png)

The command-line capture independently confirms the `LAB-EAPTLS` profile, WPA2-Enterprise authentication, and VLAN 10 addressing.

![IT client EAP-TLS VLAN 10 command-line validation](Screenshots/42-IT-Client-EAPTLS-VLAN10-CLI-Validation.png)

The HR endpoint connected to the same SSID but received `192.168.20.21` with gateway `192.168.20.1`, proving that ISE returned a different VLAN for the approved HR endpoint.

![HR client EAP-TLS VLAN 20 command-line validation](Screenshots/43-HR-Client-EAPTLS-VLAN20-CLI-Validation.png)

## Stage 9: Negative Test and Final ISE Proof

The ISE Live Logs capture brings the policy together. It contains successful IT authorization to `PERMIT_VLAN10`, an initial HR denial, and a later HR success with `PERMIT_VLAN20`.

![Cisco ISE Live Logs showing IT and HR EAP-TLS decisions](Screenshots/44-Cisco-ISE-Live-Logs-IT-and-HR-EAPTLS-Results.png)

The HR denial was intentional and useful:

1. The HR client presented a valid certificate and reached the EAP-TLS authentication policy.
2. Its MAC address was not yet in `Approved-HR-Devices`.
3. Authorization fell through to the default `DenyAccess` result.
4. I added the real wireless MAC to the approved HR group and reconnected.
5. The client matched `HR-EAPTLS-APPROVED` and received `PERMIT_VLAN20`.

This proves that certificate authentication and endpoint authorization were separate controls. Possession of a valid certificate did not automatically grant network access.

## Validation Matrix

| Test | Evidence | Result |
|---|---|---|
| VLAN foundation | Catalyst VLAN and trunk captures | PASS |
| Routed upstream path | Router routes and active NAT translations | PASS |
| Meraki infrastructure | MS130 and two MR36 devices visible online | PASS |
| ISE deployment | VM resources, services, version, and dashboard captured | PASS |
| EAP server identity | EAP certificate assigned and internal CA hierarchy active | PASS |
| EAP-TLS-only policy | EAP-TLS enabled with password-based alternatives disabled | PASS |
| IT authorization | IT rule returned VLAN 10 and client received `192.168.10.25` | PASS |
| HR authorization | HR rule returned VLAN 20 and client received `192.168.20.21` | PASS |
| Valid certificate with unapproved MAC | Authentication reached ISE but authorization returned `DenyAccess` | PASS |
| HR approval change | Adding the HR MAC changed the result to `PERMIT_VLAN20` | PASS |

## What I Learned

- EAP-TLS removes the shared-password problem, but certificate lifecycle management becomes critical.
- Trusting the ISE server certificate is different from installing a client certificate with a private key.
- Authentication and authorization must be tested separately.
- A valid certificate can still be denied by an endpoint-group condition.
- RADIUS VLAN override allows one SSID to serve multiple departments without duplicating SSIDs.
- Randomized wireless MAC addresses can break endpoint-group matching unless they are disabled or handled deliberately.
- ISE Live Logs provide the strongest proof because they show the policy set, authentication rule, authorization rule, and returned result together.

## Troubleshooting Approach

When an EAP-TLS connection fails, I check the path in this order:

1. Confirm the client can see and associate with `LAB-EAPTLS`.
2. Confirm the Windows profile uses EAP-TLS, not PEAP.
3. Verify the client certificate, private key, template, expiry, and CA chain.
4. Check that the real wireless MAC belongs to the intended endpoint group.
5. Open ISE Live Logs and identify the exact failed stage.
6. Verify the matched policy set, authentication rule, and authorization rule.
7. Confirm that Meraki accepted the returned VLAN and that DHCP succeeded there.

## Final Result

```text
IT certificate + approved IT endpoint
-> EAP-TLS success
-> IT-EAPTLS-APPROVED
-> PERMIT_VLAN10
-> 192.168.10.25

HR certificate + unapproved endpoint
-> EAP-TLS identity accepted
-> authorization condition not met
-> DenyAccess

HR certificate + approved HR endpoint
-> EAP-TLS success
-> HR-EAPTLS-APPROVED
-> PERMIT_VLAN20
-> 192.168.20.21
```

[Back to the Cisco ISE Identity and Access Control overview](../)
