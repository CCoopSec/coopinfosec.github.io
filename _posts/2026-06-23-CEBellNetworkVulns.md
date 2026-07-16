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
I’ve been researching the security posture of the CloudEdge Bell 24T Smart Doorbell for a few months shy of a year, taking it on as my first IoT vulnerability research project. It has been an incredibly rewarding experience, not just for the findings themselves, but for the sheer volume of low-level networking and hardware knowledge learned along the way. This research has covered the full wireless communication stack (including both network communications & sub-GHz RF signal reverse engineering), and hardware components (such as exposed UART/debug interfaces & SPI flash chips). Several vulnerabilities have been uncovered during this process (some currently pending CVE assignment), but the first critical flaw I found was the firmware's complete lack of TLS certificate validation (CWE-295).

Before going deeper into CWE-295, I want to break down the device's network communication lifecycle. I established a privileged network position by using my Raspberry Pi as an access point (AP), forcing the doorbell to route all traffic through the Pi. I ran many `tshark` captures utilizing L2/L3 filters to isolate traffic between the doorbell and the mobile app across both local and external interactions (restarts, motion triggers, two-way audio, device unpairing, etc.). 

The easiest way to explain my findings from many rough hours of network traffic analysis is to break the communication methods up into three distinct planes:

**Control Plane:** This plane consists of a TLS v1.2/1.3 tunnel which is responsible for handling critical actions to and from the cloud (e.g., Delete Device, alter functionality, initiate a live video feed).

**Media Plane (P2P TUTK):** After a user initiates a live view from the app, the doorbell bypasses NAT/Firewalls using ICE (Interactive Connectivity Establishment) and STUN/TURN techniques to facilitate low-latency video streaming through through ThroughTek's (TUTK’s) P2P servers. The video/audio is streamed through a flood of bidirectional UDP packets. During the P2P handshake, the doorbell broadcasts session negotiation data in plaintext JSON (revealing a constant listener port, static device ID (UUID), and dynamic Session IDs).

