---
layout: post
title: Vulnerability Assessment on the CloudEdge Bell 24T Network Stack
subtitle: Leveraging Insecure Communication Between the Cloud Backend, Mobile App, and the Local IoT Device via Transparent Proxy Interception to Bypass TLS and Manipulate Plaintext HTTP Communications.
gh-repo: chezzuhhh.github.io
comments: true
mathjax: true
author: Chase Cooper
---

## Overview
I’ve been researching the security posture of the CloudEdge Bell 24T Smart Doorbell alongside some other smart video surveillance systems for almost a year now, taking it on as my first IoT vulnerability research project. It has been an incredibly rewarding experience, not just for the findings themselves, but for the sheer volume of low-level networking and hardware knowledge learned along the way. 

This research has covered the full wireless communication stack (including both 802.11 Wi-Fi network communications and sub-GHz RF signal reverse engineering) alongside physical hardware components (such as exposed UART serial interfaces and SPI flash chips). Several vulnerabilities have been uncovered during this process, with some currently pending CVE assignment. 

I began this research project focusing specifically on the network communication stack. This post breaks down my process: setting up the interception environment, breaking down the protocol architecture, showcasing discovered vulnerabilities, and outlining remediation steps.

---

## Setting Up the Interception Environment
To capture, analyze, and manipulate network packets, I first had to establish a privileged network position. I configured a Raspberry Pi as a wireless Access Point (AP) using `hostapd` for the wireless network, `dnsmasq` for DHCP/DNS provisioning, and `iptables` rules for kernel-level IPv4 forwarding and masquerading traffic to the Pi's upstream internet connection (`eth0`). This forced all inbound and outbound doorbell traffic through a single controllable bridge. Here is the script I put together to quickly configure this setup:

<img width="953" height="456" alt="image" src="https://github.com/user-attachments/assets/69c39df6-3351-47f9-9aff-b00da8e3999f" />
It also includes commented-out rules to quickly establish the transparent proxy, redirecting all port 80/443 traffic to mitmproxy.

Here is the visual representation of the routing topology established inside the Raspberry Pi:

<img width="676" height="358" alt="image" src="https://github.com/user-attachments/assets/c1aa6003-6bf8-4748-b1ed-397326e21562" />

With the AP running, I ran multiple tshark captures utilizing L2/L3 filters to isolate traffic between the doorbell and the mobile app across local (same LAN) and external (e.g., cellular) interactions to observe how communication behaviors altered between different communication environments. Some examples being restarts, motion triggers, two-way audio, device unpairing & deletion, etc.).


---

## Protocol Architecture

The CloudEdge Bell 24T ecosystem relies on a three-part architecture: the physical IoT edge node, a proprietary cloud backend, and the companion mobile app. System traffic operates across two distinct operational planes after initial DNS lookups for domain IP discovery:

#### Media Plane
This plane is responsible for high-bandwidth real-time video/audio streaming and alert media uploads. For live streaming, the device implements a Peer-to-Peer (P2P) framework using Interactive Connectivity Establishment (ICE) signaling and STUN servers for NAT traversal to negotiate direct UDP streams between the camera and mobile phone, falling back to cloud TURN relays if direct paths fail. Before the video can play, the doorbell recieves commands through TLS to put it in the active state to initiate the video feed. The app and the doorbell then have to negotiate a session. During this negotiation, the doorbell broadcasts configuration data across the network in plaintext JSON. This data includes a static/permanent Device ID (UUID), static UDP listener port, and dynamic Session IDs (SID). The SID's are low-entropy temporary codes for the current video session. Once the devices agree on the terms, the doorbell streams the video and audio to your phone using a flood of UDP packets. The flood of UDP packets contains the video feed as well as commands during the video session, such as activating the siren and voice changer.

When I triggered the motion sensor or rang the doorbell, the device took a picture and sent the image to a cloud OSS bucket (a data container that media gets dumped to), happening in unencrypted HTTP POST & PUT requests. The picture is then fetched and combined with some other data to be passed to the mobile app in an alert/phone notification. By extracting and analyzing the requests, I discovered the device transmits an x-oss-callback header when motion is detected. Decoding this Base64 string presented a JSON payload exposing the userID, deviceID (same UUID as before), and event metadata. Worse, the request included an x-oss-security-token (temporary Security Token Service (STS) credentials). 

<img width="1231" height="515" alt="image" src="https://github.com/user-attachments/assets/edca7bfe-7fc4-401c-9f27-9911a8654227" />
<img width="1252" height="243" alt="image" src="https://github.com/user-attachments/assets/f63a802d-1262-43a1-965e-6eb2a930dced" />

