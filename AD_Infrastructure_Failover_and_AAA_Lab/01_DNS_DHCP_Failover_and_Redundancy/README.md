# AD Infrastructure Failover Lab — Part 1: DNS & DHCP Failover and Redundancy

I wanted a lab that went beyond a single domain controller — something that would force me to deal with the coordination problems that show up the moment you add a second DC: DHCP split-scope/failover, DNS redundancy, and the AD replication issues that come from getting either of those wrong. This lab builds two domain controllers on ESXi across three server-side VLANs, sets up DHCP hot-standby failover for the client network, and — as you'll see below — surfaced a real AD replication fault along the way that I had to diagnose and fix rather than write around.

## Scope and limitations

Both DCs run as VMs on a single ESXi host, sharing the same physical switch, storage, and upstream router. What this lab actually demonstrates is **service-level redundancy**: either DC can go down and AD, DNS, and DHCP keep working. It does not demonstrate infrastructure-level high availability — an ESXi host failure, a switch failure, or a storage failure takes the whole environment down regardless of having two DCs. A production design would put DC02 on separate host/storage/upstream hardware; this lab's scope stops at the service layer.

## Topology

![Network topology diagram](Screenshots/Lab-01_001_Network-Topology-Diagram.svg)

## Addressing plan

| VLAN | Subnet | Gateway | Hosts |
|---|---|---|---|
| 10 — Management | 192.168.10.0/28 | .1 (switch SVI) | ESXi vmk0 .2, iLO .3 |
| 20 — DC01 | 192.168.20.0/28 | .1 | DC01 .2 |
| 30 — DC02 | 192.168.30.0/28 | .1 | DC02 .2 |
| 40 — Clients | 192.168.40.0/28 | .1 | DHCP pool .10–.14 |

The ESXi trunk carries VLANs 10, 20 and 30 with an unused native VLAN999 explicitly configured, so nothing untagged lands where it shouldn't.

## Build

**ESXi host and VM provisioning.** Installed ESXi (002–003), uploaded the Windows Server ISO to the datastore (004), and built the DC01 VM through the standard name/OS/hardware/review wizard flow (005–008), then ran the Windows Server install to completion (009–010). Datastore capacity and host client access confirmed clean afterward (011–012).

**VLAN trunking baseline.** Before building anything on top of the VLANs, I confirmed the fabric itself was sound: client-to-gateway connectivity across VLANs (013), switch-side gateway pings (014), the ESXi trunk correctly carrying VLANs 10/20/30 (015), and the legacy NAT/ACL configuration passing its statistics and ACL checks for VLAN40 (016–017).

**DC01 — first domain controller.** Promoted DC01, set its static IP and initial DNS (018, 022), and created the AD-integrated reverse lookup zone for VLAN20 (019) alongside forward zone verification (020). At this stage DC02 didn't exist yet, so DC01's own DNS could only point to itself.

**DC02 — additional domain controller.** Assigned DC02 its ESXi port group (025–027), joined it to the existing `noc.local` domain (028), and ran it through the ADDS deployment wizard — DNS options, additional options, and DC01 selected explicitly as the replication source (029–032). DC02's static IP and DNS configuration (021) was set at this point too.

**DHCP scope for VLAN40 clients.** Built the scope with default gateway (033) and both DNS servers handed out via options 006/015 (034), skipped WINS (035), and confirmed both DCs show up correctly in AD Sites and Services and ADUC (036–037).

**DHCP hot-standby failover.** Configured the failover relationship between DC01 (active) and DC02 (standby, 20% address reservation, 1-hour MCLT) (038–039), watched the wizard complete every step successfully (040), and confirmed `Normal` state on both partners (041).

**Client validation.** PC01 picked up a correct lease with both DNS servers and the right gateway (042), and DNS/traceroute tests confirmed both name resolution and the NAT path to the internet worked end-to-end (043). PC01 showed up correctly in ADUC (044).

**The failover test.** This is the evidence that actually matters for the "hot-standby" claim, and it's stronger than the raw stop/start commands alone:

![PC01 holding a DHCP lease issued by DC02 during a DC01 outage](Screenshots/Lab-01_045_PC01-Full-IP-Configuration.png)

Screenshot 045 shows PC01 holding a lease with `DHCP Server: 192.168.30.2` — DC02, the standby — while DC01 was down. The lease shown also carries an unusually long expiry (Aug 21 → Sep 10). I originally described that as an "MCLT-extended lease," but that's not accurate: the configured Maximum Client Lead Time for this scope is 1 hour, and a 1-hour MCLT extension does not produce a ~20-day lease. I don't have a confirmed explanation for that specific duration, so I'm not asserting one — the part that *is* solid evidence is the `DHCP Server` field itself, which is a direct, unambiguous record of which server answered the request. Screenshot 046 then shows the client renewing back from DC01 (`192.168.20.2`) on a normal 1-hour lease once it was restored, and 047 captures the `Stop-Service`/`Start-Service DHCPServer` commands that produced that outage window.

