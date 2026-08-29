# AD Infrastructure Failover Lab — Part 1: DNS & DHCP Failover and Redundancy

*ESXi host → DC01 + DC02 (Windows Server 2025) → hot-standby DHCP → redundant DNS → verified by pulling DC01's power*

Every lab I'd built up to this point ran on a single domain controller. That sidesteps the actual hard part of running Windows infrastructure — the coordination problems that only show up once a second DC enters the picture. This lab is about building that second DC properly, wiring DHCP and DNS to survive either one going down, and then proving it by actually pulling the primary offline rather than trusting the design on paper.

**In one sentence:** I built two domain controllers on ESXi across three server-side VLANs, configured DHCP hot-standby failover and DNS redundancy between them, found and fixed a real AD replication fault along the way, and closed the lab by powering DC01 off entirely and confirming a client still resolved names and held a lease through DC02 alone.

## Scope and limitations

Both DCs run as VMs on a single ESXi host, sharing the same physical switch, storage, and upstream router. What this lab demonstrates is **service-level redundancy** — either DC can go down and AD, DNS, and DHCP keep working. It doesn't demonstrate infrastructure-level high availability: an ESXi host, switch, or storage failure takes the whole environment down regardless of having two DCs. A production design would put DC02 on separate host/storage/upstream hardware; this lab's scope stops at the service layer.

## Topology

![Network topology diagram](Screenshots/Lab-01_001_Network-Topology-Diagram.svg)

## Addressing plan

| VLAN | Subnet | Gateway | Hosts |
|---|---|---|---|
| 10 — Management | 192.168.10.0/28 | .1 (switch SVI) | ESXi vmk0 .2, iLO .3 |
| 20 — DC01 | 192.168.20.0/28 | .1 | DC01 .2 |
| 30 — DC02 | 192.168.30.0/28 | .1 | DC02 .2 |
| 40 — Clients | 192.168.40.0/28 | .1 | DHCP pool .10–.14 |

The ESXi trunk carries VLANs 10, 20 and 30, with an unused native VLAN999 explicitly configured so nothing untagged lands where it shouldn't.

## Building the host and the first VM

Installing ESXi and standing up the first VM is mostly mechanical, but it's worth walking through because everything downstream depends on the datastore and networking being right from the start.

![Downloading the VMware ESXi installer](Screenshots/Lab-01_002_Download-VMware-ESXi-Installer.png)

![ESXi host initial configuration](Screenshots/Lab-01_003_ESXi-Host-Initial-Configuration.png)

With the host up, I uploaded the Windows Server ISO straight to the datastore rather than mounting it locally — cleaner for repeat VM builds later.

![Uploading the Windows Server ISO to the datastore](Screenshots/Lab-01_004_Upload-Windows-Server-ISO-to-Datastore.png)

DC01's VM went through the standard name/OS/hardware/review wizard flow:

![Creating the Windows Server VM — name and OS](Screenshots/Lab-01_005_Create-Windows-Server-VM-Name-and-OS.png)

![Configuring VM hardware](Screenshots/Lab-01_006_Configure-Windows-Server-VM-Hardware.png)

![Reviewing the VM configuration before creation](Screenshots/Lab-01_007_Review-Windows-Server-VM-Configuration.png)

![VM created](Screenshots/Lab-01_008_Windows-Server-VM-Created.png)

Then the OS install itself, start to finish:

![Installing Windows Server on the VM](Screenshots/Lab-01_009_Install-Windows-Server-on-VM.png)

![Windows Server installation complete](Screenshots/Lab-01_010_Windows-Server-VM-Installation-Complete.png)

Datastore capacity and host client access both checked out clean afterward:

![ESXi datastore capacity](Screenshots/Lab-01_011_ESXi-Datastore-Capacity.png)

![ESXi host client access](Screenshots/Lab-01_012_ESXi-Host-Client-Access.png)

## Proving the VLAN fabric before building anything on top of it

Before promoting a domain controller, I wanted the underlying network to be provably correct — gateway reachability across VLANs, and the ESXi trunk actually carrying what it's supposed to.

