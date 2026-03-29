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

---

## Remote Access via Tailscale

Both tiers work with the Tailscale subnet routing described in the network setup guide. With the Flint 2 configured as a Tailscale subnet router, all camera feeds and any local services (Frigate web UI, Home Assistant dashboard) are accessible from anywhere through Tailscale without exposing anything to the public internet.

**Tier 1:** Open the Reolink app with Tailscale enabled on your phone. The cameras are reachable by their LAN IPs through the subnet route. The Reolink app's P2P mode also works independently, but routing through Tailscale keeps traffic within your encrypted network.

**Tier 2:** Access the Frigate web UI or Home Assistant dashboard through their LAN IPs with Tailscale enabled. All event history, live views, and notifications work remotely as if you were on your home network.

No port forwarding, no cloud relay, no exposed services.

---

## Cost Comparison

| Setup | Upfront Cost (2 cameras) | Monthly Cost | Your Data |
|---|---|---|---|
| Ring (wired, plus subscription) | ~$150-200 | $10-18/month | Amazon's servers |
| Tier 1 (Reolink PoE + microSD) | ~$200-340 | $0 | Your microSD cards |
| Tier 2 (Reolink PoE + Frigate + mini PC) | ~$400-600 | $0 | Your local server |

Ring breaks even with Tier 1 in about 12-18 months of subscription fees. Tier 2 takes longer to recoup but provides significantly more capability with no ceiling on expansion.

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
- **Offsite backup.** Syncing footage to a remote machine over Tailscale for redundancy is possible but adds complexity. This may be covered in a future guide.

**Disclaimer:** This guide documents a working configuration based on the author's own testing. It is not endorsed by or affiliated with Reolink, Frigate, or Home Assistant. Camera placement and surveillance may be subject to local laws regarding recording in public or shared spaces. Check your local regulations before installing exterior cameras.
