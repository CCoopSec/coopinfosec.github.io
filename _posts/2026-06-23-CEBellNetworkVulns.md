---
layout: post
title: Security Posture of the CloudEdge Bell 24T Network Stack
subtitle: Exploitation of Insecure Communication & Certificate Validation with Cloud Backend & Mobile App
gh-repo: chezzuhhh.github.io
tags: [test]
comments: true
mathjax: true
author: Chase Cooper
---

## Overview
I’ve been researching the security posture of the CloudEdge Bell 24T Smart Doorbell for a few months shy of a year, taking it on as my first IoT vulnerability research project. It has been an incredibly rewarding experience, not just for the findings themselves, but for the sheer volume of low-level networking and hardware knowledge learned along the way. This research has covered the full wireless communication stack (including both network communications & sub-GHz RF signal reverse engineering), and hardware components (such as exposed UART/debug interfaces & SPI flash chips). Several vulnerabilities have been uncovered during this process, with some currently pending CVE assignment. 

I began this research project focusing on the network communication stack, here is a technical breakdown of my process and findings.

The first thing I did was establish a privileged network position to be able to capture and analyze the packets. I used a Raspberry Pi set up as an access (AP) point using `hostapd`, `dnsmasq`, and some `iptables` rules. This forced all inbound and outbound doorbell traffic through a controllable bridge for interception and analysis.

I wrote a quick bash script to force a static IP onto `wlan0`, bring the interface up cleanly, enable IPv4 kernel forwarding, and configure `iptables` to masquerade traffic from the AP to the Pi’s upstream (WLAN) connection (`eth0`).

<img width="953" height="456" alt="image" src="https://github.com/user-attachments/assets/69c39df6-3351-47f9-9aff-b00da8e3999f" />
It also includes commented-out rules to quickly establish the transparent proxy, redirecting all port 80/443 traffic to `mitmproxy`.

With the AP running, I ran multiple `tshark` captures utilizing L2/L3 filters to isolate traffic between the doorbell and the mobile app across local (same WiFi) and external (e.g., cellular) interactions. Some examples being restarts, motion triggers, two-way audio, device unpairing & deletion, etc.). 

The easiest way to explain my findings from some rough hours of manual packet analysis (as well as the use of some custom Python scripts found on my GitHub) is to break the important protocols used and the security flaws involved into two planes:

### Media/Data Plane
When you open the app to check your doorbell camera, several things happen to get the video to your screen:
1. When the phone and doorbell are on different private networks protected by firewalls and NAT and they need to connect directly, the system uses ICE (a framework for finding the best connection path). It uses STUN servers to help the devices discover their own public IP addresses, and if a direct connection fails, it uses TURN servers to relay the video traffic.
2. Before the video starts playing, the app and the doorbell have to negotiate a session. During this phase, the doorbell broadcasts configuration data across the network in plaintext JSON (unencrypted, easily readable text). This data includes a static/permanent Device ID (UUID), static UDP listener port, and dynamic Session IDs (SID). The SID's are low-entropy temporary codes for the current video session. Once the devices agree on the terms, the doorbell streams the video and audio to your phone using a flood of UDP packets. 
4. Because the handshake happens in unencrypted plaintext and the Session IDs are easily guessable, an attacker can capture the permanent IDs and session IDs (or brute force the SID since the low-entropy factor) to access the live video feed, trigger the siren, or overwhelm the device with traffic (a local Denial of Service attack).

When I triggered the motion sensor or rang the doorbell, the device took a picture and sent the image to a cloud OSS bucket (a data container that media gets dumped to), happening in unencrypted HTTP POST & PUT requests. The picture is then fetched and combined with some other data to be passed to the mobile app in an alert/phone notification. By extracting and analyzing the requests, I discovered the device transmits an x-oss-callback header when motion is detected. Decoding this Base64 string presented a JSON payload exposing the userID, deviceID (same UUID as before), and event metadata. Worse, the request included an x-oss-security-token (temporary Security Token Service (STS) credentials). 

<img width="1231" height="515" alt="image" src="https://github.com/user-attachments/assets/edca7bfe-7fc4-401c-9f27-9911a8654227" />
<img width="1252" height="243" alt="image" src="https://github.com/user-attachments/assets/f63a802d-1262-43a1-965e-6eb2a930dced" />

