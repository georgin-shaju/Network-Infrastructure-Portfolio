# Bringing a Cisco Aironet 1815 Onto the Main Lab Network

## Background

I'd already configured two Cisco access points on their own dedicated VLANs as part of earlier lab work. This time the task was different: take a third AP and put it directly on the **main lab network** — the same flat network the Lab Router and every other device in the lab sits on — with no VLAN segmentation on the switch side. Power comes entirely over Ethernet (PoE) from the switch, so there's no separate power brick to manage, just one cable running from the switch port to the AP.

The goal was to simulate a realistic small-office deployment: bring the AP up from a hard factory reset, configure it entirely from scratch (administrative credentials, hostname, management IP), connect a client to verify the main SSID, and then stand up a guest network that redirects visitors to a landing page — in this case our own company site, vatanix.com — after they log in.

## Topology

The AP is wired into the same PoE-capable switch used in the existing `Cisco_Real_Hardware` lab, plugged into port Gi0/4. No VLAN configuration was applied on the switch for this port, so the AP — and everything it broadcasts — lives on the default VLAN, right alongside the rest of the main network.

```mermaid
flowchart LR
    Router["Lab Router<br/>10.10.11.1/24<br/>DHCP + Default Gateway"]
    Switch["PoE Switch<br/>(shared with Cisco_Real_Hardware lab)<br/>Gi0/4 — default VLAN, untagged"]
    AP["Cisco Aironet 1815<br/>Mobility Express<br/>Mgmt IP: 10.10.11.50 (static)"]
    Client["Test Laptop<br/>Intel AC 8265 Wi-Fi"]

    Router -- Ethernet --> Switch
    Switch -- PoE --> AP
    AP -. "SSID: IT_TRAINEE (WPA2-Personal)" .-> Client
    AP -. "SSID: IT_GUEST1 (Guest / Captive Portal)" .-> Client
```

| Segment | Device / Interface | Address | Notes |
|---|---|---|---|
| Lab Router | LAN interface | 10.10.11.1/24 | Default gateway and DHCP server for the whole main network |
| Switch | Gi0/4 | Default VLAN (untagged) | Same switch as the `Cisco_Real_Hardware` lab; no VLAN config applied for this AP |
| AP management | Cisco Aironet 1815 (Mobility Express) | 10.10.11.50, static, GW 10.10.11.1 | Configured through the console setup wizard right after factory reset |
| Employee WLAN | SSID `IT_TRAINEE` | WPA2-Personal | Clients pull an address from the Lab Router's DHCP scope on 10.10.11.0/24 |
| Guest WLAN | SSID `IT_GUEST1` | Guest / Local User Account, Internal Splash Page | Captive portal hosted on the controller's virtual gateway (192.0.2.1), redirects to vatanix.com after login |

## Starting from a Hard Reset

Rather than picking up someone else's configuration, I wanted the full story — so the first step was a genuine factory reset from the console. Holding the reset button for the full 25+ seconds triggers a NAND erase and a clean reboot, which is a good way to confirm the AP's hardware and bootloader are healthy before trusting anything on top of them.

![Factory reset console boot](Screenshots/01_factory_reset_console_boot.png)

Once it came back up, the AP dropped straight into the day-0 setup wizard over the console — no GUI, no defaults, everything typed by hand: administrative password, a system name, a local admin account for the AP itself, the enable password, country code, NTP configuration (using the New Delhi/Mumbai/Kolkata GMT+5:30 zone), and then the two pieces that actually matter for connectivity — a **static** management IP of 10.10.11.50 with the default router pointed at 10.10.11.1, and the first WLAN, `IT_TRAINEE`, secured with a PSK.

![Initial setup wizard](Screenshots/02_initial_setup_wizard_config.png)

Confirming the wizard writes the configuration to flash and reboots the controller with the new settings applied.

![Config saved, system reboot](Screenshots/03_config_saved_system_reboot.png)

## First Login and Verifying Connectivity

On the next boot, all of the Mobility Express services spin up one by one — CAPWAP, the RF and mesh services, the DHCP and management daemons — ending in a login prompt on the embedded Cisco Controller CLI.

![Services starting, login prompt](Screenshots/04_services_starting_login_prompt.png)

Logging in with the admin account created during setup confirms the controller is alive and the CLI is usable, even before touching the GUI.

![Admin login on controller CLI](Screenshots/05_admin_login_controller_cli.png)

With the AP broadcasting, `IT_TRAINEE` shows up in range on a test laptop.

![Client Wi-Fi list showing IT_TRAINEE](Screenshots/06_client_wifi_list_IT_TRAINEE.png)

Connecting with the PSK set during the wizard confirms the association works end to end.

![Client connected to IT_TRAINEE](Screenshots/07_client_connected_IT_TRAINEE.png)

An `ipconfig /all` on the client confirms it picked up a real lease on the main 10.10.11.0/24 network — gateway and DHCP server both pointing back to the Lab Router at 10.10.11.1. That's expected: since the AP sits on the default VLAN with no NAT or bridging tricks applied, wireless clients land on exactly the same subnet as everything wired into the switch.

