---
layout: post
title: "Home Network Setup: Tailscale, NordVPN, and GL.iNet Flint 2"
date: 2026-03-28
tags: [networking, tailscale, nordvpn, gl-inet, mt6000, wireguard, google-pixel-7a]
---

Setting up a secure home network with remote access using a [GL.iNet Flint 2](https://store-us.gl-inet.com/products/flint-2-gl-mt6000-wi-fi-6-high-performance-home-router) router as the central hub, [Tailscale](https://tailscale.com/) (free plan) for encrypted remote access without port forwarding, and [NordVPN](https://nordvpn.com/) on the router for always-on privacy.

## Why This Guide Exists

Setting up a home network that respects your privacy shouldn't require an IT background or enterprise-grade hardware. Most consumer routers offer little beyond basic connectivity, and ISP-provided equipment -- even when it supports port forwarding -- introduces risk the moment you open an inbound port. Port forwarding creates a direct path from the public internet to a device on your LAN. Automated scanners constantly probe for open ports, and any service behind a forwarded port with weak credentials, unpatched firmware, or known vulnerabilities becomes an entry point for unauthorized access. NAT, which most home routers use by default, blocks unsolicited inbound connections as a side effect of address translation, but it was not designed as a security boundary and should not be treated as one. For the modern general consumer who streams, works remotely, manages smart devices, and wants their internet activity private by default, there is no widely accepted, standardized playbook that avoids these trade-offs.

This guide addresses that gap. It walks through building a privacy-first home network from scratch using affordable, consumer-accessible tools: the GL.iNet Flint 2 (GL-MT6000), NordVPN, and Tailscale. The Flint 2 ($159.99 at date of publishing) is an excellent fit for this setup. It runs OpenWrt-based firmware with native support for VPN clients and Tailscale, offers Wi-Fi 6 and 2.5G Ethernet, and provides an accessible admin interface that balances power with usability. NordVPN Basic covers always-on encrypted internet traffic and city-switchable streaming at an introductory price of $81.36 for 24 months. Tailscale provides secure remote access to your entire home network at no cost. The free plan supports up to 100 devices and 3 users, which is more than sufficient for a typical household.

All that is required for this setup are your existing devices (Windows or Linux PCs, Android phone), the free applications listed above, and the GL.iNet Flint 2 (GL-MT6000). No additional hardware, no paid subscriptions beyond NordVPN, and no modifications to your ISP modem. This setup also works with Apple devices -- Tailscale and NordVPN both have native iOS and macOS clients, and the process is effectively the same: install the app, sign in with your identity provider, and the device joins your network. An intermediate user should expect to complete the full setup in approximately 1-2 hours, assuming their devices and tools match the conditions described in this guide.

## Contents

- [Devices](#devices)
- [Tailscale Account and Client Setup](#tailscale-account-and-client-setup)
- [Flint 2 as Tailscale Subnet Router](#flint-2-as-tailscale-subnet-router)
- [NordVPN on the Flint 2](#nordvpn-on-the-flint-2)
- [Tailscale and NordVPN Coexistence](#tailscale-and-nordvpn-coexistence)
- [Everyday Use: Modes, Purposes, and Switching Between Them](#everyday-use-modes-purposes-and-switching-between-them)
- [Quick Reference](#quick-reference)

---

## Devices

This guide was tested with the following, but the setup works with any combination of Windows, macOS, Linux, Android, and iOS devices:

- 2x Windows PCs (Windows 10/11 Pro)
- 1x Android phone (tested on Google Pixel 7a)
- GL.iNet Flint 2 (GL-MT6000) router (firmware 4.x.x -- check [GL.iNet's download center](https://dl.gl-inet.com/router/mt6000) for the latest stable release). The admin panel at System > Upgrade will notify you when updates are available. Keep firmware current, but check the [GL.iNet forum](https://forum.gl-inet.com/) for a few days after a new release before updating -- stable releases have occasionally shipped with DNS or VPN regressions.

---

## Tailscale Account and Client Setup

### Create a Tailscale Account

1. Go to [https://login.tailscale.com](https://login.tailscale.com).
2. Sign up using a supported identity provider (Google, Microsoft, GitHub, Apple). There is no standalone username/password option. An IDP account is required.
3. Your tailnet is permanently tied to the IDP you sign up with. To change it later, you would need to create a new tailnet with a different IDP.
4. The free plan supports up to 100 devices and 3 users, which is more than sufficient for a typical household.

### Install Tailscale on Windows

1. Download from [https://tailscale.com/download/windows](https://tailscale.com/download/windows).
2. Run the installer.
3. Sign in from the system tray icon using the same IDP account.
4. Each machine receives a Tailscale IP (100.x.y.z).
5. Repeat for each Windows machine.

Once two or more machines are signed in, they can reach each other by Tailscale IP regardless of network. Tailscale punches through NAT using WireGuard and DERP relay servers.

### Install Tailscale on Android

1. Install the Tailscale app from the Google Play Store.
2. Sign in with the same IDP account.
3. The phone joins your tailnet and can reach all other nodes by Tailscale IP.

### Rename Machines

In the Tailscale admin console at [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines):

1. Click the `...` menu next to any machine.
2. Select **Edit machine name** and rename as desired.
3. These names become DNS names on your tailnet (e.g., `my-router.tailnet-name.ts.net`).

To rename the tailnet itself: Admin console > Settings > General > Tailnet name.

---

## Flint 2 as Tailscale Subnet Router

The subnet router lets any Tailscale node reach devices on your home LAN (192.168.8.x) without installing Tailscale on those devices. This is critical for IP cameras, PoE devices, and anything that cannot run Tailscale.

### Enable Tailscale on the Router

1. Access the GL.iNet admin panel at `192.168.8.1`.
2. Go to **Applications > Tailscale**.
3. Enable Tailscale.
4. Use the device bind link to connect the router to your Tailscale account.
5. **Enable "Allow Remote Access LAN" before binding.** The route advertisement is sent during the initial registration. If it was off when you first bound, you may need to disconnect and re-bind.

### Approve the Subnet Route

1. Go to the Tailscale admin console and click your Flint 2 in the machine list.
2. Click **Machine settings > Edit route settings** (near the bottom of the dropdown).
3. You should see `192.168.8.0/24` listed. Approve it.

### If No Routes Appear

If "Allow Remote Access LAN" was enabled after the initial bind:

1. In the GL.iNet admin panel, disconnect Tailscale.
2. Re-enable it and re-bind using the device bind link.
3. Ensure "Allow Remote Access LAN" is toggled on **before** binding.
4. Check the admin console again. The route should now appear for approval.

### Verify Subnet Routing

1. On your Android phone, disconnect from WiFi (use cellular only).
2. Enable Tailscale in the app.
3. Navigate to `192.168.8.1` in a browser.
4. If the router admin panel loads, subnet routing is working.
5. If it times out or refuses to connect, check the following:
   - Confirm the subnet route (`192.168.8.0/24`) is approved in the Tailscale admin console under your Flint 2's route settings.
   - Confirm Tailscale is enabled and connected on both the router and the phone.
   - Confirm the phone is actually on cellular, not WiFi. If it is on the same LAN, the test proves nothing.

### Enable Exit Node (Optional)

Configuring the Flint 2 as a Tailscale exit node allows remote devices to route all internet traffic through your home network. This is useful when you are on an untrusted network and want the protection of your home router's NordVPN tunnel. See Mode 4 in the Everyday Use section below.

1. In the GL.iNet admin panel, go to **Applications > Tailscale**.
2. Enable the **Exit Node** toggle.
3. In the Tailscale admin console, click your Flint 2 in the machine list.
4. Click **Machine settings > Edit route settings**.
5. Approve the exit node.

To use it from a remote device, open the Tailscale app and select the Flint 2 as your exit node. All traffic will flow through your home router.

### Disable Key Expiry on the Router

Tailscale node keys expire after 180 days by default. If the router's key expires, it silently drops off the tailnet and subnet routing stops working. You will not get a notification.

1. In the Tailscale admin console, click your Flint 2 in the machine list.
2. Click **Machine settings > Disable key expiry**.

Do this for any device that needs to stay connected permanently without manual re-authentication.

---

## NordVPN on the Flint 2

Running NordVPN on the router protects all LAN traffic by default without installing the app on every device.

### Add NordVPN to the Router

1. In the GL.iNet admin panel, go to **VPN > VPN Client**.
2. Add your NordVPN credentials or token.
3. Select a nearby city server (e.g., San Francisco) for best speed and location-sensitive services like banking.

### Configure Global Mode

1. In the VPN Client section, find the mode button (may be in the upper right).
2. Set to **Global Mode** so all traffic routes through NordVPN.
3. Do not use Policy Mode. It breaks DNS resolution on this firmware revision.

### Verify NordVPN is Working

With the VPN on, run from a Windows command prompt:

```
nslookup myip.opendns.com resolver1.opendns.com
```

The output will include a line like `Address: 203.0.113.45`. Note that IP. Turn the VPN off on the router and run the command again.

- **Success:** The two IPs are different. The VPN-on IP belongs to NordVPN, the VPN-off IP is your ISP.
- **Failure:** Both IPs are the same. This means traffic is not routing through the VPN. Check that the VPN client is connected in the GL.iNet admin panel and that Global Mode is selected, not Policy Mode.

You can also verify at [https://ipleak.net](https://ipleak.net). It should show the NordVPN server location, not your actual city.

Close the NordVPN desktop app when testing the router VPN. The desktop app overrides the router tunnel and will give misleading results.

### Switching Cities for Streaming

The router VPN stays on a local server for everyday use. When you need to appear in a different city for streaming:

1. Open the NordVPN desktop app on the specific machine.
2. Connect to the desired city.
3. The app overrides the router tunnel for that device only.
4. Close the app to fall back to the router's local server.

---

## Tailscale and NordVPN Coexistence

Both services run simultaneously on the Flint 2 in Global Mode. Tailscale operates at a different network layer and maintains connectivity for remote access while NordVPN handles all internet-bound traffic.

Default state: Router VPN always on (Global Mode). Tailscale always on.

Verified working behavior:

- ipleak.net shows NordVPN IP, not ISP IP
- No DNS timeouts
- Tailscale remote access works from Android phone over cellular
- LAN devices reachable via Tailscale subnet route through NordVPN tunnel

---

## Everyday Use: Modes, Purposes, and Switching Between Them

Once everything is configured, your network operates in several distinct modes depending on what you're doing. Understanding these modes and how to move between them is the key to getting the most out of this setup.

### Mode 1: Default Protected Browsing

**What it is:** All devices on your LAN route internet traffic through NordVPN on the router (Global Mode, local server). Tailscale runs in the background. Your traffic is encrypted and your ISP sees nothing meaningful.

**When to use it:** General browsing, email, banking, shopping, work -- anything where you want privacy but don't need to appear in a different city. The local NordVPN server keeps speeds fast and avoids triggering fraud alerts on financial services.

**How to be in this mode:** This is the default state. NordVPN is on at the router, NordVPN desktop app is closed, Tailscale is running. No action needed.

### Mode 2: Geo-Shifted Streaming

**What it is:** You override the router's local NordVPN tunnel on a specific device by opening the NordVPN desktop or mobile app and connecting to a server in a different city.

**When to use it:** Accessing streaming content available in other regions, bypassing local blackouts, or appearing in a specific metro area.

**How to enter this mode:** Open the NordVPN app on the device where you want to stream. Select the desired city and connect. Only that device's traffic shifts. All other LAN devices remain on the router's local server.

**How to leave this mode:** Disconnect or close the NordVPN app. The device falls back to the router's local NordVPN tunnel automatically.

### Mode 3: Remote Access to Home Network

**What it is:** You are away from home and need to reach devices on your LAN -- your PC, router admin panel, a camera, or any device on 192.168.8.x.

**When to use it:** Checking a security camera, accessing files on a home machine, managing the router, or connecting to a home PC.

**How to enter this mode:** On your phone or laptop, enable Tailscale. All devices on your home LAN are reachable by their local IP (192.168.8.x) through the Tailscale subnet route.

**How to leave this mode:** Disable Tailscale on the remote device. You return to whatever network you are physically on.

**Note on NordVPN and Tailscale on mobile:** Do not run both simultaneously on Android -- they both use the VPN slot. Use Tailscale when you need home access, NordVPN when you need internet privacy on the go. Switch between them as needed.

### Mode 4: Full Tunnel Through Home

**What it is:** You are on an untrusted network (hotel WiFi, coffee shop, airport) and want all your traffic to route through your home network as if you were on your couch.

**When to use it:** When you do not trust the local network and want the protection of your home router's NordVPN tunnel wrapping all your traffic.

**How to enter this mode:** If you have configured the Flint 2 as a Tailscale exit node (Applications > Tailscale > enable Exit Node, then approve in admin console), open Tailscale on your device and select the Flint 2 as your exit node. All traffic now flows: your device > Tailscale tunnel > home router > NordVPN > internet.

**How to leave this mode:** In the Tailscale app, disable the exit node. Traffic returns to the local network.

### Mode 5: Unprotected / Diagnostic

**What it is:** NordVPN is off on both the router and the device. Your real ISP IP is exposed. Tailscale may or may not be running.

**When to use it:** Troubleshooting connectivity issues, speed testing your raw ISP connection, or verifying that VPN tunnels are actually working by comparing IPs.

**How to enter this mode:** Turn off VPN Client in the GL.iNet admin panel. Close the NordVPN app on the device. Run `nslookup myip.opendns.com resolver1.opendns.com` or visit [https://ipleak.net](https://ipleak.net) to confirm your real ISP IP.

**How to leave this mode:** Turn the router VPN back on. Confirm the IP changes.

### Switching Between Modes

| From | To | Action |
|---|---|---|
| Default Protected | Geo-Shifted Streaming | Open NordVPN app, select city |
| Geo-Shifted Streaming | Default Protected | Close NordVPN app |
| Any mode | Remote Access | Enable Tailscale on remote device |
| Remote Access | Normal browsing | Disable Tailscale on remote device |
| Any mode (away from home) | Full Tunnel Through Home | Tailscale > select Flint 2 as exit node |
| Full Tunnel | Normal remote | Disable exit node in Tailscale |
| Any mode | Diagnostic | Turn off router VPN, close NordVPN app |
| Diagnostic | Default Protected | Turn on router VPN |

### General Principles

- **The router VPN is your foundation.** Leave it on. Everything else layers on top of or beside it.
- **NordVPN app is for overrides.** Use it only when you need a different exit location than the router provides. Close it when done.
- **Tailscale is for reaching home.** It does not replace NordVPN. It solves a different problem. Enable it when you need your home LAN, disable it when you don't.
- **On Android, NordVPN and Tailscale cannot run simultaneously.** Pick the one you need for the moment and switch when the task changes.
- **When troubleshooting, strip everything back.** Turn off the router VPN, close apps, verify your real IP, then re-enable one layer at a time to find where a problem is.

---

## Quick Reference

| Service | Role | Always On |
|---|---|---|
| Tailscale (router) | Subnet router for LAN access | Yes |
| Tailscale (clients) | Remote access to tailnet | Yes |
| NordVPN (router) | Encrypt all LAN traffic, local server | Yes (Global Mode) |
| NordVPN (app) | Switch cities for streaming | On demand |

### Key IPs

- Router admin: `192.168.8.1`
- Tailscale IPs: `100.x.y.z` (per device, found in admin console)
- Tailscale admin: [https://login.tailscale.com/admin](https://login.tailscale.com/admin)
