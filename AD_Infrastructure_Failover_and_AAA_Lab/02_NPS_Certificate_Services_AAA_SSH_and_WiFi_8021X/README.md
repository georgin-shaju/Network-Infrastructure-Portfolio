# AD Infrastructure Failover Lab — Part 2: NPS, Certificate Services & AAA (SSH and WPA2-Enterprise Wi-Fi)

*NPS on both DCs → switch admin SSH via RADIUS → WPA2-Enterprise Wi-Fi via RADIUS → both failure modes proven by actually taking a DC down*

This picks up where [Part 1](../01_DNS_DHCP_Failover_and_Redundancy/README.md) left off: DC01 and DC02 are healthy, AD replication is clean, and DHCP/DNS both survive a single-DC outage. This lab adds centralized authentication on top of that foundation — switch admin SSH access authenticated against AD via RADIUS, and WPA2-Enterprise Wi-Fi doing the same — using Network Policy Server (NPS) running on both domain controllers so either one can answer.

**In one sentence:** I built NPS on both DCs, configured the switch and the Aironet access point to authenticate against both, tracked down two real RADIUS configuration bugs along the way, hardened the switch's SSH host key, and proved every failure mode — switch RADIUS down, AP RADIUS down, and both the authorized and denied paths through the policy — by actually testing them rather than trusting the configuration.

A Root CA (`NOC-LAB-ROOT-CA`) was already deployed on DC01 ahead of this lab to issue the NPS server certificate PEAP relies on.

## Addressing additions

| Device | Address | Notes |
|---|---|---|
| Switch RADIUS source (Vlan10 SVI) | 192.168.10.1 | `ip radius source-interface Vlan10` |
| Aironet 1815 Mobility Express | 192.168.10.4 | Management address |
| NPS1 (DC01) | 192.168.20.2 | Priority 1 RADIUS server |
| NPS2 (DC02) | 192.168.30.2 | Priority 2 RADIUS server |

RADIUS clients and network policies don't replicate between DC01 and DC02 automatically, so everything below — every RADIUS client, every policy — was built manually on both and confirmed identical before testing failover.

## Part A — Switch SSH AAA

**AD group and RADIUS client setup.** Confirmed group membership for the test account in `NOC-Network_Admins`, then added the switch as a RADIUS client — friendly name `SW-3560CX`, address `192.168.10.1` — identically on DC01 and DC02, same shared secret on both, matching the switch's own `radius server` key.

![AD group membership for the switch-admin test account](Screenshots/Lab-02_001_AD-Group-NOC-Network-Admins-Membership.png)

![DC01 RADIUS client for the switch](Screenshots/Lab-02_002_DC01-NPS-RADIUS-Client-Switch.png)

![DC02 RADIUS client for the switch, matching DC01](Screenshots/Lab-02_003_DC02-NPS-RADIUS-Client-Switch-Parity.png)

**Building the switch-administrator policy.** The `01-NOC-Switch-Administrators` network policy conditions on Windows Group plus Client Friendly Name, so only this specific switch's requests for accounts in this specific group ever match it.

![Switch-admin policy overview and conditions](Screenshots/Lab-02_004_Switch-Admin-Policy-Overview-and-Conditions.png)

![Switch-admin policy conditions detail](Screenshots/Lab-02_005_Switch-Admin-Policy-Conditions-Detail.png)

For a device shell login like this one, the policy needs **Service-Type = Login** with no Framed-Protocol attribute — Cisco IOS reads Framed-Protocol as an instruction to start PPP on the VTY line, which isn't what an SSH admin session is.

![Switch-admin policy Standard attribute — Service-Type Login](Screenshots/Lab-02_006_Switch-Admin-Policy-Standard-Service-Type-Login.png)

Privilege level comes through as a Cisco AV-Pair — `shell:priv-lvl=15`, with a colon. I confirmed the exact syntax by running `debug radius` on the switch during a test login and watching the attribute parse cleanly and apply.

![Switch-admin policy Vendor Specific attribute — Cisco AV-Pair](Screenshots/Lab-02_007_Switch-Admin-Policy-Vendor-Specific-Cisco-AVPair.png)

**Switch-side AAA configuration.** `aaa new-model`, both `radius server` entries (NPS-DC01 and NPS-DC02) in a server group, `ip radius source-interface Vlan10`, and separate method lists for console (local-only) versus VTY (RADIUS with local fallback):

![Switch running-config — AAA, RADIUS, and SSH](Screenshots/Lab-02_008_Switch-Running-Config-AAA-RADIUS-SSH.png)

`show aaa servers` gave a clean baseline against both RADIUS servers before any test traffic:

![Switch show aaa servers baseline](Screenshots/Lab-02_009_Switch-Show-AAA-Servers-Baseline.png)