![Client ipconfig lease confirmed](Screenshots/08_client_ipconfig_lease_confirmed.png)

## Getting Into the GUI

With Layer 3 confirmed, the next step was reaching the management interface itself at `https://10.10.11.50`. The Mobility Express login splash comes up first.

![GUI login splash page](Screenshots/09_gui_login_splash_page.png)

Clicking through prompts for the admin credentials configured during the initial wizard.

![GUI admin sign-in prompt](Screenshots/10_gui_admin_signin_prompt.png)

And that lands on the Network Summary dashboard — one wireless network, one AP, one active client on 2.4GHz. This is the real confirmation that the whole chain (PoE power, switch port, static management IP, DHCP relay through the router) is actually working, not just the console wizard reporting success.

![Network summary dashboard](Screenshots/11_network_summary_dashboard.png)

At this point, the WLAN configuration page shows exactly one active WLAN — `IT_TRAINEE`, WPA2-Personal, open to all radios.

![WLAN config showing only IT_TRAINEE](Screenshots/12_wlan_config_IT_TRAINEE_only.png)

## Building the Guest Network

The main employee SSID was solid, so the next piece was the guest experience. Cisco's Mobility Express separates this into two parts: a global splash-page configuration (what the captive portal looks like, and where it sends people after login), and the actual guest WLAN itself.

Starting with the splash page — I set the page headline to "Welcome to Vatanix," a short message pointing people to the system admin if anything breaks, and most importantly the **Redirect URL After Login: vatanix.com**, so once someone authenticates on the guest network, their browser gets handed straight to our company site instead of a blank confirmation page.

![Guest WLAN global splash settings](Screenshots/13_guest_wlan_global_splash_settings.png)

Previewing the page before going further confirms the layout: Cisco branding, the welcome message, and a simple username/password form.

![Guest splash page preview](Screenshots/14_guest_splash_page_preview.png)

Back on the WLAN list — still just the one SSID at this point, about to add the second.

![WLAN list before guest added](Screenshots/15_wlan_list_before_guest_added.png)

Adding the new WLAN: ID 2, profile and SSID both named `IT_GUEST1`, enabled, open to all radios, with SSID broadcast turned on.

![Add WLAN general tab, IT_GUEST1](Screenshots/16_add_wlan_general_IT_GUEST1.png)

On the security tab, flipping this WLAN's **Guest Network** toggle on is what actually ties it to the splash-page settings configured earlier — captive portal set to the internal splash page, with local user accounts (created on the controller itself) as the access type. No MAC filtering or ACLs were layered on top of it; this was intentionally kept as a basic, straightforward guest configuration rather than a hardened enterprise one.

![Add WLAN security tab, guest network](Screenshots/17_add_wlan_security_guest_network.png)

I went back into the General tab afterward and enabled **Local Profiling**, so the controller fingerprints connecting devices on this SSID.

![Add WLAN general tab, local profiling on](Screenshots/18_add_wlan_general_local_profiling_on.png)

With that applied, the WLAN list now shows two active networks side by side — `IT_TRAINEE` on Personal (WPA2) and `IT_GUEST1` on the Guest security policy.

![WLAN list, two active WLANs](Screenshots/19_wlan_list_two_active_wlans.png)

## Creating a Guest Account

Guest WLANs using local user accounts need an actual account to log in with, so the next step was adding one under WLAN Users: username `guest1`, a 24-hour (86,400 second) lifetime, tied to the `IT_GUEST1` profile, with a description of "24 Hr Access." The controller enforces its own password policy for these accounts — no reuse of the word "cisco," no reversed username, no three repeated characters in a row, and a minimum mix of character categories.

![Add WLAN user, guest1](Screenshots/20_add_wlan_user_guest1.png)

Once applied, the account shows up in the WLAN Users list, ready to authenticate against.

![WLAN Users list showing guest1](Screenshots/21_wlan_users_list_guest1.png)

## Verifying Both Networks — and Where It Broke

Before switching over, I checked the dashboard while still connected to `IT_TRAINEE`: one client, real usage logged against the SSID, confirming the employee network was carrying traffic normally.

![Clients and usage on IT_TRAINEE](Screenshots/22_clients_usage_on_IT_TRAINEE.png)

With both WLANs now active, I switched the laptop's Wi-Fi profile over to `IT_GUEST1`.

![Two WLANs active, switching to guest](Screenshots/23_two_wlans_active_switching_to_guest.png)

Connecting to a guest SSID for the first time triggers the expected captive-portal detection behavior — Windows flags "no internet, action needed," and the browser tries to reach the controller's captive portal at its virtual gateway address (192.0.2.1). Because that's served over a self-signed certificate, Chrome throws a private-connection warning first, which is normal and expected for an internal lab captive portal rather than a sign of anything wrong.

![Guest captive portal cert warning](Screenshots/24_guest_captive_portal_cert_warning.png)

Proceeding past the warning brings up the actual login page — the same "Welcome to Vatanix" splash configured earlier.

![Guest captive portal login form](Screenshots/25_guest_captive_portal_login_form.png)

Signing in with the `guest1` account created on the controller:

