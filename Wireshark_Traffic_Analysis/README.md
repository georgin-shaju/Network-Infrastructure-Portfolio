# Wireshark Protocol Analysis Lab — ARP, ICMP, TCP, UDP & Switch MAC Learning

*Lab conducted with Pragadeesh as part of Vatanix Technologies IT Network & Hardware Training.*

I wanted to stop treating ARP, TCP, and DNS as diagram concepts and actually watch them happen on the wire. This lab walks through six small experiments — each one captured live in Wireshark — to see what a "ping," a "connection," or a "lookup" really looks like in packets.

## Lab 1 — How does a PC learn another PC's MAC address?

**Setup:** PC1 (`192.168.10.10`) and PC2 (`192.168.10.20`) sit on the same switch. I cleared PC1's ARP cache with `arp -d *`, started a Wireshark capture, then pinged PC2.

Before PC1 could send a single ping, it had no idea which physical port PC2 lived behind — it only had an IP address. So the very first frame out wasn't a ping at all, it was a broadcast: *"who has 192.168.10.20? Tell 192.168.10.10."* Every device on the segment saw that request, but only PC2 answered, giving up its MAC address directly. Only after that handshake did the actual ICMP ping request and reply go out.

| Step | Frame | Source | Destination | What it means |
|---|---|---|---|---|
| ARP Request | 16 | PC1 (`88:ae:dd:bd:ff:08`) | Broadcast (`ff:ff:ff:ff:ff:ff`) | "Who has 192.168.10.20? Tell 192.168.10.10" |
| ARP Reply | 17 | PC2 (`28:f1:0e:36:58:f3`) | PC1 | "192.168.10.20 is at 28:f1:0e:36:58:f3" |
| ICMP Echo Request | 18 | 192.168.10.10 | 192.168.10.20 | PC1 sends the actual ping |
| ICMP Echo Reply | 19 | 192.168.10.20 | 192.168.10.10 | PC2 replies to the ping |

The interesting part is what happens *after*. PC1 doesn't repeat the ARP request for every subsequent ping — it caches the IP-to-MAC mapping the moment it learns it and just looks it up locally from then on. That's why the second, third, and fourth pings in a sequence are noticeably faster than the first: the discovery cost is paid exactly once, until the cache entry expires or gets cleared again.

**Screenshots:** `Screenshots/01_arp_cache_cleared_ping.png` through `05_icmp_echo_reply_frame19.png`

## Lab 2 — Comparing ICMP behaviour across two public destinations

**Setup:** Pinged `google.com` (172.217.24.110) and `8.8.8.8` from PowerShell while Wireshark filtered on `icmp`.

| Destination | Request TTL | Reply TTL | Avg RTT |
|---|---|---|---|
| google.com (172.217.24.110) | 128 | 118 | 10 ms |
| 8.8.8.8 | 128 | 117 | 11 ms |

Both replies started life at TTL 128 — Windows' default — but arrived with different values. That single-digit difference (118 vs 117) is a small but concrete proof that the two destinations sit behind a different number of router hops: every hop along the return path decrements TTL by exactly one, so the number that lands on my machine is really a hop-count fingerprint of the path taken.

**Screenshots:** `Screenshots/06_ping_results_google_and_dns.png`, `07_wireshark_icmp_filter.png`

## Lab 3 — Watching the TCP three-way handshake before HTTPS

**Setup:** Opened `duckduckgo.com` in the browser with Wireshark capturing, then filtered on `tcp`.

| Packet | Source | Destination | Flags |
|---|---|---|---|
| 15 | 172.18.2.8 : 49157 | 20.204.244.192 : 443 | SYN |
| 17 | 20.204.244.192 : 443 | 172.18.2.8 : 49157 | SYN, ACK |
| 18 | 172.18.2.8 : 49157 | 20.204.244.192 : 443 | ACK |

No webpage data moves until both sides have proven they can hear each other in both directions. The client's SYN carries its starting sequence number; the server's SYN-ACK does the same in reverse while confirming the client's; the final ACK closes the loop. Only after that three-step negotiation does the TLS Client Hello — the start of the actual HTTPS session — appear in the capture.

**Screenshots:** `Screenshots/08_duckduckgo_browser_loaded.png` through `10_wireshark_tcp_handshake_filter.png`

## Lab 4 — DNS riding on UDP

**Setup:** Ran `ipconfig /flushdns`, then `nslookup google.com`, with Wireshark filtered on `dns`/`udp`.

| Item | Packet | Details |
|---|---|---|
| DNS Query | 15 | 172.18.2.8 → 8.8.8.8 — Standard query `0x0002` A `google.com` |
| DNS Response | 16 | 8.8.8.8 → 172.18.2.8 — Answer: `google.com` A `142.251.223.14` |
| Source Port | — | 58695 (ephemeral) |
| Destination Port | — | 53 |
| Transaction ID | — | `0x0002` |

