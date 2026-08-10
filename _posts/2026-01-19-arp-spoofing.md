---
title: "ARP Spoofing: Validating a MITM Scenario in a Lab"
date: 2026-01-19
layout: single
lang: en
translation_key: arp-spoofing
categories: [Labs]
tags: [ARP, Spoofing, MITM, Wireshark]
excerpt: "ARP spoofing lab in an isolated network, validating a man-in-the-middle position through ARP tables, tcpdump, and Wireshark."
permalink: /labs/arp-spoofing/
header:
  teaser: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image_description: "Network cables connected to switches."
  image_height: 300px
author_profile: true
author: julia_en
---

<em>
<strong>Important:</strong> this test was performed exclusively in a private and controlled lab network.
</em>

### Project summary

In this lab, I tested how manipulating **IP-to-MAC associations** through ARP spoofing can change the traffic path inside a local network.

The environment was built in **Proxmox**, and the man-in-the-middle position was validated using:

- ARP tables;
- `tcpdump`;
- Wireshark.

> This lab establishes the MITM position only. A separate project uses that position to test DNS spoofing.
>
> [DNS Spoofing via ARP-Based MITM](/labs/dns-spoofing/){: target="\_blank" rel="noopener noreferrer" }

### Lab environment

The three systems involved in the test were connected to the same virtual bridge and subnet, allowing direct Layer 2 communication.

| System               | Role                       |
| -------------------- | -------------------------- |
| **WINDOWS10_VICTIM** | Windows 10 client          |
| **KALI_ATTACKER**    | Host used for ARP spoofing |
| **UBUNTU_SERVER**    | Legitimate server          |

A `DNS_SERVER` was also part of the environment for DNS resolution testing.

All systems were hosted in a private and isolated lab network. Real addresses were replaced with identifiers in the post.

### Baseline

Before the test, I validated that communication between `WINDOWS10_VICTIM` and `UBUNTU_SERVER` occurred directly and that the ARP mappings pointed to the legitimate MAC addresses.

```text
WINDOWS10_VICTIM ←→ UBUNTU_SERVER
```

I also confirmed that `ubuntu.lab` resolved to the correct server address.

![Baseline — ubuntu.lab resolution from Windows](/assets/images/arp-spoofing/BASE_A_WINDOWS_NSLOOKUP.png)

This state was used as a reference for comparing the evidence collected after the ARP tables were modified.

### Controlled ARP spoofing execution

To alter the traffic path, I used `arpspoof` on the Kali system against both endpoints.

The goal was to modify the ARP mappings in both directions:

```text
WINDOWS10_VICTIM
Server IP → Kali MAC

UBUNTU_SERVER
Victim IP → Kali MAC
```

On Kali, I ran one `arpspoof` instance for each endpoint:

```bash
sudo arpspoof -i eth0 -t WINDOWS10_VICTIM_IP UBUNTU_SERVER_IP
```

```bash
sudo arpspoof -i eth0 -t UBUNTU_SERVER_IP WINDOWS10_VICTIM_IP
```

The first process caused the Windows client to associate the server IP with Kali's MAC address. The second caused the Ubuntu server to associate the victim IP with the same Kali MAC address.

With both mappings changed, the traffic path changed from:

```text
WINDOWS10_VICTIM ←→ UBUNTU_SERVER
```

to:

```text
WINDOWS10_VICTIM
        ↓
KALI_ATTACKER
        ↓
UBUNTU_SERVER
```

The IP endpoints remained unchanged; the modification occurred in Layer 2 forwarding.

### Validation through ARP tables

The first evidence came from the ARP tables on both endpoints.

On Ubuntu, the Windows system's IP address became associated with Kali's MAC address.

![ARP table on Ubuntu after the change](/assets/images/arp-spoofing/BASE_B_UBUNTU_ARP_2.png)

On Windows, the Ubuntu server IP became associated with the MAC address of `KALI_ATTACKER`.

![ARP table on Windows after the change](/assets/images/arp-spoofing/BASE_B_ARP_WINDOWS.png)

Together, the two tables showed:

```text
Windows: Server IP → Kali MAC
Ubuntu:  Victim IP → Kali MAC
```

These mappings showed that both endpoints were sending frames to Kali that would normally have been delivered directly to each other.

### Interception validation

After modifying the ARP tables, I observed the traffic on Kali's network interface with `tcpdump`.

The capture showed:

- an HTTP request sent by `WINDOWS10_VICTIM`;
- the HTTP `200` response from `UBUNTU_SERVER`;
- the packets traversing the interface of `KALI_ATTACKER`.

![Traffic observed on Kali with tcpdump](/assets/images/arp-spoofing/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)

The IP addresses still identified the victim and server as the communication endpoints. The change was in the Layer 2 path:

```text
IP:  WINDOWS10_VICTIM → UBUNTU_SERVER
MAC: WINDOWS10_VICTIM → KALI_ATTACKER → UBUNTU_SERVER
```

This confirmed that Kali was participating in the traffic path rather than only observing broadcast traffic on the network.

### Conclusion

The lab showed how changes to **IP-to-MAC mappings** can modify the Layer 2 traffic path without changing the IP endpoints of the communication.

The MITM position was validated through two main pieces of evidence:

- the endpoints' ARP tables associated the legitimate IP addresses with Kali's MAC address;
- traffic between the victim and server was observed traversing the intermediate Kali interface.

From a defensive perspective, unexpected IP-to-MAC changes, multiple IP addresses mapping to the same MAC, and traffic traversing an unexpected host are signals that can be correlated with ARP tables, packet captures, and switch telemetry.

Controls such as **DHCP snooping**, **Dynamic ARP Inspection (DAI)**, and network segmentation can help prevent or limit this behavior, while encrypted protocols such as **HTTPS and SSH** reduce the impact if traffic is intercepted.