![Guest credentials entered](Screenshots/26_guest_credentials_entered.png)

A successful login pops a small confirmation window ("Login Successful," with a logout link) while the main browser tab automatically redirects to the configured URL —

![Web auth login successful popup](Screenshots/28_web_auth_login_successful_popup.png)

— landing on the actual vatanix.com homepage, confirming the whole guest flow worked end to end: connect, get redirected to the captive portal, authenticate, and land on the intended website.

![Redirected to vatanix.com](Screenshots/27_redirected_to_vatanix_website.png)

**Here's where it got interesting.** After finishing the guest verification, I switched the laptop's Wi-Fi back to `IT_TRAINEE`, the main network. Internet access worked fine over that connection — but I could no longer reach the AP's management GUI at 10.10.11.50 from that same laptop. The page just wouldn't load.

My working theory is that this is tied to the guest WLAN's **Local Profiling** setting I turned on earlier, or a related client-exclusion/isolation behavior on the controller: once a client's MAC address gets registered and authenticated against the guest network, the controller appears to keep treating that MAC as a guest-scoped client even after it reconnects to the trusted SSID, which blocks it from reaching the management interface. I haven't fully isolated the exact mechanism yet — it could also be a stale ARP/DNS entry on the client left over from the captive-portal redirect, since Windows sometimes doesn't cleanly reset routing state after a fast SSID switch. Rebooting the client or clearing its network cache would be the next thing to test to separate a genuine controller-side policy from a client-side quirk.

## What I Learned

- Doing the full flow — hard reset, day-0 console wizard, first GUI login — end to end makes it obvious how many moving pieces have to line up before the AP is even reachable: PoE negotiation on the switch port, the static management IP actually matching the router's subnet, and the controller's own services finishing their startup sequence before the web server responds.
- Guest networking on Mobility Express is split across two places — the global splash-page/redirect settings and the per-WLAN guest toggle — and both have to be configured correctly for the redirect-after-login behavior to work as expected.
- The self-signed certificate warning when a guest client hits the captive portal isn't a misconfiguration; it's expected behavior for an internal virtual gateway address and something worth calling out so it doesn't get mistaken for a real problem.
- Enabling extra client-tracking features like Local Profiling on a guest SSID isn't free — it can have side effects on how the controller treats that client afterward on other networks, which is exactly the kind of thing that's easy to miss in a quick lab setup but would matter a lot in production.
- Putting an AP straight onto the default VLAN with no switch-side segmentation is the simplest possible deployment, but it also means the guest network's only real isolation comes from the WLAN's own guest policy — there's no VLAN boundary backing it up, which is a meaningful limitation compared to a true segmented guest deployment.

## Interview Questions

**Q: Why did the management interface stop being reachable after the client reconnected to the trusted SSID, having previously authenticated as a guest?**
A: The most likely explanation is that the controller retained some state tied to the client's MAC address from its time on the guest network — possibly because Local Profiling was enabled on the guest WLAN — and continued applying guest-scoped restrictions even after the client moved to the trusted SSID. A stale client-side ARP or DNS cache from the captive-portal redirect is a secondary possibility. Confirming which one it is would mean checking the controller's client table for that MAC's assigned policy/VLAN state, and separately testing with a freshly rebooted client to rule out a local caching issue.

**Q: Why does the guest captive portal trigger a certificate warning in the browser?**
A: The Mobility Express controller serves its captive portal over HTTPS using a self-signed certificate on its internal virtual gateway address (192.0.2.1 in this case). Browsers can't validate that certificate against a trusted CA, so they flag it as "not private." This is expected and standard for on-box captive portals unless a proper certificate has been installed and trusted separately.

**Q: What's the difference between the Guest WLAN splash-page settings and the per-WLAN "Guest Network" toggle?**
A: The splash-page settings (under Wireless Settings > Guest WLANs) define the shared content and behavior of the internal captive portal — page headline, message, branding, and the redirect URL after a successful login. That configuration applies globally to any WLAN using the internal splash page. The per-WLAN "Guest Network" toggle, set inside a specific WLAN's security settings, is what actually classifies that WLAN as a guest network and ties it to the captive portal and access-type (local account vs. external RADIUS, etc.).

**Q: Why did the wireless client land on the same subnet (10.10.11.0/24) as the wired devices instead of getting a separate address range?**
A: Because the AP was deliberately placed on the default VLAN with no VLAN segmentation configured on the switch port, and the AP's management DHCP scope was declined during setup. Traffic from the WLAN is bridged straight onto the same Layer 2 domain as the rest of the main network, so wireless clients pick up leases from the Lab Router's existing DHCP scope rather than from a separate pool on the AP itself.

**Q: Why configure the AP entirely through the console wizard instead of just resetting to defaults and using the GUI?**
A: Starting from a genuine factory reset and working through the day-0 CLI wizard confirms the AP's hardware, bootloader, and NAND are healthy, and it also forces every setting — admin credentials, hostname, management IP, first SSID — to be deliberately chosen rather than inherited from a previous configuration. It's closer to what a real first-time deployment looks like out of the box.
