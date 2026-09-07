---
layout: post
title: Hardware Hacking the Merkury MI-CW051 IP Camera
subtitle: Reverse Engineering the PCB, Accessing Exposed Debug Pads, Bypassing Bootloaders, Chip-Off Firmware Extraction, Static Analysis in Ghidra, and Uncovering Cryptographic Vulnerabilities.
gh-repo: chezzuhhh.github.io
comments: true
mathjax: true
author: Chase Cooper
---

My goal here was to understand the camera's hardware and software architectures, access exposed serial/debug pads, extract the firmware, and see exactly what kind of data it leaves sitting in flash memory. Here is a step-by-step breakdown of how I gained a root shell through serial console access, as well as the chip-off extraction process and the secrets buried inside the firmware.

- **Manufacturer:** Merkury Innovations
- **Model:** MI-CW051-199W-A
- **Device ID:** pve-4c37debf794a
- **Firmware Version:** 3.0.0.086

<img width="971" height="928" alt="image" src="https://github.com/user-attachments/assets/e6b4c63c-f8a3-4986-bf20-75475cc45c63" />

## Hardware Reconnaissance
The first thing I needed to do was understand what components I was dealing with. I began by identifying the primary integrated circuits (ICs) and pulling their datasheets.

The CPU is an AK3918 HD IP Camera SoC (ARM architecture) with support for debugging and serial protocols (such as UART and JTAG). Locating these serial interfaces on the board was my first priority. I ran voltage readings across all of the exposed pads to map the board. After ruling out several pads all over the surface of the board, I discovered this set of 4 UART candidates:
<img width="643" height="345" alt="image" src="https://github.com/user-attachments/assets/94e68f29-cda1-4f9b-b03d-b5d0f6ba0f67" />
- **Far Left (VCC):** 3.3V
- **Middle Left (GND):** 0V
- **Middle Right (TX):** Fluctuates from 0V to 1.8V
- **Far Right (RX):** 0V

## UART Access
Connecting a Bus Pirate 5 to the enumerated UART pins was my next step in mapping out the hardware.

<img width="965" height="903" alt="image" src="https://github.com/user-attachments/assets/7f084316-9c1e-468c-a752-a11e4abc2afb" />

Following a successful boot, the operating system locks down the console and does not provide a shell, so my focus then altered to the boot sequence itself.

## U-Boot Interruption
After taking a closer look at the bootloader environment variables, it exposed a configuration of `bootdelay=1`. This config is how I exploited the bootloader, it leaves a one-second window to manually intercept the startup process. Since access to the U-Boot menu was not disabled (as it could be by setting `bootdelay=-1`), I just had to disconnect and reconnect power to activate the startup sequence again. This time I was able to send a keystroke during the 1-second boot delay window to halt the autoboot sequence and drop straight into the U-Boot menu console. 

From here, I modified the `bootargs` environment variable to alter the kernel's initialization parameters. Replacing `init=/sbin/init` with `init=/bin/sh` forced the kernel to bypass its normal startup scripts post-boot and drop me directly into a root shell instead.

