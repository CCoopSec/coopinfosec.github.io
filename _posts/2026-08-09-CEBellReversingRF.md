---
layout: post
title: Reverse Engineering and Exploiting the Cloudedge Bell 24T Sub-GHz RF Signals
subtitle: Taking Advantage of Unencrypted Static ASK/OOK Signals to Bypass Authentication and Replay/Jam Indoor Chime Triggers Using SDR.
gh-repo: chezzuhhh.github.io
comments: true
mathjax: true
author: Chase Cooper
---

## Overview

Following up on my previous vulnerability analysis on the network communication stack of the CloudEdge Bell 24T Smart Doorbell, I shifted my focus to the sub-GHz radio frequency (RF) communications between the smart doorbell unit and its paired indoor chime.

The control and data planes for network communication suffered from severe cryptographic failures in their API implementation, and the RF signaling mechanism was no different lacking cryptographic implementation entirely. By capturing and analyzing the raw transmission payloads, I discovered that the doorbell relies on completely static RF identifiers. This failure to implement rolling codes or cryptographic implementation (such as challenge-response protocols) allows an attacker to execute an authentication bypass by capture-replay, as well as targeted RF jamming. This completely compromises the availability and intended functionality of the device without requiring physical access.

Here is a technical breakdown of how I captured, reverse-engineered, and successfully replayed the signal/payload.  

## Signal Capture & Observation

To establish a baseline, I needed to capture the raw RF transmissions generated when the doorbell button is physically pressed. I connected an RTL-SDR Blog v4 to SDR# to monitor the spectrum and isolate the operating frequencies. Doing so resulted in discovering that doorbell transmits the same payload across both the 433 MHz and 868 MHz bands.
  
![[Pasted image 20260805153408.png]]
_Live Fast Fourier Transform (FFT) view of 433MHz (left) & 868MHz (right) signals sent from the smart doorbell. Analyzed using SDR# + RTL-SDR._

After capturing the raw I/Q data during a legitimate button press, I imported the file into Universal Radio Hacker (URH) and Inspectrum. This allowed me to visually break down the modulation and line encoding techniques utilized:

By observing the time-domain waveform (using URH) and frequency-domain (using Spectrogram), it became clear the device utilizes Amplitude Shift Keying (ASK), specifically On-Off Keying (OOK). The data is transmitted using Pulse Width Modulation (PWM) with a symbol time of 1 ms, resulting in a 1000 Baud rate (1/0.001 s).

![[Pasted image 20260805153916.png]]
_Deeper look into intercepted smart doorbell RF signal using URH (top), and Inspectrum (bottom)_

By decoding the waveform, I was able to map out the complete packet anatomy: the preamble, sync word, static payload, and tail. Because the payload remained static between button presses, the system is fundamentally vulnerable to replay attacks.

**NON RETURN TO ZERO DECODING**

![[Pasted image 20260619130215.png]]
![[Pasted image 20260619130235.png]]
_Detailed look into the static payload across 5 transmissions using URH. Note the complete lack of variance across multiple triggers._

## Exploiting the Replay Attack and Signal Jamming

With the packet anatomy and technical signal transmission parameters identified, I moved to exploit the static payload using a Yard Stick One software-controlled transceiver.

Using this script written around `rfcat`, I configured the Yard Stick One to match the targeted environment parameters:

- **Frequencies:** 433 MHz and 868 MHz
- **Modulation:** Amplitude Shift Keying (ASK / OOK)
- **Line Encoding:** Pulse Width Modulation (PWM)
- **Symbol Time:** 1 ms
- **Symbol (Baud) Rate:** 1000 Baud ($1 / 0.001\text{ s}$)



By passing the previously captured packet structure into `rfcat`, I successfully crafted and transmitted a synthetic signal that perfectly replicated the doorbell's original transmission. The indoor chime activated immediately. Because the receiver does not track state or validate signal freshness, it blindly accepted the replayed payload as a legitimate, authenticated button press.

  
## Impact & Threat Model

By exploiting this lack of signal variance, an attacker can completely compromise the availability and intended function of the doorbell's physical alerting mechanism without requiring any physical interaction with either device.

- **Unauthorized Initiation:** Replaying the static payload allows an attacker to trigger the chime repeatedly at any time, causing a nuisance or creating false alerts inside the home.
- **Denial of Service (Jamming):** By continuously transmitting a continuous carrier wave or flooding the 433/868 MHz operational frequencies with the static payload, an attacker can completely deafen the chime receiver. This prevents legitimate doorbell presses from triggering the chime, effectively masking the arrival of visitors or malicious actors.

## Conclusion & Mitigations

This segment of the research highlights a critical, but common, architectural flaw in consumer IoT hardware: the reliance on static RF identifiers for sub-GHz communication. To resolve this vulnerability, the vendor must update the firmware and hardware logic to implement the following controls:

1. **Implement Rolling Codes:** The communication stack must utilize a rolling code scheme or a cryptographic challenge-response protocol. Each physical doorbell press must generate a mathematically unique, single-use payload.
2. **Time-Windowed Rejection:** The chime receiver must maintain a synchronized state with the transmitter and explicitly reject any previously used identifiers or payloads that fall outside an acceptable temporal window.
