## Everyday Use — Modes, Purposes, and Switching Between Them

Once everything is configured, your network operates in several distinct modes depending on what you're doing. Understanding these modes and how to move between them is the key to getting the most out of this setup.

### Mode 1: Default Protected Browsing

**What it is:** All devices on your LAN route internet traffic through NordVPN on the router (Global Mode, local server). Tailscale runs in the background. You're online, your traffic is encrypted, and your ISP sees nothing meaningful.

**When to use it:** General browsing, email, banking, shopping, work — anything where you want privacy but don't need to appear in a different city. The local NordVPN server keeps speeds fast and avoids triggering fraud alerts on financial services.

**How to be in this mode:** This is the default state. NordVPN is on at the router, NordVPN desktop app is closed, Tailscale is running. No action needed.

### Mode 2: Geo-Shifted Streaming

**What it is:** You override the router's local NordVPN tunnel on a specific device by opening the NordVPN desktop or mobile app and connecting to a server in a different city.

**When to use it:** Accessing streaming content available in other regions, bypassing local blackouts, or appearing in a specific US metro area.

**How to enter this mode:** Open the NordVPN app on the device where you want to stream. Select the desired city and connect. Only that device's traffic shifts — all other LAN devices remain on the router's local server.

**How to leave this mode:** Disconnect or close the NordVPN app. The device falls back to the router's local NordVPN tunnel automatically.

### Mode 3: Remote Access to Home Network

**What it is:** You're away from home and need to reach devices on your LAN — your PC, router admin panel, a camera, or any device on 192.168.8.x.

**When to use it:** Checking a security camera, accessing files on a home machine, managing the router, or connecting to a home PC.

**How to enter this mode:** On your phone or laptop, enable Tailscale. All devices on your home LAN are reachable by their local IP (192.168.8.x) through the Tailscale subnet route.

**How to leave this mode:** Disable Tailscale on the remote device. You return to whatever network you're physically on.

**Note on NordVPN and Tailscale on mobile:** Don't run both simultaneously on Android — they both use the VPN slot. Use Tailscale when you need home access, NordVPN when you need internet privacy on the go. Switch between them as needed.

### Mode 4: Full Tunnel Through Home

**What it is:** You're on an untrusted network (hotel WiFi, coffee shop, airport) and want all your traffic to route through your home network as if you were sitting on your couch.

**When to use it:** When you don't trust the local network and want the protection of your home router's NordVPN tunnel wrapping all your traffic.

**How to enter this mode:** If you've configured the MT6000 as a Tailscale exit node (Applications → Tailscale → enable Exit Node, then approve in admin console), open Tailscale on your device → select the MT6000 as your exit node. All traffic now flows: your device → Tailscale tunnel → home router → NordVPN → internet.

**How to leave this mode:** In the Tailscale app, disable the exit node. Traffic returns to the local network.

### Mode 5: Unprotected / Diagnostic

**What it is:** NordVPN is off on both the router and the device. Your real ISP IP is exposed. Tailscale may or may not be running.

**When to use it:** Troubleshooting connectivity issues, speed testing your raw ISP connection, or verifying that VPN tunnels are actually working by comparing IPs.

**How to enter this mode:** Turn off VPN Client in the GL.iNet admin panel. Close the NordVPN app on the device. Run `nslookup myip.opendns.com resolver1.opendns.com` or visit [https://ipleak.net](https://ipleak.net) to confirm your real ISP IP.

**How to leave this mode:** Turn the router VPN back on. Confirm the IP changes.

### Switching Between Modes — Summary

| From | To | Action |
|---|---|---|
| Default Protected | Geo-Shifted Streaming | Open NordVPN app, select city |
| Geo-Shifted Streaming | Default Protected | Close NordVPN app |
| Any mode | Remote Access | Enable Tailscale on remote device |
| Remote Access | Normal browsing | Disable Tailscale on remote device |
| Any mode (away from home) | Full Tunnel Through Home | Tailscale → select MT6000 as exit node |
| Full Tunnel | Normal remote | Disable exit node in Tailscale |
| Any mode | Diagnostic | Turn off router VPN, close NordVPN app |
| Diagnostic | Default Protected | Turn on router VPN |

### General Principles

- **The router VPN is your foundation.** Leave it on. Everything else layers on top of or beside it.
- **NordVPN app is for overrides.** Use it only when you need a different exit location than the router provides. Close it when you're done.
- **Tailscale is for reaching home.** It doesn't replace NordVPN — it solves a different problem. Enable it when you need your home LAN, disable it when you don't.
- **On Android, NordVPN and Tailscale can't run simultaneously.** Pick the one you need for the moment and switch when the task changes.
- **When troubleshooting, strip everything back.** Turn off the router VPN, close apps, verify your real IP, then re-enable one layer at a time to find where a problem is.