_(Note: In scenarios where you are provided with a password-protected shell post-boot, and access to the boot menu hasn't been disabled, this exact method allows you to bypass authentication entirely just by tweaking those initialization arguments.)_

## Fault Injection through Data-Line Glitching
I wanted to test the hardware's resilience and see what would happen if the vendor had explicitly disabled U-Boot console access. To do this I performed a timing-based fault injection attack.

Because U-Boot is stored directly on the SPI flash, the initial Boot ROM phase must complete uninterrupted to copy the bootloader into volatile memory (RAM). I introduced a physical short (by grounding the Data Out (DO) pin of the SPI flash chip to Ground (GND)) immediately after U-Boot was established and began executing from RAM, but before it initiated its secondary phase of reading the OS firmware from the flash chip.

This prevented the SoC from correctly reading the firmware payload. The forced failure triggered a read error on the serial output and subsequently dropped the UART connection directly into the U-Boot bootloader menu. From there, I could take the exact same steps as before to adjust the `bootargs` environment variable for shell access post-boot.

While obtaining a live root shell gave me the capability to dump the MTD blocks over the network or onto an SD card, I wanted to take the route of a physical chip-off extraction. Desoldering the flash chip and reading it directly using a flash programmer ensures the cleanest extraction of the firmware image by removing the risk of accidentally sending power to the rest of the board (which happened many times when trying an on-chip firmware extraction using SOIC8 test clips and a Tigard board running `flashrom`).

## Flash Memory Analysis

**Package:** ZBIT ZB25VQ64 (8MB SPI NOR Flash)

This is where the firmware lives. The datasheet confirmed standard SPI read instructions, meaning the firmware could be easily dumped once the chip was removed.

<img width="617" height="402" alt="image" src="https://github.com/user-attachments/assets/f4537e8b-a03d-43eb-92b3-04614164ddd6" />
This illustrates the standard 8-pin package layout by highlighting the SPI communication lines: Chip Select (CS#), Clock (CLK), Data In (DI), and Data Out (DO), alongside power delivery (VCC, GND) and hardware control pins (WP#, HOLD#/RESET#).

<img width="967" height="292" alt="image" src="https://github.com/user-attachments/assets/03495c2d-4738-40e3-aa00-55a6250d831e" />
This outlines the read data instruction used by the flash chip. The master device drives the chip select line low, then transmits the standard `03h` read opcode followed by a 24-bit address over the slave-in line. In response, the flash memory sends out the requested 8-bit data chunks over the slave-out line, which is the sequence used to dump the firmware.

Before resorting to a full chip-off extraction, I attached a BitMagic logic analyzer to the flash chip's pins while it was still on the PCB to see the boot sequence and verify the read data instruction as raw signals.
<img width="967" height="916" alt="image" src="https://github.com/user-attachments/assets/a6c86abb-7346-4647-9025-eef5cfe19148" />

Using the logic analyzer sampling at 24 MHz (slight overkill), the boot sequence was captured. Chip select drops low to initiate the transfer, the processor then transmits the `03` opcode followed by the 24-bit address (`00 00 00`) over the slave-in line. The SPI flash/EEPROM decoder translated this transaction into a `READ` command. This allowed for visualization of the communications used by the flash chip and CPU upon boot, which further supported that a valid dump should occur once the chip was desoldered.
<img width="967" height="515" alt="image" src="https://github.com/user-attachments/assets/e208f806-0956-45ea-adc7-faa5358f8661" />

## Chip-Off Extraction
I desoldered the ZB25VQ64 flash chip from the PCB and seated it into an XGecu T48 universal flash programmer. By providing the exact chip parameters derived from the datasheets into the XGpro software, I was able to read the memory contents and dump the raw 8MB binary as `FW.bin`.
<img width="967" height="475" alt="image" src="https://github.com/user-attachments/assets/36c5dc4b-2999-43a0-a22c-deae494765fc" />

## Initial Binary Triage
Before carving filesystems, I ran `strings` on the raw `FW.bin` file. This immediately yielded a massive amount of data regarding the device tree and boot parameters:
- **The Serial Console:** `bootargs=console=ttySAK0,115200n8`. This confirmed the hardware interface (`ttySAK0`) and the `115200` baud rate needed for the Bus Pirate.
- **Hardware Architecture:** `ANYKAH3B, anyka,ak39ev330, cpus cpu@0 arm, arm926ej-s`. This confirms the ARM926EJ-S architecture, which was important context for decompiling binaries later.
- **TFTP Network Recovery Mechanism:** `ipaddr=192.168.1.99, serverip=192.168.1.1, update=tftp $(loadaddr) $(image_name)`. The bootloader contains a hardcoded fallback mechanism to update its firmware over the network. If triggered, the camera looks for a TFTP server hosted at `192.168.1.1` to pull down a file named `uImage`. _This TFTP mechanism could absolutely be weaponized to force the bootloader to swallow a maliciously crafted firmware image._

## Carving the Filesystems
I then used `binwalk` to parse the extracted binary. It successfully carved the U-Boot boot environment, two read-only SquashFS filesystems, and two read/write JFFS2 partitions.
<img width="971" height="408" alt="image" src="https://github.com/user-attachments/assets/a2d2cff8-a627-41c7-aea7-4615a9499744" />

After running `binwalk -e FW.bin`, I organized the extracted contents by partition to understand the operating system, vendor applications, and user data.

### Read-Only Partitions (SquashFS)
These partitions contain the core of the camera's software stack.
- **Base OS (`1C3000.squashfs`):** This partition holds the foundational Linux environment. Inside `/etc/init.d/`, startup scripts dictate the boot order. The `/lib/modules/` directory houses the core Wi-Fi drivers and cryptographic kernel modules required for secure connections.
- **The Camera's Brain (`3DA000.squashfs`):** This partition contains all vendor-specific logic. It includes the Anyka hardware drivers, local networking binaries, and the main application payload at `/pepper/pepper_app`. 
	- This `pepper_app` binary is a powerhouse, handling RTSP media streaming, MP4 recording to the SD card, AWS WebRTC integration, OTA updates, and MQTT messaging.

**Static Analysis (`/pepper/pepper_app`)**
To reverse-engineer the operational logic of the custom executables, static analysis with Ghidra was implemented. Because I found `arm926ej-s` in the strings earlier, I knew to select the **ARM v5T (little-endian)** architecture to ensure the decompiler handled the instruction set correctly.

This file was compiled as a statically linked, stripped ELF binary lacking standard debugging symbols. Deploying Ghidra allowed me to manually reconstruct targeted control flows and identify proprietary authentication routines. Reversing these custom network protocol serialization methods uncovered hardcoded RSA keys and certificates buried inside the binary.

### Writable Partitions (JFFS2)
While the SquashFS partitions hold the read-only firmware, the writable JFFS2 partitions (`3B3000.jffs2` and `3CFA7C.jffs2`) are used to store dynamic configuration data. Inspection of these filesystems exposed highly sensitive plaintext data.

Inside `jffs2-root`, I found a `config` file containing a JSON-formatted dictionary mapping numerical keys to the device's operational settings, which held severe security implications:
- **Wi-Fi Credentials:** Stored in plaintext. Key 132 held the SSID for my home network and Key 133 held the password.
- **Platform Identity:** Key 134 held an X.509 Client Certificate (Valid 2026 - 2036). The issuer is "Pepper IoT prod CA" with a Subject Organization of "Smart Home Ventures, LLC".
<img width="971" height="140" alt="image" src="https://github.com/user-attachments/assets/f9c56b8a-38bc-4e5c-bddf-00a6db7ed349" />
- **Device UUIDs:** Keys 138 and 139 contained unique identifiers. The X.509 certificate's Common Name (CN) combines these exact UUIDs (`stream|634e...|adb07...`) to authenticate the hardware.
- **Cryptographic Material:** Key 135 held a 1024-bit RSA Private Key.
<img width="971" height="140" alt="image" src="https://github.com/user-attachments/assets/5e546938-41ed-433d-8f4c-abcb6ca77af1" />
- The remaining keys contain non-sensitive operational data (e.g., timezone configurations and default sensor thresholds) that provide no value for my purposes here.

I also found a `config.128` file containing a single string: `pve-4c37debf794a`. This is the device's hostname, which translates directly to the camera's physical MAC address: `4C:37:DE:BF:79:4A`.

### Impact
The device uses these keys for mutual TLS (mTLS) authentication to securely connect to the Pepper IoT cloud servers. By extracting the private key, an attacker can completely bypass the device's certificate pinning defenses. On a local scale, this allows you to impersonate this specific camera on the manufacturer's network, or set up a man-in-the-middle proxy to intercept and decrypt its MQTT and HTTPS traffic.

However, the bigger risk here lies in the potential blast radius across the ecosystem. The device's platform identity and X.509 certificate Common Name (CN) are directly tied to the UUIDs found in Keys 138 and 139. If these UUIDs are generated sequentially or using a predictable algorithm (or if an attacker obtains target UUIDs using other methods) an attacker with this extracted cryptographic material could theoretically forge authentication requests for other users' cameras, potentially intercepting broader telemetry or pivoting into the backend cloud infrastructure.

## Conclusion & Mitigations
The Merkury MI-CW051 is a great representation of the average consumer IoT architecture. By mapping the hardware, dumping the flash memory, and decompiling stripped binaries in Ghidra, it was clear to see exactly how the hardware and software interact.

This unrestricted access allows for total compromise of the host operating system. The discovery of hardcoded RSA private keys, plaintext Wi-Fi credentials, and a highly vulnerable TFTP fallback mechanism highlights the ongoing risks of physical device compromise in the smart home ecosystem.

**Next Steps:** My next objective for this project is to write a reverse shell script, inject it into the startup sequence within `/etc/init.d/`, repack the SquashFS filesystems, and flash the modified firmware back onto the chip to achieve persistent remote access.

Countermeasures can be implemented across the hardware, firmware, and physical enclosure layers. However, when designing consumer IoT devices like the Merkury MI-CW051, manufacturers face very thin profit margins. While physical tamper switches and epoxy potting provide excellent security, they are often financially unviable for a $30 camera. Therefore, implementing software-based controls and low-cost hardware mitigations are the most realistic industry solutions.

Here are strategies for protecting flash memory in low-cost consumer devices:

### Cryptographic Defenses
- **Encrypted Storage Partitions:** Devices should implement encrypted storage partitions to protect against direct physical memory manipulation and chip-off extractions. Encrypting the filesystem at rest prevents the offline extraction of sensitive cryptographic materials.
- **Hardware-Backed Secure Boot:** Implementing a hardware-backed Secure Boot chain establishes a Root of Trust. By cryptographically verifying the digital signature of the bootloader and kernel before execution, this ensures that unverified firmware cannot be flashed back onto non-volatile memory components.

### Hardware & Interface Hardening
- **Disablement of Debug Ports:** Manufacturers must disable or physically obscure UART interfaces in production firmware and hardware builds. Busses should be permanently burned or blocked at the silicon level to prevent bootloader interruption and raw console access.
- **Write-Protect (WP) Pins:** Many SPI flash chips feature a physical Write Protect pin. Grounding or pulling this pin high (depending on the chip's logic) at the hardware level, combined with setting the corresponding Status Register bits, can lock the memory blocks to prevent malicious firmware overwrites with basically zero cost.
- **Component Packaging:** Shifting from exposed-pin packages (like the standard 8-pin SOIC) to Ball Grid Array (BGA) packages hides the solder joints completely beneath the chip. This severely complicates multimeter probing, logic analysis, and desoldering, raising the bar for a successful chip-off extraction without significantly impacting manufacturing costs.

### Physical Obfuscation (High-Security Contexts)
- **Epoxy Potting:** Encasing the PCB, or specifically the CPU and flash memory cluster, in a hardened opaque epoxy resin provides a physical barrier. While rarely implemented in budget consumer devices due to cost constraints, it makes direct probing pretty much impossible without first employing destructive chemical or thermal decapsulation.

