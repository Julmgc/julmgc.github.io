---
title: "Cloud Incident Investigation: SSRF Signal → IMDS Risk → CloudTrail Timeline → Hardening"
date: 2026-03-10
layout: single
lang: en
translation_key: cloud-incident-ssrf-imds-cloudtrail-hardening
author_profile: true
toc: true
toc_sticky: true
categories: [Labs, SOC, Cloud]
tags:
  [
    AWS,
    EC2,
    CloudTrail,
    SSRF,
    IMDSv2,
    Wazuh,
    OWASP,
    Incident-Response,
    MITRE-ATT&CK,
  ]
excerpt: "A cloud case: detected an SSRF attempt on an EC2-hosted app, investigated related AWS identity activity in CloudTrail, and reduced risk with IMDSv2, egress filtering, and least privilege."
permalink: /labs/cloud-incident-ssrf-imds-cloudtrail-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A small crack in trust can open the door to deception — DNS poisoning in action."
  image_height: 300px
---

## Overview

This project demonstrates a focused cloud SOC workflow using AWS-native evidence:

- deployed a small Flask app on **EC2** with a URL-fetching endpoint (`/fetch?url=...`)
- generated an **SSRF-style request** targeting the EC2 metadata address to simulate credential exposure risk
- investigated related **identity and API activity** in **CloudTrail Event history**
- correlated **web telemetry** with **cloud evidence** in a SIEM-friendly way
- reduced risk by enforcing **IMDSv2**, restricting **egress**, and tightening **IAM permissions**

![Proj-3 - Architecture - ssrf-to-cloud-investigation](/assets/images/proj-3/ARCH-ssrf-cloudtrail.png)

<small><em>
A simple architecture view showing the app on EC2, the SSRF-style request path, metadata risk, CloudTrail investigation, and hardening controls.
</em></small>

## Setup (controlled): EC2 app + risky lab role context

I ran a small Flask service on an EC2 instance with a URL-fetching endpoint. For lab purposes, the instance used an IAM role with broader permissions than necessary so I could demonstrate why metadata access would increase risk if application controls were weak.

This was a controlled lab scenario using only disposable resources and non-sensitive data.

![Proj-3 - EC2 role attached to instance](/assets/images/proj-3/EC2-role-attached.png)

<small><em>
Blast-radius context — shows the EC2 instance profile / IAM role that would matter if metadata credentials were exposed.
</em></small>

## Signal: SSRF-style request in application telemetry

To produce a SOC-usable signal, the application logs each request as structured JSON. I first generated a normal fetch request as a baseline, then submitted a request targeting the EC2 metadata address (`169.254.169.254`) to simulate an SSRF attempt toward IMDS.

The point of this project is the investigation workflow, not exploitation. The key detection signal is that the application received a suspicious request directed at cloud metadata space.

![Proj-3 - SSRF request logged in JSON](/assets/images/proj-3/APP-ssrf-log.png)

<small><em>
Detection signal — structured application telemetry showing a fetch request to the EC2 metadata address (169.254.169.254), including the requested URL, source IP, status, and timestamp.
</em></small>

## Why this mattered

The metadata IP is significant because EC2 uses the Instance Metadata Service to expose instance information and, when applicable, temporary IAM role credentials. A request attempting to reach that address from an application fetcher is not normal user behavior and deserves investigation.

From an analyst perspective, this raises questions such as:

- Was the request blocked or allowed?
- Could the application reach metadata?
- Was there any related AWS API activity shortly afterward?
- What role would have been affected if credentials were exposed?

## Investigation: CloudTrail Event history timeline

To build a defensible timeline, I pivoted into CloudTrail Event history around the SSRF window and reviewed AWS identity and API activity associated with the instance role.

What I looked for:

- STS-related activity such as `GetCallerIdentity`
- enumeration patterns such as `List*` and `Describe*`
- access to S3 or other services if relevant to the role
- session / actor context, timestamps, and event sequence

This let me answer the core analyst questions: what identity was active, what actions occurred, and how close they were to the suspicious web event.

![Proj-3 - CloudTrail sequence](/assets/images/proj-3/CT-killchain-seq.png)

<small><em>
Investigation proof — CloudTrail shows the relevant identity/API activity aligned to the suspicious request window.
</em></small>