A DNS lookup is about as simple as network conversations get — one small question, one small answer — so there's no real benefit to paying for a TCP handshake first. UDP just fires the query and waits for the reply, and the Transaction ID is what ties a specific answer back to its matching question if multiple lookups are in flight.

**Screenshots:** `Screenshots/11_dns_flush_nslookup.png`, `12_wireshark_dns_udp_capture.png`

## Lab 5 — TCP vs UDP, side by side

**Setup:** Looked up `torproject.org` with `nslookup`, then loaded the site in a browser, capturing both the DNS (UDP) and HTTPS (TCP) traffic together.

| Feature | TCP | UDP |
|---|---|---|
| Handshake | Yes — SYN, SYN-ACK, ACK before any HTTPS data | No — the DNS query went out and was answered directly |
| Reliable | Yes — guarantees ordered, correct delivery | No — best-effort only |
| Sequence numbers | Yes — every byte numbered | Not used |
| Acknowledgement | Yes — ACK flag on receipt | None |
| Retransmission | Yes — lost data is automatically resent | No — a lost UDP packet is just gone unless the app resends it |
| Typical use | Web browsing, file transfer, email | DNS, streaming, gaming, VoIP |

Seeing both protocols in the same capture window made the trade-off concrete rather than theoretical: TCP spends packets up front on reliability, UDP spends nothing and leaves reliability to whoever needs it.

**Screenshots:** `Screenshots/13_nslookup_torproject.png` through `16_wireshark_tcp_https_torproject.png`

## Lab 6 — Watching a Cisco switch learn MAC addresses live

**Setup:** Connected to the switch console (PuTTY over COM3), checked the MAC address table before any traffic, pinged from PC1 to PC2, then checked the table again.

| VLAN | MAC Address | Type | Port |
|---|---|---|---|
| 1 | 28f1.0e36.58f3 (PC2) | DYNAMIC | Gi0/5 |
| 1 | 88ae.ddbd.ff08 (PC1) | DYNAMIC | Gi0/3 |

The table was empty before the ping and populated with two entries right after — the same two MAC addresses seen back in the Lab 1 ARP exchange. That's switch MAC learning in its plainest form: the switch doesn't need to be told which device sits on which port, it just watches the source MAC of frames as they arrive and builds the table from traffic alone. Both entries showed as `DYNAMIC`, meaning nobody configured them manually, and by default a Cisco switch will age out an unused dynamic entry after 300 seconds of silence from that address.

**Screenshots:** `Screenshots/17_mac_table_before_ping.png` through `19_mac_table_after_ping.png`

## What I Learned

Running these six labs back to back made the OSI layers feel a lot less like a chart and more like a sequence of dependencies. ARP has to resolve before ICMP can even be addressed at Layer 2; TCP has to shake hands before HTTPS has anywhere to put data; DNS answers a question that everything else depends on, using the cheapest possible protocol to do it. The switch MAC table tied it all together from the other side — the exact same MAC addresses I'd just watched negotiate in Wireshark showed up, unprompted, in the switch's own memory. Nothing here was new in theory, but watching it happen frame by frame is what made it stick.

## Interview Questions

**Q1. Why does a PC send an ARP broadcast instead of just pinging directly?**
Because IP addresses aren't enough to deliver a frame on a LAN — the NIC needs a destination MAC address. If the PC doesn't already have that mapping cached, it has to ask the whole segment, since it doesn't yet know which physical device owns that IP.

**Q2. If TTL starts at 128, why do two different destinations return different reply TTLs?**
Every router that forwards the packet decrements TTL by one. A destination reached through more hops will show a lower reply TTL than one reached through fewer, so the TTL value is effectively a rough hop-count indicator.

**Q3. Why does TCP need three steps instead of just sending data immediately like UDP?**
Because TCP promises reliable, ordered delivery, and it can't keep that promise until both sides have confirmed they can send *and* receive from each other, and have agreed on starting sequence numbers. UDP makes no such promise, so it skips straight to sending.

**Q4. Why is DNS built on UDP when TCP is the "reliable" protocol?**
Because the cost of TCP's handshake would outweigh the benefit for a request this small and latency-sensitive. If a UDP-based query is lost or a response is too large, DNS has fallback mechanisms — retry, or escalate to TCP for large or truncated responses — rather than paying the handshake cost on every single lookup.

**Q5. How does a switch build its MAC address table without any manual configuration?**
It inspects the source MAC address of every frame it receives and records which physical port that frame arrived on. Over time this builds a complete map of which device sits behind which port, entirely from observed traffic, and entries that go quiet are aged out automatically.
