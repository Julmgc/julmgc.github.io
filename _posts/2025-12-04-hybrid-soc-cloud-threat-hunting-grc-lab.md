---
layout: single
title: "Hybrid SOC, Cloud Threat Hunting & GRC Lab"
subtitle: "From pfSense and Proxmox to AWS GuardDuty, Config and a real risk register"
date: 2025-12-04 10:00:00 -0300
last_modified_at: 2025-12-04 10:00:00 -0300
toc: true
toc_sticky: true
author_profile: true
categories:
  - Labs
tags:
  - soc
  - threat-hunting
  - cloud-security
  - aws
  - grc
  - pfsense
  - proxmox
---

**Hybrid SOC, Cloud Threat Hunting & GRC Lab** — a hands-on environment that combines on-premise infrastructure with AWS services to investigate attacks end to end.
{% comment %}

<!-- This post kicks off a new long-term lab project I’m building to combine:

- my on-prem Proxmox homelab (pfSense, Kali, Ubuntu, Windows)
- AWS (EC2, CloudTrail, GuardDuty, Config, SNS, EventBridge)
- and a real **risk register + compliance mapping** (ISO 27001 / NIST)

The goal is to simulate attacks in a hybrid environment and show not only _how to detect them_, but also how to translate them into risk, controls and governance. -->

<!-- ## 🎯 Goals

This project is designed to prove, with real evidence, that I can:

- Investigate attacks in a **hybrid environment** (on-prem + AWS)
- Correlate events between:
  - pfSense (firewall)
  - Linux (Ubuntu Server)
  - Windows 10 (optional)
  - raspberry-dns (internal DNS, optional)
  - AWS EC2 (CloudWatch + CloudTrail)
  - GuardDuty
  - AWS Config
- Build **threat hunting hypotheses** instead of just staring at logs
- Detect:
  - brute force attacks
  - scans
  - simple post-exploitation
  - exposed services in cloud
  - security compliance drift
- Translate technical events into:
  - risk
  - security controls
  - basic GRC artefacts (risk register, policy, mapping to frameworks)

In other words: this is a **SOC + Threat Hunting + Cloud Security + GRC** project in one. -->

<!-- --- -->
<!--
## Overview -->

{% endcomment %}
This project demonstrates a **Hybrid SOC, Cloud Threat Hunting & GRC Lab** — a hands-on environment that combines on-premise infrastructure with AWS services to investigate attacks end to end.

The goal is to show how a defender can:

- collect and correlate logs from **pfSense, Linux, Windows and AWS**
- detect **scans, brute force attempts and simple post-exploitation**
- identify **misconfigurations in Security Groups** and compliance drift
- translate these technical events into **risk, controls and governance (GRC)**

The lab is divided into three main parts:

1. **On-prem Monitoring & Detection** – pfSense, Kali, Ubuntu Server and optional Windows.
2. **Cloud Detection & Misconfiguration** – AWS VPC + EC2 with CloudTrail, GuardDuty and AWS Config.
3. **GRC Layer** – risk register, compliance mapping (ISO 27001 / NIST) and a 1-page security policy.

All tests are performed using a **Proxmox homelab** on the LAN and a dedicated **AWS VPC** in the cloud, simulating a small company with both on-prem and cloud workloads.

<!-- --- -->

## Lab Setup

### On-prem Environment (Proxmox)

| Role         | Hostname        | Description                                              |
| ------------ | --------------- | -------------------------------------------------------- |
| FIREWALL     | `pfsense`       | Edge firewall and gateway, logs scans and brute force    |
| ATTACKER     | `kali`          | Internal attacker, runs Nmap, Hydra/Ncrack, etc.         |
| LINUX SERVER | `ubuntu-server` | Main internal target, SSH service and system logs        |
| WINDOWS 10\* | `win10-victim`  | Optional workstation target, Event Viewer / Sysmon logs  |
| DNS SERVER\* | `raspberry-dns` | Optional internal DNS resolver for DNS-related scenarios |

\* Windows 10 and the DNS server are optional, depending on how deep you want to go in this iteration of the lab.

### Cloud Environment (AWS)

| Role             | Resource                | Description                                                                 |
| ---------------- | ----------------------- | --------------------------------------------------------------------------- |
| CLOUD HOST       | EC2 instance            | Linux host in a dedicated VPC/subnet, target for external scans/brute force |
| MONITORING       | CloudTrail & CloudWatch | Management and system logs from AWS and the EC2 instance                    |
| THREAT DETECTION | GuardDuty               | Detects suspicious activity (scans, brute force, anomaly patterns)          |
| COMPLIANCE       | AWS Config              | Detects misconfigured Security Groups (e.g., ports open to 0.0.0.0/0)       |

