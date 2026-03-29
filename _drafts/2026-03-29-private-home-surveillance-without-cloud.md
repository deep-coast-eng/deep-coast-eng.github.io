---
layout: post
title: "Private Home Surveillance Without the Cloud"
date: 2026-03-29
tags: [surveillance, reolink, poe, frigate, home-assistant, privacy, intermediate]
published: false
---

A guide to building a home surveillance system where your footage stays on your network, your alerts stay on your phone, and no company gets to sell your data. This builds on the [home network setup guide](/2026/03/28/home-network-tailscale-nordvpn-mt6000/) and assumes you already have a working Tailscale + NordVPN + Flint 2 configuration.

## Contents

- [Who This Is For](#who-this-is-for)
- [What You Give Up](#what-you-give-up)
- [Tier 1: PoE Cameras with On-Device Detection](#tier-1-poe-cameras-with-on-device-detection)
- [Tier 2: Frigate NVR with AI Object Detection](#tier-2-frigate-nvr-with-ai-object-detection)
- [Remote Access via Tailscale](#remote-access-via-tailscale)
- [How This Connects to the Home Network Setup](#how-this-connects-to-the-home-network-setup)
- [Pitfalls, Risks, and Honest Comparisons](#pitfalls-risks-and-honest-comparisons)
- [Cost Comparison](#cost-comparison)

---

## Who This Is For

This is not a Ring replacement in the plug-it-in-and-forget-it sense. Ring costs less upfront, installs in minutes, and works out of the box. If convenience is the priority and you're comfortable with Amazon storing and analyzing your video footage on their servers, Ring is a fine product.

This guide is for people who have decided that trade-off is not worth it. Cloud surveillance systems like Ring, Nest, and Arlo upload your footage to company servers. That footage can be accessed by the company, shared with law enforcement (sometimes without a warrant), used to train AI models, and factored into data profiles tied to your household. You pay a monthly subscription for the privilege.

The system described here keeps all footage on your local network. No footage leaves your home. No company has access to your video. There are no subscriptions. The trade-off is that setup takes longer, costs more in hardware upfront, and requires an intermediate level of technical comfort. If you followed the network setup guide, you can handle Tier 1. Tier 2 requires familiarity with Docker and configuration files.

---

## What You Give Up

Be honest with yourself about these before starting:

- **Single-app polish.** Ring's app is purpose-built and smooth. The Reolink app is functional but rougher. Tier 2 uses Home Assistant, which is powerful but not as intuitive.
- **Doorbell integration.** PoE doorbell cameras exist (Reolink makes one) but the experience is not as seamless as Ring's doorbell-to-app flow.
- **Community features.** Ring's Neighbors feature and Flock's shared surveillance network don't have a self-hosted equivalent. If you want community-level awareness, this isn't the setup for that.
- **Cloud backup redundancy.** If someone steals your recording device, the footage goes with it. You can mitigate this by syncing clips to a remote machine over Tailscale, but that's additional configuration.
- **Plug-and-play simplicity.** This requires running cable, configuring cameras, and (for Tier 2) running software on a local machine.

What you gain: complete ownership of your footage, no subscriptions, no third-party access, better detection accuracy (especially at Tier 2), and a system that works entirely over your private network with remote access through Tailscale.

---

## Tier 1: PoE Cameras with On-Device Detection

This is the recommended starting point. No server required, no Docker, no configuration files. The cameras do the detection on-device and send alerts through their own app.

### What You Need

| Item | Approximate Cost | Purpose |
|---|---|---|
| Reolink PoE camera (RLC-510A, RLC-810A, or similar "A" series) | $50-80 each | Camera with built-in person/vehicle/animal detection |
| PoE switch (4+ ports, unmanaged) | $30-60 | Powers cameras over Ethernet, connects to Flint 2 |
| microSD card per camera (up to 512GB) | $20-40 each | Local recording on the camera itself |
| Ethernet cable (Cat5e or Cat6) | Varies | Runs from PoE switch to each camera location |

Total for a 2-camera setup: approximately $200-340 with no recurring costs.

### Why Reolink

Reolink's "A" series cameras run person, vehicle, pet, and animal detection on the camera hardware itself. No cloud processing, no subscription. The free Reolink app delivers push notifications with detection labels and lets you view live feeds and recorded footage. Cameras record to their own microSD cards, so there is no central recording device to manage.

Other brands (Amcrest, Dahua, Hikvision) also make PoE cameras with RTSP streams, but Reolink's on-device AI detection and app experience are the closest to Ring-like convenience you'll get without cloud infrastructure.

### Setup

1. Connect the PoE switch to a LAN port on the Flint 2.
2. Connect each camera to the PoE switch with Ethernet cable. The switch provides both power and data over the single cable.
3. Open the Reolink app on your phone. The app will discover cameras on the local network.
4. Follow the app setup for each camera: set a password, name the camera, configure detection zones and alert types.
5. In the app's detection settings, enable the alert types you want (person, vehicle, pet) and disable general motion alerts to avoid false positives from wind, shadows, and rain.
6. Insert a microSD card in each camera. Configure recording to motion-triggered or 24/7 in the app.

Once configured, the cameras record locally and send push notifications to your phone when a person, vehicle, or animal is detected. No additional software or hardware is needed.

### Limitations of Tier 1

- **No unified timeline.** Each camera has its own recording and playback. There is no single view of all events across all cameras.
- **Detection accuracy is decent but not great.** On-device detection works well for clear, well-lit scenes but produces more false positives than a dedicated AI system like Frigate, particularly in rain, at night, or when headlights sweep the frame.
- **No advanced automation.** You cannot trigger actions based on detections (turn on lights, lock doors, send a custom notification) without additional software.
- **microSD card is a single point of failure.** Cards wear out over time with continuous writes. Check them periodically.

### Offsite Backup (Recommended)

The biggest vulnerability in Tier 1 is that your footage lives on microSD cards inside the cameras. If someone steals or destroys a camera, the recording goes with it. Three options, from free to low-cost:

**Option A: Syncthing to a friend's machine (free).** Configure each camera to write motion-triggered clips to an FTP server or NAS on your LAN. Most Reolink cameras support FTP natively in the app settings. Use any cheap always-on device on your network as the FTP target -- an old laptop, a Raspberry Pi with an external drive, or a USB drive plugged into the Flint 2 (the router supports basic network storage via its USB port). Then use [Syncthing](https://syncthing.net/) to sync that folder to a machine on a trusted friend or family member's Tailscale network. Fully encrypted, peer-to-peer, no cloud provider involved. A Raspberry Pi with a USB drive runs about $60-80 and handles this for any number of cameras.

**Option B: Zero-knowledge cloud storage ($4-10/month).** If you don't have a trusted remote machine to sync to, use a zero-knowledge encrypted cloud provider where the company cannot access your files. [Proton Drive](https://proton.me/drive) (Swiss privacy laws, open source, audited) or [Filen](https://filen.io/) (open source, generous storage per dollar) are good options. Use [rclone](https://rclone.org/) on your local backup device to automatically sync the clips folder to the cloud. Only encrypted data leaves your network and the provider holds no decryption keys.

**Option C: Internxt lifetime plan (one-time ~$100-150).** [Internxt](https://internxt.com/) offers zero-knowledge encrypted lifetime plans that eliminate the monthly cost entirely. A 2TB lifetime plan on sale covers years of motion-triggered clips. Same rclone setup as Option B.

This is not required, but it closes the biggest gap in a local-only setup.

For many people, Tier 1 is sufficient. If you want better accuracy, a unified event timeline, or automation, continue to Tier 2.

---

## Tier 2: Frigate NVR with AI Object Detection

Tier 2 adds a local server running Frigate, an open-source NVR that uses AI to detect objects in your camera feeds in real time. This is a significant step up in both capability and complexity.

### What You Need (in addition to Tier 1 hardware)

| Item | Approximate Cost | Purpose |
|---|---|---|
| Mini PC or NAS that supports Docker (e.g., Intel N100-based mini PC) | $150-250 | Runs Frigate, Home Assistant, MQTT broker |
| AI accelerator (optional but recommended) | $0-150 | Offloads object detection from CPU. Intel integrated GPU works via OpenVINO at no added cost on N100 hardware. NVIDIA GPU or Hailo module for better performance. |

### What Frigate Does

Frigate watches your camera streams using a two-stage process. First, it runs lightweight motion detection on a low-resolution sub-stream. When motion is detected, it sends that frame region to an AI model that identifies whether the motion is a person, car, dog, cat, or other object. Only confirmed detections trigger recordings and alerts.

This approach produces far fewer false positives than camera-based detection. You can define zones in the camera view (porch, driveway, sidewalk) and configure alerts only for specific object types entering specific zones. For example: alert when a person enters the porch zone, but ignore vehicles on the street.

Frigate records clips of detected events with thumbnails and supports 24/7 continuous recording with retention policies (keep footage with people for 30 days, everything else for 7 days, for example).

As of version 0.16 (August 2025), Frigate includes face recognition and license plate recognition at no cost.

### What Home Assistant Does

Home Assistant connects to Frigate and provides the automation and notification layer. When Frigate detects a person on your porch, Home Assistant can send a push notification to your phone with a snapshot, turn on a porch light, or trigger any other automation you define.

The Home Assistant mobile app (iOS and Android) is the closest thing to a Ring-like notification experience in a self-hosted setup. You receive a notification with a thumbnail of what was detected and can tap through to the live feed.

### Setup Overview

This is not a step-by-step walkthrough. Frigate and Home Assistant each have extensive documentation, and a detailed guide for both would be its own article. This section outlines what is involved so you can decide if it's within your comfort level.

1. **Install Docker** on your mini PC or NAS.
2. **Deploy Frigate** as a Docker container. Configure it with your camera RTSP stream URLs, detection zones, and object types in a YAML configuration file.
3. **Deploy an MQTT broker** (Mosquitto) as a Docker container. This handles communication between Frigate and Home Assistant.
4. **Deploy Home Assistant** as a Docker container. Install the Frigate integration and configure notifications.
5. **Tune detection.** Adjust motion sensitivity, zone boundaries, and minimum confidence scores to reduce false positives for your specific camera placement and environment.

Resources:
- Frigate documentation: [https://docs.frigate.video](https://docs.frigate.video)
- Home Assistant documentation: [https://www.home-assistant.io/docs](https://www.home-assistant.io/docs)
- Frigate + Home Assistant notification blueprint: available in the Home Assistant community forums

### Is Tier 2 Worth It?

If you have 1-2 cameras and just want alerts when someone is at the door, Tier 1 is enough. Tier 2 is worth the effort if you want:

- Near-zero false positive alerts
- A unified event timeline across all cameras
- Custom automations (lights, locks, custom notifications)
- Centralized recording with retention policies
- Face and license plate recognition
- A system you can expand without hitting a ceiling

### Offsite Backup (Recommended)

Frigate stores recordings on the mini PC's local drive. The same theft/destruction risk applies here, but centralized storage makes backup simpler. The same three options from Tier 1 apply:

**Option A: Syncthing to a remote machine (free).** Use [Syncthing](https://syncthing.net/) to sync the Frigate events and clips folder to a trusted machine on another Tailscale network. Peer-to-peer, encrypted, no cloud. Syncing only the events folder (detected clips and snapshots, not 24/7 continuous recording) keeps bandwidth and storage manageable. A typical household generates a few hundred MB to a few GB of event clips per day depending on camera count and activity.

**Option B: Zero-knowledge cloud ($4-10/month).** Use [rclone](https://rclone.org/) to sync the Frigate events folder to [Proton Drive](https://proton.me/drive) or [Filen](https://filen.io/). Zero-knowledge encryption means the provider cannot access your footage. Rclone can run on a cron schedule so backups happen automatically.

**Option C: Internxt lifetime (one-time ~$100-150).** Same as Option B but eliminates the recurring cost. A 2TB [Internxt](https://internxt.com/) lifetime plan on sale is enough for years of event clips.

---

## Remote Access via Tailscale

Both tiers work with the Tailscale subnet routing described in the network setup guide. With the Flint 2 configured as a Tailscale subnet router, all camera feeds and any local services (Frigate web UI, Home Assistant dashboard) are accessible from anywhere through Tailscale without exposing anything to the public internet.

**Tier 1:** Open the Reolink app with Tailscale enabled on your phone. The cameras are reachable by their LAN IPs through the subnet route. The Reolink app's P2P mode also works independently, but routing through Tailscale keeps traffic within your encrypted network.

**Tier 2:** Access the Frigate web UI or Home Assistant dashboard through their LAN IPs with Tailscale enabled. All event history, live views, and notifications work remotely as if you were on your home network.

No port forwarding, no cloud relay, no exposed services.

---

## How This Connects to the Home Network Setup

This surveillance system is designed to sit on top of the network described in the [home network setup guide](/2026/03/28/home-network-tailscale-nordvpn-mt6000/). That guide is not strictly required, but the combination is what makes this setup work as well as it does. Here is why.

**PoE cameras stay off the internet entirely.** The cameras connect to a PoE switch, which connects to the Flint 2. The cameras have LAN IPs and can talk to devices on your network, but NordVPN on the router wraps all outbound traffic. No camera firmware is phoning home over an unencrypted connection. If a camera has a vulnerability, it is not exposed to the public internet because there are no forwarded ports and no UPnP.

**Tailscale provides remote access without exposure.** The Flint 2 acts as a Tailscale subnet router, which means every device on your LAN (including cameras, the Frigate server, and the Home Assistant dashboard) is reachable through Tailscale from anywhere. You do not need to open ports, configure DDNS, or expose any service to the internet. This is the single biggest security advantage over a DIY setup without Tailscale, where people often resort to port forwarding their NVR or camera web interface.

**The exit node wraps remote viewing in your home VPN.** If you configured the Flint 2 as a Tailscale exit node (covered in the network guide), you can route all your traffic through your home network when you are on untrusted WiFi. That means viewing your cameras from a hotel or coffee shop sends traffic through your encrypted Tailscale tunnel to your home router, then through NordVPN to the internet. No one on the local network can see what you are accessing.

**NordVPN on the router prevents ISP surveillance of your surveillance.** Without the router VPN, your ISP can see that you are sending traffic to and from devices on your network, even if the content is encrypted. With NordVPN in Global Mode, your ISP sees encrypted traffic to a NordVPN server and nothing else. This matters if you are running a system specifically because you value privacy.

**The network handles camera traffic without configuration.** PoE cameras on the LAN behind the Flint 2 are automatically covered by every layer of the network setup: NordVPN for outbound privacy, Tailscale for remote access, and the Flint 2's firewall for inbound protection. No per-camera configuration is needed beyond plugging them into the PoE switch.

If you are building this surveillance system without the home network setup, the cameras and Frigate still work on any network. You lose the remote access without port forwarding, the always-on VPN coverage, and the exit node capability. The cameras are still local-only and subscription-free, but the security posture is weaker.

---

## Pitfalls, Risks, and Honest Comparisons

No system is without trade-offs. This section covers what can go wrong with any home surveillance approach, including this one.

### Common Pitfalls with This Setup

- **Ethernet cable runs are the hardest part.** Getting Cat5e or Cat6 from inside your home to exterior camera locations requires planning, drilling, and possibly conduit. This is the step that stops most people before they start. Budget time for it and watch a few installation videos before committing to camera placement.
- **microSD cards fail silently.** In Tier 1, cameras record to microSD cards that wear out over time. You will not get an alert when one dies. Check them periodically or set up the offsite backup so you have a second copy.
- **Frigate has a learning curve.** Tier 2 involves Docker, YAML configuration files, and tuning detection sensitivity. The documentation is good but it is not a guided installer. Expect to spend a few hours on initial setup and another few hours tuning false positives over the first week.
- **Power outages kill everything.** PoE cameras, the switch, the router, and the Frigate server all need power. A UPS (uninterruptible power supply) on the PoE switch and mini PC keeps the system recording during short outages. Budget $50-100 for a basic UPS if this matters to you.
- **Camera firmware updates.** Like any network device, PoE cameras receive firmware updates. Apply them, but check community forums first, the same way the network guide recommends for the Flint 2. Bad firmware updates can break detection features or RTSP streams.

### Common Pitfalls with Any Surveillance System

- **Legal exposure.** Recording video on your own property in public-facing areas (porch, driveway, yard) is legal in California and most US states. Audio is where the law gets strict. California is a two-party consent state under Penal Code 632, meaning recording any confidential communication without consent from all parties is illegal. If your camera has a microphone enabled and captures a conversation, even on your own porch, that could be a violation. Most PoE cameras including Reolink models ship with microphones enabled by default. **The safest approach is to disable audio recording in the camera settings unless you have a specific reason to keep it on.** If you do keep audio enabled, post clearly visible signage at each camera location stating that audio and video recording is in progress. Visible cameras plus signage weaken any claim of reasonable expectation of privacy, but this is not a legal guarantee in California. Note that Penal Code 633.5 provides a limited exception allowing recordings made to gather evidence of certain serious crimes (including felony violence, extortion, and kidnapping), and evidence obtained under this exception is admissible in court. However, this exception applies to recordings made with a reasonable belief that a crime is being committed at the time. A camera recording 24/7 is not selectively recording based on that belief, so 633.5 is better understood as an after-the-fact defense for admissibility than a blanket permission to keep audio running. In most cases, video alone is sufficient for identification and prosecution. This applies equally to Ring, Nest, and any self-hosted system. Ring's doorbell announces "you are currently being recorded" by default, which provides some notice. Your cameras will not do that unless you configure it. This is not legal advice. Consult an attorney if you have questions about recording laws in your jurisdiction.
- **False sense of security.** Cameras deter and document, but they do not prevent. A camera recording someone stealing a package does not get the package back. Combine surveillance with physical security (locks, lighting, visible camera placement) for actual deterrence.
- **Network dependency.** Cloud systems go down when the company has an outage or your internet drops. Local systems go down when your power or local network fails. Neither is immune. The difference is who controls the fix.

### Ring vs. Nest vs. This Setup

| Feature | Ring | Nest (Google) | This Setup (Tier 1) | This Setup (Tier 2) |
|---|---|---|---|---|
| Setup difficulty | Easy | Easy | Moderate | Hard |
| Monthly subscription | $10-18/camera | $8-15/camera | $0 | $0 |
| Person/vehicle detection | Yes (cloud) | Yes (cloud) | Yes (on-device) | Yes (local AI) |
| Face recognition | No | Yes (cloud) | No | Yes (local, free) |
| License plate recognition | No | No | No | Yes (local, free) |
| Custom detection zones | Basic | Basic | Basic | Advanced (polygonal) |
| False positive rate | Moderate | Moderate | Moderate | Low |
| Footage storage | Amazon's cloud | Google's cloud | Local microSD | Local server |
| Company access to footage | Yes | Yes | No | No |
| Law enforcement access without warrant | Possible | Possible | No | No |
| Remote viewing | Via Ring app (cloud relay) | Via Nest app (cloud relay) | Via Reolink app + Tailscale | Via Home Assistant + Tailscale |
| Works without internet | No | No | Yes (local recording) | Yes (local recording + AI) |
| Two-way audio | Yes | Yes | Yes (via camera app) | Limited (camera-dependent) |
| Doorbell integration | Excellent | Good | Functional | Functional |
| Expandable without added cost | No (subscription per camera) | No (subscription per camera) | Yes | Yes |
| Data used for company AI training | Possible | Possible | No | No |
| Open source | No | No | No (cameras) | Yes (Frigate, Home Assistant) |

The bottom line: Ring and Nest are better products if convenience and polish are your priorities. This setup is a better system if privacy, ownership, and long-term cost are your priorities. Neither is wrong. The question is what you value more.

---

## Cost Comparison

| Setup | Upfront Cost (2 cameras) | Monthly Cost | Your Data |
|---|---|---|---|
| Ring (wired, plus subscription) | ~$150-200 | $10-18/month | Amazon's servers |
| Tier 1 (Reolink PoE + microSD) | ~$200-340 | $0 | Your microSD cards |
| Tier 1 + cloud backup (Internxt lifetime) | ~$300-490 | $0 | Your cards + encrypted cloud |
| Tier 1 + cloud backup (Proton/Filen) | ~$200-340 | $4-10/month | Your cards + encrypted cloud |
| Tier 2 (Reolink PoE + Frigate + mini PC) | ~$400-600 | $0 | Your local server |
| Tier 2 + cloud backup (Internxt lifetime) | ~$500-750 | $0 | Your server + encrypted cloud |
| Tier 2 + cloud backup (Proton/Filen) | ~$400-600 | $4-10/month | Your server + encrypted cloud |

Ring breaks even with Tier 1 in about 12-18 months of subscription fees, even when you add a one-time cloud backup cost. Tier 2 takes longer to recoup but provides significantly more capability with no ceiling on expansion.

The math improves the more cameras you add. Ring charges a subscription per camera. A 4-camera Ring setup at $10/month per camera runs $480/year. A 4-camera Tier 2 setup adds maybe $150 in extra cameras and cables over the 2-camera baseline, with the mini PC already paid for. Frigate is free regardless of how many cameras you connect. By year two of a multi-camera setup, the self-hosted approach is significantly cheaper and the gap only widens from there.

---

## Who Should Build This

This setup makes the most sense for homeowners and long-term renters who plan to stay put. You are running Ethernet cable, mounting cameras, and investing in hardware that lives on your property. This is not a setup for your first apartment or a place you're leasing for a year.

The ideal candidate is someone who:

- Owns their home or has a landlord who permits exterior mounting and cable runs
- Already has or is willing to set up a home network with a capable router (the Flint 2 setup from the companion guide)
- Is comfortable with basic networking concepts and app configuration (Tier 1) or Docker and YAML (Tier 2)
- Has decided that cloud surveillance is not acceptable for their household
- Wants a system they can expand over time without hitting subscription walls or vendor lock-in

If you are renting short-term or want something you can take with you in a box, battery-powered WiFi cameras with local microSD recording (Reolink Argus series, for example) are a more portable option, though they sacrifice the reliability of PoE and wired connections.

---

## What This Guide Does Not Cover

- **Outdoor wiring and mounting.** Running Ethernet cable to exterior camera locations may require drilling, conduit, and weatherproofing. This varies by home and is outside the scope of this guide.
- **Doorbell camera setup.** Reolink and Amcrest make PoE doorbells that work with this setup, but the integration is less polished than a dedicated guide would warrant.

**Disclaimer:** This guide documents a working configuration based on the author's own testing. It is not endorsed by or affiliated with Reolink, Frigate, or Home Assistant. Camera placement and surveillance may be subject to local laws regarding recording in public or shared spaces. Check your local regulations before installing exterior cameras.