![Client-to-VLAN gateway connectivity tests](Screenshots/Lab-01_013_Client-to-VLAN-Gateway-Connectivity-Tests.png)

![Switch-side VLAN gateway ping tests](Screenshots/Lab-01_014_Switch-VLAN-Gateway-Ping-Tests.png)

![Verifying the ESXi trunk carries VLANs 10, 20, 30](Screenshots/Lab-01_015_Verify-ESXi-Trunk-VLANs-10-20-30.png)

The legacy switch's NAT/ACL setup from an earlier build also got a clean bill of health for VLAN40 traffic before I moved on:

![Legacy switch NAT statistics check](Screenshots/Lab-01_016_Legacy-Switch-NAT-Statistics-Check.png)

![Legacy NAT ACL covering VLAN40](Screenshots/Lab-01_017_Legacy-NAT-ACL-VLAN40-27.png)

## DC01 — the first domain controller

With the fabric confirmed, DC01 got promoted, given its static IP and initial DNS configuration, and set up as the first authority for `noc.local`.

![DC01 Server Manager configuration](Screenshots/Lab-01_018_DC01-Server-Manager-Configuration.png)

![DC01 DNS reverse lookup zone](Screenshots/Lab-01_019_DC01-DNS-Reverse-Lookup-Zone.png)

![Verifying DC01 DNS lookups](Screenshots/Lab-01_020_Verify-DC01-DNS-Lookup.png)

![DC01 static IP and DNS configuration](Screenshots/Lab-01_022_DC01-Static-IP-and-DNS-Configuration.png)

At this stage DC02 didn't exist yet, so DC01's own DNS could only point at itself — that comes back later. The routing table and a live ping out to the internet from VLAN20 confirmed DC01 had a working path out before I moved on to the second DC.

![DC01 IP route table](Screenshots/Lab-01_023_DC01-IP-Route-Table.png)

![Switch-side internet ping from VLAN20](Screenshots/Lab-01_024_Switch-Internet-Ping-from-VLAN20.png)

## DC02 — the second domain controller

DC02 needed its own ESXi port group before anything else — I checked the existing groups, reviewed how VLAN10 management networking was laid out, then created the VLAN20-tagged group DC01 used and a matching one for DC02 on VLAN30.

![ESXi existing port groups](Screenshots/Lab-01_025_ESXi-Existing-Port-Groups.png)

![ESXi management network VLAN10 topology](Screenshots/Lab-01_026_ESXi-Management-Network-VLAN10-Topology.png)

![Creating the ESXi VLAN20 port group for DC01](Screenshots/Lab-01_027_Create-ESXi-VLAN20-DC01-Port-Group.png)

DC02 joined the existing `noc.local` domain and went through the ADDS deployment wizard as an additional domain controller — DNS options, additional options, and DC01 selected explicitly as the replication source:

![Adding DC02 to the existing NOC domain](Screenshots/Lab-01_028_Add-DC02-to-Existing-NOC-Domain.png)

![DC02 ADDS deployment configuration](Screenshots/Lab-01_029_DC02-ADDS-Deployment-Configuration.png)

![DC02 ADDS DNS options](Screenshots/Lab-01_030_DC02-ADDS-DNS-Options.png)

![DC02 ADDS additional options](Screenshots/Lab-01_031_DC02-ADDS-Additional-Options.png)

![Selecting DC01 as the replication source](Screenshots/Lab-01_032_Select-DC01-as-Replication-Source.png)

DC02's static IP and DNS configuration went in at this point too:

![DC02 static IP and DNS configuration](Screenshots/Lab-01_021_DC02-Static-IP-and-DNS-Configuration.png)

## DHCP scope and hot-standby failover

With both DCs up, the VLAN40 client scope came next — default gateway, both DNS servers handed out via options 006/015, and WINS skipped since nothing in this environment needs it:

![DHCP scope option 003 — default gateway](Screenshots/Lab-01_033_DHCP-Scope-Option-003-Default-Gateway.png)