Alongside the AAA config, I generated a 2048-bit RSA host key for SSH:

![Switch SSH RSA key regenerated to 2048-bit](Screenshots/Lab-02_038_Switch-SSH-RSA-Key-Regenerated-2048-bit.png)

**Testing both directions.** With the policy and switch config in place, I ran both the path that should work and the path that shouldn't.

![SSH login success — privilege 15 via AD account](Screenshots/Lab-02_010_SSH-Login-Success-Privilege-15-AD-Account.png)

SSH as the AD test account succeeded, `show privilege` returned 15 with no `enable` needed, and `show users` showed the AD username rather than the local `admin` account — the actual proof RADIUS granted access rather than local fallback catching a rejected login.

![SSH access denied via RADIUS for incorrect credentials](Screenshots/Lab-02_039_SSH-RADIUS-Access-Reject-Wrong-Credentials.png)

Then the negative case: an incorrect credential attempt against the same policy, with the switch's own RADIUS debug showing a real `Access-Reject` from the server and a clean `Access denied` at the password prompt. That pair — one granted, one denied — is what actually demonstrates the policy is doing its job in both directions.

**Failover — proven, not just configured.** Confirmed NPS running on DC01 as a baseline, then stopped it.

![NPS/IAS service running baseline on DC01](Screenshots/Lab-02_011_NPS-IAS-Service-Running-Baseline.png)

![DC01 IAS service stopped](Screenshots/Lab-02_012_DC01-IAS-Service-Stopped.png)

![Switch debug showing RADIUS timeouts, failover to DC02, and Access-Accept](Screenshots/Lab-02_013_SSH-Debug-Failover-to-DC02-Access-Accept.png)

This is the strongest single piece of evidence in this lab: switch debug output shows three real RADIUS retransmit timeouts to `192.168.20.2`, then literally `Fail-over to (192.168.30.2:1812,1813)`, then an `Access-Accept` from DC02 carrying the correct `Service-Type=Login` and `shell:priv-lvl=15` attributes.