<!-- In the next sections, I’ll detail the architecture, attack scenarios, hunting hypotheses and the GRC artefacts (risk register, compliance mapping and security policy) that turn this lab into something ready for a SOC / Cloud Security / GRC portfolio. -->

<!-- ## 🧱 High-Level Architecture

### On-prem (Proxmox)

- **pfSense**

  - Firewall and gateway
  - Logs for:
    - scans
    - brute force
    - suspicious connections

- **Ubuntu Server**

  - Main target for internal attacks
  - Logs:
    - `/var/log/auth.log`
    - `/var/log/syslog` / `journalctl`

- **Windows 10** (optional)

  - Logs:
    - Event Viewer → Security
    - Sysmon (optional, but I love it)

- **Kali**

  - Internal attacker
  - Tools:
    - `nmap`
    - Hydra / Ncrack
    - other usual suspects

- **raspberry-dns** (optional)
  - Internal DNS server
  - Logs for:
    - DNS queries
    - anomalies and suspicious domains

--- -->

## AWS Side – Cloud Security & Detection

#### VPC via CloudFormation

All AWS resources live in a dedicated VPC, created with a simple CloudFormation template[🔗](https://github.com/Julmgc/hybrid-soc-cloud-threat-hunting-grc-lab/blob/main/cloudformation/ec2-basic.yml){:target="\_blank" rel="noopener"}
.

Planned resources:

- 1 VPC `10.10.0.0/24`
- 1 public subnet `10.10.0.0/24`
- 1 Internet Gateway + route table with default route → IGW
- 2 Security Groups:
  - `secure-ec2-sg`
    - SSH (22) only from my IP
  - `lab-misconfig-sg`
    - port **2222/TCP** open to `0.0.0.0/0`  
      (created by the template, attached manually only during the exposure scenario)
- 1 EC2 Linux instance (`t3.micro`), created via CloudFormation and attached to the **secure** Security Group by default

The idea is not to build a production-grade VPC for a bank, but to show:

- why the default VPC is not ideal
- how to create a dedicated, controlled VPC via IaC
- how to **consciously misconfigure** something to observe detections

### CloudTrail, GuardDuty & AWS Config

---

To monitor and detect what happens in the cloud side, I’ll use three main services.

#### CloudTrail

- 1 trail for **management events** in the lab region
- Logs to a dedicated S3 bucket
- Used to track:
  - security group changes
  - EC2 lifecycle (start/stop)
  - relevant configuration changes

#### GuardDuty

- Enabled in the same region as the VPC
- Focus on detecting:
  - SSH brute force
  - external scans
  - suspicious behaviour around the EC2 instance

The plan is to generate real findings (not just “it’s enabled”) and document:

- attack commands
- relevant GuardDuty findings
- how they correlate with host and network logs

#### AWS Config (minimal but meaningful)

To avoid going crazy with 30+ rules, I’ll keep it small but relevant:

- AWS Config enabled in the region
- **1–2 managed rules** focusing on:
  - security groups allowing `0.0.0.0/0` on sensitive ports (SSH, RDP, etc.)

The goal here is to show **security drift detection**, not to recreate an entire enterprise baseline.

### SNS + EventBridge – Alerting

---

Detection is nice, but in real life someone needs to actually **see** the problem.

To simulate that:

- 1 SNS Topic: `security-alerts`
  - with my e-mail subscribed
- 1 EventBridge Rule:
  - triggers on **Config rule NON_COMPLIANT**
  - sends the event to the `security-alerts` SNS topic
- (Optional, but planned): another EventBridge rule:
  - triggers on **GuardDuty findings** above a severity threshold
  - also sends alerts to `security-alerts`

This gives me:

- compliance alerting (Config → SNS)
- threat alerting (GuardDuty → SNS)

### 🔍 Attack & Hunting Scenarios

Some of the scenarios I’ll implement and document:

1. **Internal recon and brute force (on-prem)**

   - Kali scanning the internal network (Nmap)
   - brute force SSH against the Ubuntu server
   - evidence collected from:
     - pfSense firewall logs
     - Ubuntu `auth.log` and `syslog`

2. **Exposed EC2 with SSH on port 2222**

   - EC2 attached to `lab-misconfig-sg` (port 2222 open to the internet)
   - external scan & brute force attempts
   - detections in:
     - GuardDuty
     - CloudTrail
     - EC2 system logs (auth log)

3. **(Optional) DNS anomalies**
   - using `raspberry-dns` to resolve suspicious domains
   - hunting for malicious or weird DNS patterns

For each scenario I want at least:

- attack commands and tools used
- where it appears in logs (on-prem + cloud)
- how I would write a **hunting query** to find similar activity again

All hunting notes will live in:

```text
hunting-queries/
  ├── linux-ssh.md
  ├── post-exploitation.md
  ├── windows-logon.md
  ├── aws-ec2.md
  └── guardduty.md
```