![DHCP scope options 006 and 015 — DNS](Screenshots/Lab-01_034_DHCP-Scope-Options-006-and-015-DNS.png)

![Skipping WINS configuration](Screenshots/Lab-01_035_DHCP-Scope-Skip-WINS-Configuration.png)

Both DCs showed up correctly in AD Sites and Services and ADUC before I moved on to failover itself:

![Verifying DC01 and DC02 in AD Sites and Services](Screenshots/Lab-01_036_Verify-DC01-and-DC02-in-AD-Sites.png)

![Verifying DC01 and DC02 in ADUC](Screenshots/Lab-01_037_Verify-DC01-and-DC02-in-ADUC.png)

The failover relationship itself: DC01 active, DC02 standby, 20% address reservation, 1-hour maximum client lead time (MCLT).

![Configuring DHCP hot-standby failover](Screenshots/Lab-01_038_Configure-DHCP-Hot-Standby-Failover.png)

![Reviewing the DHCP failover configuration](Screenshots/Lab-01_039_Review-DHCP-Failover-Configuration.png)

![DHCP failover configuration succeeded](Screenshots/Lab-01_040_DHCP-Failover-Configuration-Success.png)

![Verifying DHCP failover Normal status](Screenshots/Lab-01_041_Verify-DHCP-Failover-Normal-Status.png)

## Client validation

PC01 picked up a correct lease with both DNS servers and the right gateway, and DNS/traceroute tests confirmed both name resolution and the NAT path to the internet worked end to end. PC01 also showed up correctly in ADUC once domain-joined.

![PC01 DHCP lease, gateway, and DNS](Screenshots/Lab-01_042_PC01-DHCP-Lease-Gateway-and-DNS.png)

![PC01 DNS and traceroute tests](Screenshots/Lab-01_043_PC01-DNS-and-Traceroute-Tests.png)