**Data Plane:** When I triggered the motion sensor or rang the doorbell, the device took a picture and sent the image to a cloud OSS bucket, which is then passed to the mobile phone in an alert/notification, all happening in unencrypted HTTP POST & PUT requests. By extracting and analysing the requests, I discovered the device transmits an x-oss-callback header when motion is detected. Decoding this Base64 string presented a JSON payload exposing the userID, deviceID (same UUID as before), and event metadata. Worse, the request included an x-oss-security-token (which an attacker could use to access the user's cloud storage bucket), as well as all data passing through (e.g., alerts, images)). On top of all of that, the server does not validate that the x-Ca-nonce header when sending an alert from ringing the doorbell was really only used once, meaning the replay prevention parameters are completely useless allowing an attacker to edit and transmit this alert at will.

<img width="1231" height="515" alt="image" src="https://github.com/user-attachments/assets/edca7bfe-7fc4-401c-9f27-9911a8654227" />
<img width="1252" height="243" alt="image" src="https://github.com/user-attachments/assets/f63a802d-1262-43a1-965e-6eb2a930dced" />

I wrote and used these two custom Python tools to extract the capture data into formats I could programmatically query to use alongside wireshark for easier analysis (the code is on my GitHub):
1. The first script converts raw `.mitm` capture files into JSON. By using the `mitmproxy.io` library on the `.mitm` file to iterate through the intercepted flows, it extracts the raw state data, and decodes the byte streams into UTF-8 strings. Writing the parsed dictionary out to a formatted JSON file allows me to grep across large chunks of traffic for specific API endpoints or authorization headers.
2. The second is a bulk parser that extracts specific protocol fields from `.pcapng` captures and outputs them to CSV I can easily drop into Timeline Explorer. The script recursively searches the working directory for `.pcapng` files and executes a `tshark` subprocess against each one. It utilizes a predefined list of display filters to rip out specific data points across L2/L3/L4 (DNS, HTTP, TLS, STUN/WebRTC).

---

Now, narrowing back in on the control plane, since I didn’t have root access (yet) to drop a custom root CA into the doorbell’s firmware, I wanted to test how the security of its validation actually was. I presented it with my own self-signed certificate to see if it would blindly accept the connection or drop it. To execute this, I deployed a transparent proxy (a proxy invisible to both the device and cloud server) on the Pi, and performed a **Man-in-the-Middle (MitM) Attack**. By weaponizing the routing capabilities of a Raspberry Pi to work alongside `mitmproxy`, I was able to capture, analyze, manipulate, and replay network traffic entering and leaving the doorbell.

I utilized `hostapd` paired with `dnsmasq` (alongside some `iptables` routing rules) to convert the Pi's Wi-Fi card on the `wlan0` interface into a wireless access point. This architecture allowed for full scale network traffic manipulation. With the Pi operating as a fully functional router, the next step was dynamically intercepting specific traffic from the doorbell. To do so, I deployed `mitmproxy` in transparent mode, listening on port 8081:

`mitmproxy --mode transparent --showhost -p 8081 -k`

When the doorbell attempts to establish a secure TLS connection with its backend server, `mitmproxy` intercepts the request, dynamically generates a forged TLS certificate for the requested domain, and presents it to the device. If the device accepts, `mitmproxy` then establishes its own secure connection to the actual cloud server, masquerading as the doorbell. Because `mitmproxy` sits in the middle acting as the termination point for both TLS tunnels, it can decrypt the traffic from the doorbell using the forged certificate, log the plaintext payload, re-encrypt it using the legitimate cloud server's certificate, and forward it along. Neither the device nor the server is aware of the interception.

To force the traffic into the proxy rather than allowing it to route normally through the upstream interface, I configured more `iptables` rules to redirect all incoming traffic destined for port 443 to the proxy's listening port:

`sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j REDIRECT --to-port 8081`

If you have ever tried to set up an AP on a Raspberry Pi, you know that NetworkManager loves to interfere with `hostapd` on who gets to own `wlan0`. To ensure the environment deployed cleanly every time without conflicts, I wrote a deployment script:

<img width="953" height="456" alt="image" src="https://github.com/user-attachments/assets/69c39df6-3351-47f9-9aff-b00da8e3999f" />

This script forces a static IP onto `wlan0` and brings the interface up cleanly. It enables IPv4 kernel forwarding and configures `iptables` to masquerade outbound traffic from the AP through to the Pi's upstream primary connection (`eth0`). It also includes commented-out rules to quickly establish the transparent proxy, redirecting all port 80/443 traffic to `mitmproxy`.

Here is a visual representation of the routing logic:

<img width="676" height="358" alt="image" src="https://github.com/user-attachments/assets/c1aa6003-6bf8-4748-b1ed-397326e21562" />

After setting up the MitM environment, I manually connected the doorbell to the AP's SSID and triggered various actions from the paired mobile app. 

The doorbell **did not drop the connection** upon receiving the proxy's untrusted, self-signed certificate. The TLS handshake completed successfully without triggering any certificate pinning or chain validation failure routines. Decrypted HTTPS/TLS traffic to and from the doorbell was entirely accessible.

<img width="1051" height="376" alt="image" src="https://github.com/user-attachments/assets/8e6f7bee-6e46-43b9-a85b-4db06b8cd2f3" />

---

## Recommended Mitigation
To resolve CWE-295 on this device, the vendor needs to implement the following:
* **Strict Certificate Validation:** The firmware must be updated to enforce strict validation of the SSL/TLS certificate chain against a trusted root CA store prior to establishing the tunnel.
* **Certificate Pinning:** Implement public key or certificate pinning for all communications between the physical device and the backend APIs. The device should be configured to explicitly reject arbitrary, self-signed, or dynamically generated proxy certificates.



```
