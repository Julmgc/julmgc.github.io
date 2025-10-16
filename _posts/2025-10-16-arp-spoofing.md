---
title: "ARP SPOOFING"
date: 2025-10-16
categories: [Labs]
tags: [ARP, Spoofing, MITM, Wireshark]
layout: post
excerpt: "Demonstration of ARP spoofing (MITM): baseline, attack and mitigations."
header:
  teaser: /assets/images/arp-spoofing-header.jpg
---

**Important - This technique is for educational purposes only. It was executed in a private lab environment. Performing this on a network without explicit permission is illegal and unethical. Real artifacts are available to recruiters and educators on request.**

## What is ARP Spoofing?

ARP (Address Resolution Protocol) is a network protocol that maps MAC addresses (unique computer or device identifiers) to IP addresses. Networks maintain an ARP table — a database that connects MAC addresses to IP addresses. ARP spoofing happens when an attacker sends forged ARP replies, causing devices to store incorrect IP-to-MAC mappings in their ARP tables. ARP Spoofing is used for man-in-the-middle attacks, which is used to intercept, read and or modify traffic between devices; session hijacking; denial of service; combined with DNS poisoning to redirect traffic to fake sites; sniffing credentials and data.

This project demonstrates ARP Spoofing, and talks about how to prevent it. I start by showing normal communication within systems on the same LAN (Baseline A), with packet captures. Then I demonstrate ARP Spoofing (Baseline B) and DNS Poisoning (Baseline C). Finally, I present mitigation techniques to prevent these exploits from happening.

This lab was done using Proxmox, I set up four virtual machines:

### OPSEC legend (labels replace real addresses):

WINDOWS10_VICTIM → Windows 10 test target
KALI_ATTACKER → Kali attacker
DNS_SERVER → Raspberry Pi (local DNS)
UBUNTU_SERVER → Ubuntu server (file/web services)

---

## BASELINE A - everything working normally (no ARP spoofing)

Windows client, **WINDOWS10_VICTIM** browses the address `ubuntu.lab`, which is hosted on **UBUNTU_SERVER**. The client resolves the name using the local DNS resolver, **DNS_SERVER**, and then connects to the server. The screenshot below shows the DNS response from **DNS_SERVER**, which returns an A record pointing to **UBUNTU_SERVER** on Wireshark. Filter used:

dns.qry.name == "ubuntu.lab"

![Baseline A - WIRESHARK screenshot](/assets/images/BASE_A_WIRESHARK_DNS_1.png)

When **WINDOWS10_VICTIM** runs `nslookup ubuntu.lab`, the DNS query is handled by **DNS_SERVER**, which replies with the legitimate **UBUNTU_SERVER** IP address.

![Baseline A — nslookup ubuntu.lab](/assets/images/BASE_A_ns_look_up_w_to_ubuntulab.png)

---

## BASELINE B — ARP SPOOFING (Man-in-the-Middle lab)

**What is the actual damage an attacker can do once inside a network with ARP spoofing?**

![Baseline B — MITM topology](/assets/images/FINAL_BASE_B_SYSTEMS.png)
ARP spoofing (MITM): **KALI_ATTACKER** poisons both endpoints’ ARP caches so traffic between **WINDOWS10_VICTIM** and **UBUNTU_SERVER** flows through the attacker.

All virtual machines are connected on the same virtual network bridge within Proxmox. During this phase, ARP spoofing was performed so that both **WINDOWS10_VICTIM** and **UBUNTU_SERVER** resolved each other’s MAC address to the **KALI_ATTACKER** interface. As a result, all network traffic between the two legitimate hosts was transparently relayed through **KALI_ATTACKER**, creating a MITM (Man-in-the-Middle) scenario.