> If the SSRF attempt had successfully retrieved metadata credentials, a common first step for attackers would be calling **sts:GetCallerIdentity** to confirm the AWS account and role context. Analysts often look for this API call as an early indicator of credential misuse.

<!-- ## Correlation: web signal first, cloud evidence second

To keep the project recruiter-readable, the correlation story is intentionally simple:

- a suspicious metadata-targeting request appears in app telemetry
- within the same investigation window, CloudTrail shows identity/API activity relevant to the EC2 role

That is enough to demonstrate a credible SOC workflow:
detect → pivot → investigate → document → harden.

![Proj-3 - SIEM correlation view](/assets/images/proj-3/WAZUH-correlation.png)

<small><em>
Screenshot purpose: analyst workflow — one view tying the suspicious app request to the cloud investigation window for faster triage and reporting.
</em></small> -->

## Remediation: reduce the risk path

I applied three concrete hardening measures to reduce the SSRF-to-cloud-risk path:

- required **IMDSv2** on the EC2 instance
- restricted the fetcher so it could not request metadata / link-local destinations
- tightened the instance role toward **least privilege**

These changes do not just fix one request pattern; they reduce both the likelihood and the blast radius of similar issues.

![Proj-3 - IMDSv2 required](/assets/images/proj-3/EC2-imdsv2.png)

<small><em>
Chain-breaker control — shows the EC2 metadata service hardened with IMDSv2 required.
</em></small>

![Proj-3 - SSRF defense control](/assets/images/proj-3/APP-egress-or-allowlist.png)

<small><em>
Application/network hardening proof — shows that the fetcher was restricted from reaching metadata or untrusted destinations.
</em></small>

## Add application-side blocking for metadata and local addresses

Next, I implemented a simple defensive control in the Flask application to prevent the fetch endpoint from requesting metadata or local network addresses.

The /fetch endpoint was originally designed to demonstrate how an SSRF-style request could reach arbitrary URLs. To reduce that risk, the application now blocks requests to:

- 169.254.169.254 (EC2 metadata service)

- 127.0.0.0/8

- localhost

- link-local address ranges such as 169.254.\*

This defensive check ensures the application cannot be used as a proxy to access sensitive internal endpoints.

Example of the protection logic:

**BLOCKED_HOSTS** = {
"169.254.169.254",
"127.0.0.1",
"localhost"
}

**BLOCKED_PREFIXES** = (
"169.254.",
"127.",
)

When a blocked destination is requested, the application logs the event and returns a 403 response.

**<em>For this project, the goal was to demonstrate the remediation workflow rather than build a production-grade SSRF defense. In a real environment, I would also validate resolved IP addresses and restrict access to additional private, loopback, link-local, and IPv6 local address ranges to reduce the risk of filter bypass techniques.</em>**

## Validation: before/after risk reduction

After hardening, I re-ran the same SSRF-style request targeting the metadata service.

The request was still visible in application telemetry, but it no longer followed the same risk path:

- IMDSv2 was required on the instance

- the application now blocked metadata and local destinations

- the request returned 403 Forbidden instead of allowing access to the metadata endpoint

This confirmed that the suspicious pattern remained detectable while the risky behavior had been reduced through both infrastructure-side and application-side controls.

## Mini investigation checklist (TL;DR)

1. **Review app telemetry:** identify suspicious requests to metadata / link-local destinations
2. **Scope the role:** confirm which IAM role / instance profile was attached to the EC2 instance
3. **Pivot to CloudTrail:** review nearby STS and API activity for timeline and actor context
4. **Reduce exposure:** require IMDSv2, block metadata access from the fetcher, and tighten role permissions
5. **Validate:** re-run the test and confirm the risky path is reduced post-fix

**Cloud Triage Checklist**

- Review app logs for metadata-targeting requests
- Check CloudTrail for nearby STS and Describe/List activity
- Verify IMDSv2 is required
- Confirm the fetch endpoint blocks metadata/local addresses
- Record suspicious URL, source IP, timestamps, and applied fixes

## What I’d improve next

- Add a custom Wazuh rule to flag requests to metadata or link-local ranges
- Expand the app to log a request ID for easier app-to-cloud timeline stitching
- Add a small Python helper to summarize CloudTrail events around a given time window
- Compare host-level, app-level, and network-level SSRF mitigations in a follow-up lab
