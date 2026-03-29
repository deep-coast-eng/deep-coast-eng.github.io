---
layout: post
title: "Home Network Setup: Tailscale, NordVPN, and GL.iNet MT6000"
date: 2026-03-28
tags: [networking, tailscale, nordvpn, gl-inet, mt6000, wireguard, google-pixel-7a]
---

Setting up a secure home network with remote access using a GL.iNet MT6000 router as the central hub, Tailscale (free plan) for encrypted remote access without port forwarding, and NordVPN on the router for always-on privacy.

## Devices

- 2x Windows PCs (Windows 10/11 Pro)
- 1x Android phone (tested on Google Pixel 7a)
- GL.iNet MT6000 router

## Constraints

- ISP modem does not allow NAT or port forwarding configuration
- Free Tailscale plan (up to 100 devices, 3 users)
- Future additions: PoE switch and IP camera

---

## Tailscale Account and Client Setup

### Create a Tailscale Account

1. Go to [https://login.tailscale.com](https://login.tailscale.com).
2. Sign up using a supported identity provider (Google, Microsoft, GitHub, Apple). There is no standalone username/password option. An IDP account is required.
3. Your tailnet is permanently tied to the IDP you sign up with. To change it later, you would need to create a new tailnet with a different IDP.

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

## MT6000 as Tailscale Subnet Router

The subnet router lets any Tailscale node reach devices on your home LAN (192.168.8.x) without installing Tailscale on those devices. This is critical for IP cameras, PoE devices, and anything that cannot run Tailscale.

### Enable Tailscale on the Router

1. Access the GL.iNet admin panel at `192.168.8.1`.
2. Go to **Applications > Tailscale**.
3. Enable Tailscale.
4. Use the device bind link to connect the router to your Tailscale account.
5. **Enable "Allow Remote Access LAN" before binding.** The route advertisement is sent during the initial registration. If it was off when you first bound, you may need to disconnect and re-bind.

### Approve the Subnet Route

1. Go to the Tailscale admin console and click your MT6000 in the machine list.
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
   - Confirm the subnet route (`192.168.8.0/24`) is approved in the Tailscale admin console under your MT6000's route settings.
   - Confirm Tailscale is enabled and connected on both the router and the phone.
   - Confirm the phone is actually on cellular, not WiFi. If it is on the same LAN, the test proves nothing.

---

## NordVPN on the MT6000

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

Both services run simultaneously on the MT6000 in Global Mode. Tailscale operates at a different network layer and maintains connectivity for remote access while NordVPN handles all internet-bound traffic.

Default state: Router VPN always on (Global Mode). Tailscale always on.

Verified working behavior:

- ipleak.net shows NordVPN IP, not ISP IP
- No DNS timeouts
- Tailscale remote access works from Android phone over cellular
- LAN devices reachable via Tailscale subnet route through NordVPN tunnel

---

## Future Additions

### PoE Switch and IP Camera

When adding a PoE switch and camera to the LAN:

- Connect the PoE switch to the MT6000.
- The camera gets a LAN IP (e.g., 192.168.8.50).
- With the subnet route approved in Tailscale, the camera is reachable from any Tailscale node remotely. No additional configuration needed.

### Linux Migration

When migrating Windows machines to Linux:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Sign in with the same IDP account. NordVPN has a native Linux client as well.

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

### IDP Requirement

Tailscale does not support standalone username/password accounts. All users must authenticate through a supported identity provider (Google, Microsoft, GitHub, Apple, OIDC). Your tailnet identity is tied to the IDP used at signup.