In this baseline demonstration ARP spoofing was performed, this chain of evidence proves a successful Man-in-the-Middle. First, **KALI_ATTACKER** poisons the **WINDOWS10_VICTIM** arp table and the **UBUNTU_SERVER** arp table using `arpspoof`. Then, the ARP table on **UBUNTU_SERVER** and the **WINDOWS10_VICTIM** confirms the poisoning, mapping peer IPs to the attacker's MAC. Next, the attacker’s `tcpdump` capture shows the victim’s HTTP GET request and the server’s HTTP/200 response being relayed through the attacker. Finally, a Wireshark capture reveals an ICMP Redirect message observed during the attack, illustrating how the network responds to the altered topology.

Execute arpspoof on **KALI_ATTACKER**, poisoning the LAN, we will use the tool `arpspoof` from the `dsniff` package. It's simple and effective.

Open two separate terminal windows in Kali.

- **Terminal 1: Poison the Windows VM's ARP table.** Tell Windows that the Ubuntu Server's IPs at Kali's MAC address.

```bash
arpspoof -i eth0 -t WINDOWS10_VICTIM_IP UBUNTU_SERVER_IP

-i eth0: Your Kali's network interface (use ip a to find it, it might be ens18 or similar).

-t WINDOWS10_VICTIM_IP: The target (victim) IP, the Windows machine.

UBUNTU_SERVER_IP: The host's IP you want to impersonate to the target (the server).

Terminal 2: Poison the Ubuntu Server's ARP table. Tell the Ubuntu Server that the Windows IP is at Kali's MAC address.

arpspoof -i eth0 -t UBUNTU_SERVER_IP WINDOWS10_VICTIM_IP

At this point, the MITM is active. Traffic from Windows to Ubuntu, and vice versa, is now flowing through your Kali machine. You can verify this by looking at the ARP table on the Windows machine (arp -a in cmd) – you should see the Ubuntu Server's IP address mapped to Kali's MAC address.

This screenshot of the UBUNTU_SERVER ARP table cache proves the poisoning: the IP address that should map the WINDOWS10_VICTIM's IP address is instead associated with the KALI_ATTACKER MAC address.


![Baseline B — ARP UBUNTU](/assets/images/BASE_B_UBUNTU_ARP.png)

The WINDOWS10_VICTIM ARP table demonstrates the same poisoning observed on the server: both the server's and attacker's IPs are associated with the KALI_ATTACKER MAC address in the victim's ARP cache. This confirms that WINDOWS10_VICTIM has been tricked into sending frames for the server to the attacker. When combined with the server-side ARP evidence, these symmetric ARP cache entries prove that the attacker sits in the middle for both directions of the conversation.


![Baseline B — ARP WINDOWS](/assets/images/BASE_B_ARP_WINDOWS.png)

Now we can observe the victim’s HTTP GET and the server’s HTTP/200 response being forwarded with tcpdump on the KALI_ATTACKER VM.


![Baseline B — TCPDUMP KALI](/assets/images/BASE_B_KALI_ATTACKER_TCP_DUMP_ARP_SPOOFING.png)


These packets demonstrate that the attacker is transparently relaying traffic between the victim and server. Captured by the attacker, tcpdump displays the victim’s HTTP GET requests and the server’s HTTP 200 responses. Note that the IP headers show src=WINDOWS10_VICTIM and dst=UBUNTU_SERVER, but the Ethernet frames are received by the attacker. This confirms that traffic is being intercepted and relayed (MITM) while preserving the original IP/TCP endpoints.

The Wireshark capture on KALI_ATTACKER also shows an ICMP redirect (or related control message) captured on the attacker. ICMP redirect messages indicate routing/forwarding changes and, in this context, demonstrate additional network-layer side effects of the poisoning — e.g., hosts receiving route notices or the attacker's role in modifying traffic paths. Use this capture to explain how the network perceives the changed link-layer topology.


![Baseline B — WIRESHARK WINDOWS](/assets/images/BASE_B_WIN10_WIRESHARK_ARP.png)

---
```