## A real problem, found by testing rather than assumed away

While validating AD/DNS health after the DHCP failover test (048), I ran `repadmin /replsummary` on both DCs and got a result I wasn't expecting:

![repadmin /replsummary showing 2/5 failures with error 8524 on both DCs](Screenshots/Lab-01_049_AD-Replication-DNS-Failure-Evidence.png)

**Real replication failures**, `2/5` (40%) on both DC01 and DC02, error `8524 — The DSA operation is unable to proceed because of a DNS lookup failure`.

Looking into what could cause that, I found two things that were both genuinely missing:

1. **Neither DC had a secondary DNS entry.** DC01's DNS client list was `127.0.0.1` only (022); DC02's was `192.168.20.2` only (021) — neither had a fallback pointing at its replication partner.
2. **Only one reverse lookup zone existed.** `20.168.192.in-addr.arpa` was present (019), but nothing for DC02's `30.168.192.in-addr.arpa`.

I fixed both in the same pass, which means I can't isolate which one actually resolved the 8524 error — an honest gap in my own testing worth naming directly. Microsoft's documented mechanism for error 8524 centers on the requesting DC being unable to resolve the source DC's GUID-based CNAME record (under the `_msdcs` zone) or its host A/AAAA record — that's a **forward**-resolution dependency, not a reverse one. That means the alternate-DNS fix (giving each DC a working fallback resolver) has the better-documented causal link to 8524; the reverse zone addition was good DNS hygiene and fixed a real gap, but I don't have isolated evidence that it was *the* fix for this specific error. If I retest this in the future, capturing `Resolve-DnsName -Type CNAME <DC-GUID>._msdcs.noc.local`, `Resolve-DnsName DC01.noc.local`, `Resolve-DnsName DC02.noc.local`, and `Get-DnsClientServerAddress` before and after each change independently would let me actually attribute the fix instead of inferring it.

I'm documenting this because it's a real fault I found through testing, not a staged example — and fixing it, then proving the fix, was its own useful exercise, even with that gap in isolating root cause.

## The fix and retest

Fixed both: added the missing `30.168.192.in-addr.arpa` reverse zone (AD-integrated, replicated to both DCs), and set each DC's alternate DNS to point at its partner (DC01 → `192.168.30.2`, DC02 → `192.168.20.2`).

Then retested and captured the before/after:

![repadmin /replsummary showing 0/5 failures on both DCs after the fix](Screenshots/Lab-01_052_Replication-Summary-After-Reverse-Zone-Fix.png)

- **052** — `repadmin /replsummary` immediately after the fix: `0/5` fails on both DC01 and DC02.
- **053** — DNS Manager on DC02 showing both reverse zones (`20.168.192` and `30.168.192`) now present and replicated, plus both DCs' host records.
- **054** — `dcdiag /test:dns` returns **PASS** across every check (Auth, Basc, Forw, Del, Dyn, RReg) for both DC01 and DC02.
- **055** — `nltest /dsgetdc`, `gpupdate /force`, and `dir \\noc.local\SYSVOL` all succeed cleanly.
- **056** — a second `repadmin /replsummary` check, still `0/5` on both DCs.
- **057** — DHCP service startup type restored to Automatic and re-verified authorized on both DCs via `Get-DhcpServerInDC`.
- **058–059** — client-side confirmation: correct dual-DNS lease, and `Resolve-DnsName` returning both DC host records cleanly.

**Final infrastructure state** (050–051): both DC VMs and the final ESXi VLAN port group layout, captured once everything above was confirmed working.

### Before vs. after

| Check | Before fix | After fix |
|---|---|---|
| `repadmin /replsummary` | 2/5 fails (40%), error 8524, both DCs | 0/5 fails, both DCs, confirmed twice |
| Reverse lookup zones | Only `20.168.192.in-addr.arpa` | Both `20.168.192` and `30.168.192`, AD-replicated |
| DC01 DNS client list | `127.0.0.1` only | `127.0.0.1` preferred, `192.168.30.2` alternate |
| DC02 DNS client list | `192.168.20.2` only | `127.0.0.1` preferred, `192.168.20.2` alternate |
| `dcdiag /test:dns` | Not run at time of fault | PASS, all checks, both DCs |
| Client DNS resolution during a real DC01 outage | Not tested | Confirmed — unqualified `Resolve-DnsName` succeeds via DC02 while DC01 is fully powered off |