![DC02's own security event log corroborating the failover](Screenshots/Lab-02_014_DC02-Security-Event-6272-Failover-Corroboration.png)

That's independently corroborated by DC02's own Security log — Event ID `6272`, "Network Policy Server granted access to a user," for the same account, at the same time. Restored DC01's NPS service and set it to Automatic startup afterward.

![DC01 IAS service restored with Automatic startup](Screenshots/Lab-02_015_DC01-IAS-Service-Restored-Automatic-Startup.png)

## Part B — WPA2-Enterprise Wi-Fi

**AP RADIUS client and console setup.** Added the AP (`AP-MOBILITY-EXPRESS`, `192.168.10.4`) as a RADIUS client on DC02, then ran it through the Mobility Express console wizard — management IP, SSID `NOC-STAFF`, WPA2-Enterprise, RADIUS server address.

![DC02 RADIUS client for the access point](Screenshots/Lab-02_016_DC02-NPS-RADIUS-Client-Access-Point.png)

![Aironet 1815 Mobility Express console wizard](Screenshots/Lab-02_017_Aironet-1815-Mobility-Express-Console-Wizard.png)

The switch trunk to the AP came up on Gi0/6 exactly as planned — native VLAN 10, allowed VLANs 10 and 40:

![Switch trunk Gi0/6 to the access point](Screenshots/Lab-02_018_Switch-Trunk-Gi0-6-to-Access-Point.png)

**Certificate chain.** The custom `NOC-NPS-SERVER` template issued certificates to both DC01 and DC02, with Server Authentication and Client Authentication EKUs both present — Server Authentication is what PEAP actually relies on to identify the NPS server and protect the tunnel, and having Client Authentication in the same template leaves it ready to issue client certificates too if this environment adds EAP-TLS later.

![CA-issued certificates for DC01 and DC02 NPS servers](Screenshots/Lab-02_019_CA-Issued-Certificates-DC01-DC02-NPS-Server.png)

![NOC-NPS-SERVER certificate template EKUs](Screenshots/Lab-02_020_NOC-NPS-SERVER-Certificate-Template-EKUs.png)

**Wireless policy.** Built via the NPS 802.1X wizard: connection type Secure Wireless, PEAP with the DC01 certificate, MSCHAPv2 inner method, conditioned on NAS Port Type = Wireless and the `NOC-WiFi-802.1x` group.

![NPS connection request policy for wireless](Screenshots/Lab-02_021_NPS-Connection-Request-Policy-Wireless.png)

![Wireless policy overview and conditions](Screenshots/Lab-02_022_Wireless-Policy-Overview-and-Conditions.png)

![Wireless policy conditions detail](Screenshots/Lab-02_023_Wireless-Policy-Conditions-Detail.png)

![Wireless policy constraints — PEAP](Screenshots/Lab-02_024_Wireless-Policy-Constraints-PEAP.png)

![Wireless policy settings — Service-Type Framed](Screenshots/Lab-02_025_Wireless-Policy-Settings-Framed-Service-Type.png)

Here `Service-Type = Framed` is the right value for network-layer client access, which is what a Wi-Fi connection is. The wizard also carries a `Framed-Protocol = PPP` attribute alongside it; 802.1X authenticators don't use that attribute (RFC 3580 §3.6), so it sits there unused rather than affecting anything.

Network placement is handled separately, on the AP itself: the `NOC-STAFF` WLAN's **VLAN & Firewall** tab maps the SSID to VLAN40, so an authenticated client lands on the client network rather than the AP's management interface.

![Editing the NOC-STAFF WLAN profile on the access point](Screenshots/Lab-02_033_AP-WLAN-Edit-NOC-STAFF-Profile.png)

**AP-side RADIUS redundancy.** The AP itself is configured with both DC01 and DC02 as RADIUS authentication (1812) and accounting (1813) servers, so the failover setup isn't just an NPS-side concern — the AP is genuinely told about both:

![AP RADIUS authentication and accounting servers, both DCs](Screenshots/Lab-02_028_AP-RADIUS-Auth-and-Accounting-Servers-Both-DCs.png)

![AP WLAN security — WPA2-Enterprise, both RADIUS servers listed](Screenshots/Lab-02_029_AP-WLAN-Security-WPA2Enterprise-RADIUS-Servers.png)

**Client connection walkthrough.** A client authenticating to `NOC-STAFF` with domain credentials, trusting the PEAP server certificate issued to `DC01.noc.local` by `NOC-LAB-ROOT-CA`, and showing up in the AP's live client list:

![Client connecting to NOC-STAFF with AD credentials](Screenshots/Lab-02_030_Client-Connecting-to-NOC-STAFF-AD-Credentials.png)

![Client trusting the PEAP server certificate issued to DC01](Screenshots/Lab-02_031_Client-Server-Certificate-Trust-Prompt-DC01.png)

![AP network summary showing the active client](Screenshots/Lab-02_032_AP-Network-Summary-Active-Clients.png)

Client-side and AP-side together confirm the same result: a successful connection lands on VLAN40.

![AP client view confirming VLAN40, PEAP, and RADIUS authentication](Screenshots/Lab-02_040_AP-Client-View-VLAN40-PEAP-RADIUS.png)

![Client IP configuration confirming a VLAN40 address over Wi-Fi](Screenshots/Lab-02_041_Client-IP-Configuration-VLAN40-Wireless.png)

The AP's own Mobility State view shows the client tagged `Client (VLAN40)`, authenticated via PEAP against the RADIUS server; the client's own `ipconfig /all` shows a `192.168.40.11` address, correct gateway, and both DCs as DNS servers — the same evidence pattern from two independent vantage points used throughout this lab.

**Testing both directions here too.** `test2noc`, a member of `NOC-WiFi-802.1x`, connected and was granted — Event ID `6272` on DC01.

![Wireless authentication granted — Event 6272, test2noc](Screenshots/Lab-02_026_Wireless-Auth-Granted-Event-6272-test2noc.png)

`test1noc`, a member of the switch-admin group but not the Wi-Fi group, was correctly denied — Event ID `6273`.

![Wireless authentication denied — Event 6273, test1noc](Screenshots/Lab-02_027_Wireless-Auth-Denied-Event-6273-test1noc.png)

That denial confirms the group scoping is actually gating access between the switch-admin and Wi-Fi populations, not just granting access to whichever account happens to be tested.

**Wireless failover — proven the same way as SSH.** Stopped NPS on DC01, then authenticated a client to `NOC-STAFF` again.

![DC01 NPS stopped for the wireless failover test](Screenshots/Lab-02_034_DC01-IAS-Stopped-Wireless-Failover-Test.png)

![DC02 Event 6272 showing Authentication Server DC02.noc.local during the DC01 outage](Screenshots/Lab-02_035_DC02-Event-6272-Authentication-Server-DC02.png)

DC02's own Security log shows Event ID `6272`, with `Authentication Server: DC02.noc.local`, PEAP, EAP-MSCHAPv2, granted to `test2noc`.

![DC02 Event 6272 detail — user test2noc](Screenshots/Lab-02_036_DC02-Event-6272-User-test2noc.png)

![AP client view confirming the same session live, post-failover](Screenshots/Lab-02_037_AP-Client-View-test2noc-Post-Failover.png)

The AP's own client view confirms the same session live. Same evidence standard as Part A: an actual outage, and independent server-side corroboration that the surviving DC handled it.

## Validation Matrix

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| Switch RADIUS client parity, DC01 and DC02 | Identical client config on both | Confirmed | PASS |
| Switch-admin policy — authorized account | AD account in `NOC-Network_Admins` gets privilege 15 via SSH | Confirmed, `show users` shows AD account | PASS |
| Switch-admin policy — unauthorized/incorrect credential | RADIUS denies access | Confirmed, real `Access-Reject` from NPS | PASS |
| Switch SSH host key | 2048-bit RSA minimum | Confirmed at 2048-bit | PASS |
| Switch RADIUS failover | DC02 answers when DC01's NPS is down | Confirmed via switch debug (Fail-over line, Access-Accept) and DC02 Event 6272 | PASS |
| Wireless policy — authorized account | Member of `NOC-WiFi-802.1x` granted | Confirmed, Event 6272 | PASS |
| Wireless policy — unauthorized account | Non-member denied | Confirmed, Event 6273 | PASS |
| Wireless client VLAN placement | Client lands on VLAN40 | Confirmed both client-side (`ipconfig`) and AP-side (Client View) | PASS |
| Wireless RADIUS failover | DC02 answers when DC01's NPS is down | Confirmed via DC02 Event 6272, `Authentication Server: DC02.noc.local` | PASS |

## What I Learned

The two RADIUS attribute details in Part A — Service-Type for a shell login, and the exact syntax of a Cisco AV-Pair — both come down to the same thing: NPS generates a perfectly valid Access-Accept regardless of whether the attributes inside it are what the receiving device actually needs. Neither issue shows up as an NPS-side error; both only become visible from the device's own logs, which is what makes `debug radius` on the switch the actual diagnostic tool, not the NPS console.

Authentication and network placement are two genuinely separate concerns on the wireless side, handled by two different systems that don't know about each other — NPS decides whether to grant access, and the AP's own WLAN interface mapping decides which VLAN a granted client lands on. Proving both matters: checking the client's actual address is what closes the loop, not just whether it connected.

A positive test proves a working path exists. Pairing it with a negative test — on both the switch and the Wi-Fi side — is what actually demonstrates the group scoping is doing something, rather than assuming a policy's conditions are correct because nothing's obviously broken.

## Interview Questions

**Q: A switch RADIUS login succeeds but immediately drops the SSH session afterward. What would you check first?**
A: I'd check the NPS network policy's Standard RADIUS attributes for Service-Type. If it's left at the default "Framed" — often paired with a Framed-Protocol=PPP attribute — Cisco IOS interprets that as an instruction to start a PPP session on the VTY line, which is invalid for an interactive shell login and drops the connection right after granting access. The fix is setting Service-Type explicitly to "Login" for any policy governing device administration.

**Q: How do you prove a RADIUS failover actually happened, rather than just that the client eventually connected?**
A: The connection succeeding on its own isn't proof — I want to see the failure itself. On the switch side, `debug radius authentication` during the outage shows the retransmit timeouts to the down server followed by an explicit fail-over line to the surviving server. On the server side, the surviving NPS's own Security event log for the same account, at the same time, corroborates it independently. Having both — not just a successful login afterward — is what makes the claim solid.

**Q: Why would the same Cisco AV-Pair attribute work for one RADIUS client but not another, even with an apparently identical NPS policy?**
A: Because the attribute value is a literal string IOS parses, and small typos produce a valid-looking Access-Accept that the device silently can't use. `shell:priv-lvl=15` — colon — is correct syntax; a hyphen in that position produces an Access-Accept with no visible NPS-side error, but `debug radius` on the switch would show it explicitly being ignored as an unrecognized VSA. NPS has no way to know the string is malformed, so the only place that surfaces is on the device consuming it.

**Q: A wireless client authenticates successfully via 802.1X/RADIUS but ends up on the wrong VLAN. What would you check?**
A: Authentication succeeding means RADIUS and the certificate chain are fine — the placement itself is controlled separately, by either the WLAN's own static VLAN/interface mapping on the AP or controller, or by the RADIUS server sending per-user VLAN assignment via `Tunnel-Type`, `Tunnel-Medium-Type`, and `Tunnel-Private-Group-ID` attributes. In this lab the WLAN's VLAN & Firewall interface mapping handles it directly — the simpler approach, and the right one when every user of that SSID should land on the same network.

**Q: Why does the NPS server certificate for PEAP need Client Authentication as well as Server Authentication?**
A: It doesn't, strictly — PEAP-MSCHAPv2 only requires Server Authentication on the NPS certificate, since that's what identifies the server and protects the TLS tunnel. Including Client Authentication in the template doesn't cause any problems for PEAP; it's just unused for this specific use case. It matters more if a client-certificate flow like EAP-TLS gets added later, where a *client* certificate would need it — building the template with both from the start just means it's ready for that without reissuing certificates.
