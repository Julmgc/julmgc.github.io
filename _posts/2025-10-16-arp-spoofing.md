---
title: "ARP SPOOFING"
date: 2025-10-16
layout: single
categories: [Labs]
tags: [ARP, Spoofing, MITM, Wireshark]
excerpt: "Demonstration of ARP spoofing (MITM): baseline, attack and mitigations."
permalink: /labs/arp-spoofing/
header:
  teaser: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/arp-spoofing/arp-spoofing-header.jpg
  image_description: "Network cables connected to switches"
  image_height: 300px
author_profile: true
---

<em>
**Important - This technique is for educational purposes only. It was executed in a private lab environment. Performing this on a network without explicit permission is illegal and unethical. Real artifacts are available to recruiters and educators on request.**</em>

## What is ARP Spoofing?

<div class="justify-text">
<p class="indent" style="text-indent: 2rem;">ARP (Address Resolution Protocol) is a network protocol that maps MAC addresses (unique computer or device identifiers) to IP addresses. Networks maintain an ARP table — a database that connects MAC addresses to IP addresses. </p>

<p class="indent" style="text-indent: 2rem;">ARP spoofing happens when an attacker sends forged ARP replies, causing devices to store incorrect IP-to-MAC mappings in their ARP tables. ARP spoofing is commonly used for man-in-the-middle attacks to intercept, read, and/or modify traffic between devices; for session hijacking; for denial-of-service; in combination with DNS poisoning to redirect traffic to fake sites; and to sniff credentials and data.</p>

<p class="indent" style="text-indent: 2rem;">This project demonstrates ARP Spoofing, and talks about how to prevent it. I start by showing normal communication within systems on the same LAN (Baseline A), with packet captures. Then I demonstrate ARP Spoofing (Baseline B) and DNS Poisoning (Baseline C). Finally, I present mitigation techniques to prevent these exploits from happening.</p>

</div>
<p style="text-align:center"><em>This lab was done using Proxmox, I set up four virtual machines:</em></p>

<h4>OPSEC legend (labels replace real addresses):</h4>

<ul class="opsec-list">
  <li><span class="label"><strong>WINDOWS10_VICTIM</strong></span><span class="arrow">→</span><span class="desc">Windows 10 test target</span></li>
  <li><span class="label"><strong>KALI_ATTACKER</strong></span><span class="arrow">→</span><span class="desc">Kali attacker</span></li>
  <li><span class="label"><strong>DNS_SERVER</strong></span><span class="arrow">→</span><span class="desc">Raspberry Pi (local DNS)</span></li>
  <li><span class="label"><strong>UBUNTU_SERVER</strong></span><span class="arrow">→</span><span class="desc">Ubuntu server (file/web services)</span></li>
</ul>

---

## BASELINE A - everything working normally (no ARP spoofing)

<p class="indent" style="text-indent: 2rem;">Windows client, <strong>WINDOWS10_VICTIM</strong> browses the address `ubuntu.lab`, which is hosted on <strong>UBUNTU_SERVER</strong>. The client resolves the name using the local DNS resolver, <strong>DNS_SERVER</strong>, and then connects to the server. The screenshot below shows the DNS response from <strong>DNS_SERVER</strong>, which returns an A record pointing to <strong>UBUNTU_SERVER</strong> on Wireshark. Filter used:</p>

dns.qry.name == "ubuntu.lab"

![Baseline A - WIRESHARK screenshot](/assets/images/arp-spoofing/BASE_A_WIRESHARK_DNS.png)

<p class="indent" style="text-indent: 2rem;">When <strong>WINDOWS10_VICTIM</strong> runs `nslookup ubuntu.lab`, the DNS query is handled by <strong>DNS_SERVER</strong>, which replies with the legitimate <strong>UBUNTU_SERVER</strong> IP address.</p>

![Baseline A — nslookup ubuntu.lab](/assets/images/arp-spoofing/BASE_A_WINDOWS_NSLOOKUP.png)

---

## BASELINE B — ARP SPOOFING (Man-in-the-Middle lab)

**What is the actual damage an attacker can do once inside a network with ARP spoofing?**

![Baseline B — MITM topology](/assets/images/arp-spoofing/FINAL_BASE_B_SYSTEMS.png)
ARP spoofing (MITM): **KALI_ATTACKER** poisons both endpoints’ ARP caches so traffic between **WINDOWS10_VICTIM** and **UBUNTU_SERVER** flows through the attacker.

<p class="indent" style="text-indent: 2rem;">All virtual machines are connected on the same virtual network bridge within Proxmox. During this phase, ARP spoofing was performed so that both <strong>WINDOWS10_VICTIM</strong> and <strong>UBUNTU_SERVER</strong> resolved each other’s MAC address to the <strong>KALI_ATTACKER</strong> interface. As a result, all network traffic between the two legitimate hosts was transparently relayed through <strong>KALI_ATTACKER</strong>, creating a MITM (Man-in-the-Middle) scenario.</p>