#### Control Plane
This plane consists of a TLS v1.2/1.3 tunnel which is responsible for handling critical C2 actions to and from the cloud (e.g., Delete Device, alter state/functionality, initiate a live video feed).


---

## Vulnerability Breakdown


#### 1. TLS Bypass via Improper Certificate Validation

To evaluate the doorbell's TLS stack, I routed the device traffic destined for ports 443 through the `mitmproxy` transparent gateway. When the doorbell initiated an outbound TLS connection to its cloud C2 backend, `mitmproxy` intercepted the handshake and presented a dynamically generated self-signed certificate. **The Doorbell Did Not Drop the Connection.** The device completely failed to validate the certificate chain and lacked any certificate pinning enforcement. The TLS handshake completed successfully, allowing `mitmproxy` to capture and decrypt the TLS payloads into cleartext, log the contents, optionally pause and edit the payloads, and re-encrypt the traffic to be forwarded upstream.

<img width="1051" height="376" alt="image" src="https://github.com/user-attachments/assets/8e6f7bee-6e46-43b9-a85b-4db06b8cd2f3" />

---

#### 2. Plaintext ICE Signaling & Local P2P Stream Hijacking

During video session negotiation, the camera broadcasts setup configurations across the local subnet in plaintext JSON. This broadcast contains a static Device UUID, and a dynamic Session ID (SID). The mobile app responds by binding a listener socket to a UDP port and advertising its network parameters in cleartext: 

```
{
  "local": {
    "ip": ["<Client_IP>"],
    "port": 16685,
    "UUID": 61c3696166134d69,
    "SID": e3623cc700000002
  }
}
```

The doorbell then takes this JSON payload containing the socket 

Because the ICE signaling lacks mutual authentication, an attacker on the local network can exploit a hole-punching race condition. By intercepting the plaintext JSON handshake and substituting the client's destination socket with an attacker-controlled socket, the doorbell is tricked into establishing the P2P stream directly with the attacker. The unencrypted UDP packet stream (H.264 video feed) can then be forwarded directly on an arbitrary listening socket and piped into media stream players like `ffplay` for unauthorized real-time surveillance:

```
nc -u -l 16685 | ffplay -f h264 -
```

---

#### 3. Replay Attacks, IDOR, Payload Substitution & SSRF

When motion is detected or the doorbell button is pressed, the camera uploads alert imagery to an Object Storage Service (OSS) bucket over **unencrypted HTTP (Port 80)** via `POST` and `PUT` requests.

Interception of these unencrypted uploads revealed several severe flaws:

* The request headers include a cleartext `x-oss-security-token` containing temporary Security Token Service (STS) credentials. Extracting this token grants an attacker full read/write access to the cloud storage bucket.

* While the use of custom signing headers (`X-Ca-Key`, `X-Ca-Nonce`, `X-Ca-Timestamp`) is deployed, the backend servers fail to validate timestamp freshness or track nonce uniqueness. An attacker can capture valid API requests to activate alerts, and replay them arbitrarily. The cloud backend accepts replayed requests without error, allowing an attacker to manipulate device operational states or execute localized Denial of Service (DoS) conditions. Furthermore, because access control relies heavily on static hardware UUIDs, modifying these identifiers within replayed payloads exposes potential Insecure Direct Object Reference (IDOR) pathways for unauthorized cross-account device manipulation.

* The OSS API does not perform client-side cryptographic hash validation on uploaded media. An attacker can intercept a legitimate image upload, substitute the payload with arbitrary image media, and allow it to route upstream. The backend accepts the altered file, pushing forged alert imagery to the user's mobile app. Additionally, because the `x-oss-callback` header is constructed entirely client-side, altering the `callbackUrl` parameter forces the cloud OSS backend to act as an open proxy, creating Server-Side Request Forgery (SSRF) risks against proprietary infrastructure.

---

## Conclusion & Mitigations

This vulnerability assessment highlights critical structural flaws across transport security, local peer discovery, and cloud storage integrations within consumer IoT systems. To mitigate these vectors, the vendor must implement the following architectural fixes:

* **Mandatory Certificate Pinning:** Hardcode the server certificate's public key directly within the device firmware to reject untrusted, self-signed TLS certificates.


* **Enforce Strict TLS Transport:** Deprecate plain HTTP (Port 80) across all media pipelines in favor of TLS 1.2+.


* **Cryptographic Payload Validation:** Implement SHA-256 payload hashing and validate hashes backend-side prior to processing OSS storage callbacks.


* **Replay & API Hardening:** Enforce strict timestamp checking windows and maintain server-side nonce caches to reject replayed API calls.


* **Encrypted Local Signaling:** Deprecate plaintext UDP broadcasts in favor of encrypted discovery protocols (e.g., DTLS) with mutual authentication to prevent stream hijacking.