On top of all of this, the server does not validate that the x-Ca-nonce header was really only used once, meaning the replay prevention parameters the developers implemented are completely useless. All of these factors present some pretty serious security concerns, such as a malicious actor's ability to read and write to the user’s cloud storage bucket, as well as all data passing through (e.g., alerts, images).

### Control Plane
This plane consists of a TLS v1.2/1.3 tunnel which is responsible for handling critical actions to and from the cloud (e.g., Delete Device, alter functionality, initiate a live video feed).

While the cloud server responsible for the IoT-to-cloud control plane utilizes custom HMAC headers (`X-Ca-Key`, `X-Ca-Nonce`), the backend fails to strictly validate timestamp and nonce uniqueness. This cryptographic failure allows an attacker to capture and replay API requests to manipulate the device state.

Because hardware identifiers (UUIDs) heavily dictate access controls, the leakage of these identifiers could (in theory - this has not been tested to confirm) open pathways for cross-account manipulation (**IDOR**), such as unauthorized binding requests or state modifications. Additionally, the `x-oss-callback` header (which instructs the cloud on where to send event notifications) is constructed entirely client-side. Modifying this header forces the OSS infrastructure to act as a proxy, potentially exposing internal networks via Server-Side Request Forgery (**SSRF**).

##### Exploiting CWE-295 (Improper Certificate Validation)

Since I didn’t have root access (yet) to drop a custom root CA into the doorbell’s firmware, I wanted to test how the security of its validation actually was. I presented it with my own self-signed certificate to see if it would blindly accept the connection or drop it. To execute this, I deployed a transparent proxy (a proxy invisible to both the device and cloud server) on the Pi, and performed a **Man-in-the-Middle (MitM) Attack**. By weaponizing the routing capabilities of a Raspberry Pi to work alongside `mitmproxy`, I was able to capture, analyze, manipulate, and replay network traffic entering and leaving the doorbell.

 zdeployed `mitmproxy` in transparent mode, listening on port 8081:

`mitmproxy --mode transparent --showhost -p 8081 -k`

In a vulnerable environment, when the doorbell attempts to establish a TLS connection with its backend server, `mitmproxy` intercepts the request, dynamically generates a forged TLS certificate for the requested domain, and presents it to the device. If the device accepts, `mitmproxy` then establishes its own secure connection to the actual cloud server, masquerading as the doorbell. Because `mitmproxy` sits in the middle acting as the termination point for both TLS tunnels, it can decrypt the traffic from the doorbell using the forged certificate, log the plaintext payload, re-encrypt it using the legitimate cloud server's certificate, and forward it along. Neither the device nor the server is aware of the interception.

To force the traffic into the proxy rather than allowing it to route normally through the upstream interface, I configured more `iptables` rules to redirect all incoming traffic destined for port 443 to the proxy's listening port:

`sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j REDIRECT --to-port 8081`

Here is a visual representation of the routing logic:

<img width="676" height="358" alt="image" src="https://github.com/user-attachments/assets/c1aa6003-6bf8-4748-b1ed-397326e21562" />

After setting up the MitM environment, I manually connected the doorbell to the AP's SSID and triggered various actions from the paired mobile app. 

The doorbell **did not drop the connection** upon receiving the proxy's untrusted, self-signed certificate. The TLS handshake completed successfully without triggering any certificate pinning or chain validation failure routines. Decrypted HTTPS/TLS traffic to and from the doorbell was entirely accessible.

<img width="1051" height="376" alt="image" src="https://github.com/user-attachments/assets/8e6f7bee-6e46-43b9-a85b-4db06b8cd2f3" />

### Conclusion & Mitigations
The network stack section of the research successfully identified critical security flaws spanning the network layers and the cloud API infrastructure. To resolve these architectural flaws, the vendor needs to implement the following controls:
- **Certificate Pinning:** Hardcode the server certificate's public key directly into the device firmware to completely neutralize SSL/TLS bypass via untrusted self-signed certificates.
- **Secure Transport:** Deprecate plain HTTP communications entirely.
- **Cryptographic Validation:** Implement strict hash-based validation (e.g., SHA-256) for all file uploads and enforce a strict window for request timestamps to neutralize payload substitution and replay attacks.
- **Local Network Hardening:** Deprecate plaintext UDP broadcasting of static UUIDs in favor of localized encryption (DTLS) and require mutual authentication for all localized UDP hardware commands.



```