![Verifying PC01's computer object in ADUC](Screenshots/Lab-01_044_Verify-PC01-Computer-in-ADUC.png)

## The failover test that actually proves it

This is the evidence that matters for the "hot-standby" claim, and it's stronger than the raw stop/start commands alone:

![PC01 holding a DHCP lease issued by DC02 during a DC01 outage](Screenshots/Lab-01_045_PC01-Full-IP-Configuration.png)

With DC01's DHCP service stopped, PC01's lease shows `DHCP Server: 192.168.30.2` — DC02, the standby — proof the failover relationship actually handed out an address rather than just existing on paper. Once DC01 came back, PC01 renewed cleanly back onto it:

![PC01's renewed DHCP lease after DC01 is restored](Screenshots/Lab-01_046_PC01-Renewed-DHCP-Lease.png)

![Stop/restart of the DHCP service that produced the outage window](Screenshots/Lab-01_047_DHCP-Failover-Service-Stop-and-Restart-Test.png)

## Finding a real problem instead of assuming everything was fine

While validating AD/DNS health after the DHCP failover test, I ran through the standard checks — policy update, SYSVOL access — and then `repadmin /replsummary` on both DCs.

![AD DNS policy and SYSVOL validation](Screenshots/Lab-01_048_AD-DNS-Policy-and-SYSVOL-Validation.png)

![repadmin /replsummary showing real replication failures on both DCs](Screenshots/Lab-01_049_AD-Replication-DNS-Failure-Evidence.png)

That wasn't the clean result I expected: `2/5` replication failures (40%) on both DC01 and DC02, error `8524 — The DSA operation is unable to proceed because of a DNS lookup failure`. Digging into it turned up two real gaps — neither DC had a secondary DNS server configured (each pointed only at itself or its partner, with no fallback), and only one of the two subnets had a reverse lookup zone. I fixed both in the same pass: added the missing `30.168.192.in-addr.arpa` reverse zone and gave each DC's DNS client settings a working alternate pointing at its partner. Because both fixes went in together, I can't isolate which one specifically resolved the 8524 error — Microsoft's documented mechanism for that error centers on forward resolution of the source DC's CNAME/host records rather than reverse lookups, so the alternate-DNS fix likely did the heavier lifting, but I'm not claiming more certainty than the evidence supports.

Final infrastructure state, captured once everything below was confirmed working:

![ESXi DC01 and DC02 VM overview](Screenshots/Lab-01_050_ESXi-DC01-and-DC02-VM-Overview.png)

![ESXi final VLAN port group layout](Screenshots/Lab-01_051_ESXi-Final-VLAN-Port-Groups.png)

## Retesting the fix

![repadmin /replsummary clean on both DCs after the fix](Screenshots/Lab-01_052_Replication-Summary-After-Reverse-Zone-Fix.png)

`repadmin /replsummary` came back `0/5` on both DCs immediately after the fix. From there:

![Both reverse lookup zones present and replicated on DC02](Screenshots/Lab-01_053_DC02-Reverse-Lookup-Zones-VLAN20-and-VLAN30.png)

DNS Manager on DC02 now shows both reverse zones — `20.168.192.in-addr.arpa` and `30.168.192.in-addr.arpa` — present and replicated, along with both DCs' host records.

![dcdiag DNS test passing on both DCs](Screenshots/Lab-01_054_DNS-Server-Test-DC01-DC02-Pass.png)

`dcdiag /test:dns` came back **PASS** across every check — Auth, Basc, Forw, Del, Dyn, RReg — on both DC01 and DC02.

![Client nltest, gpupdate, and SYSVOL verification](Screenshots/Lab-01_055_Client-Nltest-GPUpdate-SYSVOL-Verification.png)

`nltest /dsgetdc`, `gpupdate /force`, and `dir \\noc.local\SYSVOL` all ran clean from a client.

![Second repadmin check, still clean](Screenshots/Lab-01_056_Replication-Summary-Second-Check.png)

A second `repadmin /replsummary` check, still `0/5` on both DCs — I wanted to see the clean result twice before trusting it.

![DHCP service restart and authorization check](Screenshots/Lab-01_057_DHCP-Service-Restart-and-Authorization-Check.png)

DHCP service startup type restored to Automatic and both DCs reverified as authorized via `Get-DhcpServerInDC`.

![PC01's DHCP lease after the fix](Screenshots/Lab-01_058_PC01-DHCP-Lease-Post-Fix.png)

![PC01's Resolve-DnsName tests](Screenshots/Lab-01_059_PC01-Resolve-DnsName-Tests.png)

Client-side confirmation closed the loop: the correct dual-DNS lease, and `Resolve-DnsName` returning both DC host records cleanly.

## Proving DNS survives a real DC outage

Everything above proved the fix held under normal conditions. The last thing I wanted was proof it held under an actual outage — not a stopped service, the whole VM powered off.

![DC01's VM powered fully off at the ESXi host level](Screenshots/Lab-01_060_DC01-VM-Powered-Off-ESXi-Host-Client.png)

DC01 powered fully off at the ESXi host level — `Power on` is the only enabled action, console thumbnail black.

![Client ping to DC01 timing out during the outage](Screenshots/Lab-01_061_Client-Ping-DC01-Unreachable-During-Outage.png)

From a client, pings to `192.168.20.2` (DC01) succeed initially, then after `ipconfig /flushdns`, time out completely — 100% loss, confirming DC01 is genuinely unreachable rather than just slow.

![Unqualified DNS resolution succeeding through DC02 while DC01 is confirmed down](Screenshots/Lab-01_062_Client-Unqualified-DNS-Resolution-Survives-DC01-Outage.png)

With DC01 confirmed down, `Resolve-DnsName noc.local` run **unqualified** — no DNS server specified — still resolved successfully. That's the test that actually matters: the client's own resolver, left to its own defaults, fell back to DC02 without any help.

## Validation Matrix

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| ESXi trunk carries VLANs 10/20/30 | Tagged traffic passes for all three VLANs | Confirmed on switch and ESXi sides | PASS |
| DC01 promotion, DNS, static IP | DC01 authoritative for `noc.local` | Confirmed | PASS |
| DC02 promotion as additional DC | Joins domain, replicates from DC01 | Confirmed | PASS |
| DHCP hot-standby failover relationship | DC01 active, DC02 standby, Normal state | Confirmed on both partners | PASS |
| DHCP failover under a real outage | DC02 issues a lease while DC01 is down | Client lease shows `DHCP Server: 192.168.30.2` | PASS |
| AD replication health | `repadmin /replsummary` clean | 2/5 failures found (error 8524), fixed, reverified at 0/5 twice | PASS (after fix) |
| `dcdiag /test:dns` | PASS on all checks, both DCs | PASS — Auth, Basc, Forw, Del, Dyn, RReg | PASS |
| DNS resolution during a real DC outage | Client resolves via DC02 with DC01 fully powered off | Unqualified `Resolve-DnsName` succeeded | PASS |

## What I Learned

Hot-standby DHCP failover proves itself best through the lease's own metadata — which server issued it — not through the admin commands used to trigger the outage alone. That's what made screenshot 045 the real evidence, not the stop/start commands around it.

AD replication's dependency on DNS runs in more directions than the one you interact with day to day. It's easy to build the forward zone and consider DNS "done," since that's the zone you actually query — the reverse zone only matters when something else, like replication's own health checks, needs it. And when two fixes go in during the same pass, I can't claim credit for either individually. Next time I want to isolate root cause specifically, I'd change one variable at a time rather than fix everything that looks wrong at once.

The clearest lesson, though, came from the last test. Querying DC02 directly proves DC02 works. Querying with no server specified, and confirming the client's own resolver falls back on its own, proves failover actually happens for a real client. Those are different claims, and only the second one is the one that matters in an actual outage.

## Interview Questions

**Q: How does Windows DHCP hot-standby failover differ from load-balance mode, and why does that matter for a client-facing lease?**
A: In hot-standby, one partner is authoritative for lease decisions and only the standby honors the same scope if the active partner stops responding within the client lead time. The clearest evidence I have for the handover itself is the `DHCP Server` field in the client's lease pointing at the standby's address during the outage. Load-balance mode instead splits requests between both servers continuously by a configurable ratio, so there's no single "active" server and no equivalent handover signature to look for.

**Q: You found a real AD replication failure during this lab. Walk through how you diagnosed it.**
A: `repadmin /replsummary` showed non-zero failures with error 8524 on both DCs, which specifically points at DNS rather than the AD database. From there I checked each DC's own DNS client configuration and found neither had a secondary DNS server, and I also found only one of the two subnets had a reverse lookup zone. I fixed both in the same pass and reran `repadmin /replsummary` twice afterward, both times at `0/5`. What I can't claim is which specific change fixed it, since I changed two things at once — Microsoft's documented mechanism for 8524 points at forward CNAME/host record resolution, which makes the alternate-DNS fix the more likely cause, but I'd want to isolate the variables in a future retest to state that as fact rather than a reasonable inference.

**Q: Why do domain controllers need a secondary DNS server pointing at another DC, given each DC can run its own DNS service?**
A: Because a DC's local DNS service is a single point of failure for that DC's own name resolution, including for its own replication traffic. If the service briefly restarts or hangs, a DC with no fallback DNS server can't resolve the records it needs — including the source DC's CNAME/host records that error 8524 specifically flags. Pointing each DC's alternate DNS at its partner gives it somewhere to fall back to without depending on an external resolver.

**Q: What's the difference between proving a DNS server works and proving DNS failover works for a real client?**
A: Querying a specific server directly — `nslookup noc.local 192.168.30.2` — only proves that server can answer. It says nothing about whether a client experiencing a real outage would actually reach it. The test that matters is querying with no server specified and confirming the client's own resolver, using its normal timeout and retry behavior, falls back to the surviving server on its own. That's what I did once DC01 was confirmed powered off, and it's a meaningfully stronger claim than a direct query to the server that was never down in the first place.
