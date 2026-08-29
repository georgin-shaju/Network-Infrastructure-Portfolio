# AD Infrastructure Failover & AAA Lab

*Two Windows Server 2025 domain controllers, hot-standby DHCP, redundant DNS, and centralized RADIUS authentication for both switch admin access and Wi-Fi*

Every lab I'd built up to this point ran on a single domain controller. That's fine for learning AD basics, but it sidesteps the actual hard part of running Windows infrastructure — the coordination problems that only show up once a second DC enters the picture: DHCP split-scope and failover, DNS redundancy, and the AD replication issues that come from getting either of those wrong. This lab was about building that second DC properly and then proving, by actually pulling the primary down, that the redundancy held.

**In one sentence:** I built two domain controllers on ESXi across three server-side VLANs, set up DHCP hot-standby failover and DNS redundancy between them, layered centralized RADIUS authentication (NPS) on top for switch admin SSH and WPA2-Enterprise Wi-Fi, then added certificate-based EAP-TLS authentication via GPO autoenrollment — and tested every failure mode by actually taking a DC down, not just by reading the configuration and assuming it would work.

## Topology

![Network topology diagram](01_DNS_DHCP_Failover_and_Redundancy/Screenshots/Lab-01_001_Network-Topology-Diagram.svg)

## Repository structure

```text
AD_Infrastructure_Failover_and_AAA_Lab/
├── README.md                                          (this file)
├── 01_DNS_DHCP_Failover_and_Redundancy/
│   ├── README.md
│   └── Screenshots/                                   (62 files)
├── 02_NPS_Certificate_Services_AAA_SSH_and_WiFi_8021X/
│   ├── README.md
│   └── Screenshots/                                   (41 files)
└── 03_EAP-TLS_Certificate_Authentication/
    ├── README.md
    └── Screenshots/                                   (13 files)
```

## The three parts

**[01 — DNS & DHCP Failover and Redundancy](01_DNS_DHCP_Failover_and_Redundancy/README.md)**
Builds DC01 and DC02 across three server-side VLANs on ESXi, sets up DHCP hot-standby failover for the client network, and gets DNS redundant between both DCs. A real AD replication fault surfaced during testing, got diagnosed and fixed, and the whole thing closes with a genuine DC-down test — DC01 powered off at the host level, and a client still resolving names and holding a lease through DC02 the entire time.

**[02 — NPS, Certificate Services & AAA](02_NPS_Certificate_Services_AAA_SSH_and_WiFi_8021X/README.md)**
Picks up from there and adds centralized authentication on top: switch admin SSH access against AD via RADIUS, and WPA2-Enterprise Wi-Fi doing the same, using NPS on both DCs so either can answer. Both failure modes — the switch's primary RADIUS server going down mid-session, and the same for a Wi-Fi client — are proven with live debug output and independently corroborated by the surviving DC's own security event log. Wireless clients land on the correct client VLAN with an SSH host key hardened to 2048-bit RSA, and both the authorized and denied paths through RADIUS are demonstrated end to end.

**[03 — EAP-TLS Certificate Authentication](03_EAP-TLS_Certificate_Authentication/README.md)**
Swaps the wireless authentication method from a password (PEAP) to a certificate the client machine itself holds (EAP-TLS), distributed automatically to every domain PC via Group Policy autoenrollment rather than installed by hand. Everything else — the AP, the trunk, the RADIUS clients on both DCs — stays exactly as Part 2 built it; only the authentication method and the certificate source change.

## What ties the three parts together

Every RADIUS-dependent service in Parts 2 and 3 sits on top of the AD/DNS/DHCP foundation from Part 1 — NPS on both DCs, the switch and AP both configured with both DCs as RADIUS servers, and every failover test depends on the same DNS redundancy Part 1 had to prove first. None of these parts treats "redundant" or "secure" as a design claim to take on faith — every failover was demonstrated by actually removing the primary and watching the standby pick up the work, and the move from password-based to certificate-based Wi-Fi authentication in Part 3 was proven the same way: by reconnecting a real client and checking exactly what changed.