**What this batch does not, on its own, prove:** it confirms replication is healthy again and that the combined fix worked. Whether a client could still resolve names during an actual DC outage was a separate, later test — see below. It also doesn't isolate which half of the fix (alternate DNS vs. reverse zone) mattered for the 8524 error specifically — see the note above.

## Proving DNS actually survives a DC outage

*Note on naming: the source screenshot for this test is filenamed "DC02 down," but the ping target inside it (`192.168.20.2`) is DC01's address, and that's the host that goes unreachable — so what's actually being tested here is DC01 going down and DC02 covering DNS. I've captioned the screenshots by what they show rather than the original filename.*

![DC01's VM powered fully off at the ESXi host level](Screenshots/Lab-01_060_DC01-VM-Powered-Off-ESXi-Host-Client.png)

- **060** — DC01's VM powered fully off at the ESXi host level (not just a stopped service — `Power on` is the only enabled action, black console thumbnail).
- **061** — from a client, pings to `192.168.20.2` (DC01) succeed initially, then after `ipconfig /flushdns`, time out completely — 100% loss, confirming DC01 is genuinely unreachable, not just slow.

![Unqualified DNS resolution succeeding while DC01 is confirmed down](Screenshots/Lab-01_062_Client-Unqualified-DNS-Resolution-Survives-DC01-Outage.png)

- **062** — with DC01 confirmed down, `Resolve-DnsName noc.local` run **unqualified** (no DNS server specified) still resolves successfully. That's the real test: the client's own resolver, left to its own defaults, fell back to DC02 on its own.

This is the proof I didn't have in the first pass — a real outage, confirmed unreachable, with the client still resolving through it unassisted.

## What I learned

- Hot-standby DHCP failover proves itself best through the lease's own metadata (which server issued it), not through the admin commands used to trigger the outage alone — and I need to be careful not to over-explain metadata I can't fully account for, like the unexplained 20-day lease duration above.
- AD replication's dependency on DNS runs in both directions — forward *and* reverse — and it's easy to build only the forward zone since that's the one you interact with day to day. That said, when two fixes go in together, I can't claim credit for either individually — next time I'd change one variable at a time specifically when the goal is root-causing, not just resolving.
- A DC pointing its DNS client settings only at itself works fine until that DC's own DNS service has a bad moment; the fix is cheap (one alternate DNS entry) but easy to skip because nothing appears broken until you specifically test replication.
- `repadmin /replsummary` is worth running as a matter of routine after any DNS change, not just when something is visibly wrong — this fault produced no user-facing symptoms at all until I went looking.
- Proving DNS failover credibly means querying with no server specified and confirming the client's own resolver falls back on its own — querying the surviving server directly only proves that server works, not that failover actually happens for a real client.

## Interview questions

**Q: How does Windows DHCP hot-standby failover differ from load-balance mode, and why does that matter for a client-facing lease?**
A: In hot-standby, one partner is authoritative for lease decisions and only the standby honors the same scope if the active partner stops responding within the client lead time. When the standby takes over, it's designed to extend leases up to the maximum client lead time (MCLT) to avoid handing out an address the primary might still consider valid once it returns. The clearest evidence I have for the handover itself is the `DHCP Server` field in the client's lease pointing at the standby's address during the outage. Load-balance mode instead splits requests between both servers continuously by a configurable ratio, so there's no single "active" server and no equivalent handover signature to look for.

**Q: You found a real AD replication failure during this lab. Walk through how you diagnosed it, and where your evidence has a gap.**
A: `repadmin /replsummary` showed non-zero failures with error 8524 on both DCs, which specifically points at DNS rather than the AD database. From there I checked each DC's own DNS client configuration and found neither had a secondary DNS server, and I also found only one of the two subnets had a reverse lookup zone. I fixed both in the same pass and reran `repadmin /replsummary` twice afterward, both times at `0/5` — so I know the combined fix worked. What I can't claim is which specific change fixed it, since I changed two things at once. Microsoft's documented mechanism for 8524 points at forward CNAME/host record resolution, which makes the alternate-DNS fix the more likely cause, but I'd want to isolate the variables in a future retest before stating that as fact rather than a reasonable inference.

**Q: Why do domain controllers need a secondary DNS server pointing at another DC, given each DC can run its own DNS service?**
A: Because a DC's local DNS service is a single point of failure for that DC's own name resolution, including for its own replication traffic. If the service briefly restarts, hangs, or has an issue, a DC with no fallback DNS server can't resolve the records it needs — including the source DC's CNAME/host records that error 8524 specifically flags. Pointing each DC's alternate DNS at its partner gives it somewhere to fall back to without depending on an external resolver.