<p class="indent" style="text-indent: 2rem;"> To execute ARP spoofing on <strong>KALI_ATTACKER</strong> and poison the LAN, we use the tool <code>arpspoof</code> from the <code>dsniff</code> package. It is simple and effective.</p>

Open two separate terminal windows in Kali.

- **Terminal 1: Poison the Windows VM's ARP table.** Tell Windows that the Ubuntu server’s IP is at Kali’s MAC address.

```bash
arpspoof -i eth0 -t WINDOWS10_VICTIM_IP UBUNTU_SERVER_IP
```

-i eth0: Your Kali's network interface (use ip a to find it, it might be ens18 or similar).

-t <strong>WINDOWS10_VICTIM_IP</strong>: The target (victim) IP, the Windows machine.

<strong>UBUNTU_SERVER_IP</strong>: The host's IP you want to impersonate to the target (the server).

- **Terminal 2: Poison the Ubuntu Server's ARP table.** Tell the Ubuntu Server that the Windows IP is at Kali's MAC address.

```bash
arpspoof -i eth0 -t UBUNTU_SERVER_IP WINDOWS10_VICTIM_IP
```

<p class="indent" style="text-indent: 2rem;">At this point, the MITM is active. Traffic from Windows to Ubuntu, and vice versa, is now flowing through your Kali machine. You can verify this by looking at the ARP table on the Windows machine (arp -a in cmd) – you should see the Ubuntu Server's IP address mapped to Kali's MAC address.</p>

![Baseline B — ARP UBUNTU](/assets/images/arp-spoofing/BASE_B_UBUNTU_ARP_2.png)

<p class="indent" style="text-indent: 2rem;">The <strong>WINDOWS10_VICTIM_IP</strong> ARP table demonstrates the same poisoning observed on the server: both the server's and attacker's IPs are associated with the <strong>KALI_ATTACKER MAC address</strong> in the victim's ARP cache. This confirms that <strong>WINDOWS10_VICTIM</strong> has been tricked into sending frames for the server to the attacker. </p>

![Baseline B — ARP WINDOWS](/assets/images/arp-spoofing/BASE_B_ARP_WINDOWS.png)

<p class="indent" style="text-indent: 2rem;">Now we can observe the <strong>WINDOWS10_VICTIM’s</strong> HTTP GET and the server’s HTTP/200 response being forwarded with tcpdump on the <strong>KALI_ATTACKER VM.</strong> These packets demonstrate that the attacker is transparently relaying traffic between the victim and server. .</p>

<p class="indent" style="text-indent: 2rem;">Note that the IP headers show src=<strong>WINDOWS10_VICTIM</strong> and dst=<strong>UBUNTU_SERVER</strong>, but the Ethernet frames are received by the attacker. This confirms that traffic is being intercepted and relayed (MITM) while preserving the original IP/TCP endpoints.</p>

![Baseline B — TCPDUMP KALI](/assets/images/arp-spoofing/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)

<p class="indent" style="text-indent: 2rem;">The Wireshark capture on <strong>KALI_ATTACKER</strong> also shows an ICMP redirect captured on the attacker. ICMP redirect messages indicate routing/forwarding changes. In this context, demonstrate additional network-layer side effects of the poisoning — e.g., hosts receiving route notices or the attacker's role in modifying traffic paths. Use this capture to explain how the network perceives the changed link-layer topology.</p>

![Baseline B — WIRESHARK WINDOWS](/assets/images/arp-spoofing/BASE_B_WIN10_WIRESHARK_ARP.png)

## 🛡️ How to Prevent ARP Spoofing

- Enable **DHCP snooping** on switches.
- Use **ARP monitoring tools** such as _Arpwatch_ or _ARPScan_.
- Configure **static ARP entries** for critical systems.

### Defense Strategies

ARP spoofing can be mitigated using a combination of host-level, switch-level, and network-layer protections:

- **Static ARP Entries** — Pin IP→MAC mappings for critical hosts (such as gateways and important servers) to prevent unauthorized changes.

- **Host Hardening (Linux)** — Use tools like **ArpON** or kernel-level filtering with **arptables** to validate and block suspicious ARP activity.

- **Windows Protection** — On Windows endpoints, tools like **XArp** can detect, block, and alert on spoofing attempts.

- **Network Monitoring** — Run an ARP monitoring or notification service (for example, _Arpwatch_ or _Arpalert_) to detect unexpected IP↔MAC mapping changes.

- **Switch-Level Security** — Enable **Dynamic ARP Inspection (DAI)** together with **DHCP snooping** and **port security** on managed switches to automatically drop forged ARP frames at Layer 2.

- **Network IDS/IPS** — Deploy intrusion detection or prevention systems such as **Snort** or **Suricata** to identify abnormal ARP traffic patterns and forward alerts to your **SIEM** or logging system.

- **Encryption & VPNs** — Use **VPNs** and **end-to-end encryption (TLS/HTTPS)** on untrusted networks.
  > While these do not stop ARP spoofing itself, they protect data confidentiality and integrity — making it much harder for an attacker to harvest useful information even if they intercept traffic.

---
