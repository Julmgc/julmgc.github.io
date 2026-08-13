---
title: "AWS SSRF Investigation: IMDS, CloudTrail, and Hardening"
date: 2026-07-20
layout: single
lang: en
translation_key: cloud-incident-ssrf-imds-cloudtrail-hardening
author_profile: true
author: julia_en
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
excerpt: "A cloud security case: I detected an SSRF attempt against an EC2-hosted application, investigated related activity in CloudTrail, and reduced the risk with IMDSv2, internal destination blocking, and least privilege."
permalink: /labs/cloud-incident-ssrf-imds-cloudtrail-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "Door Ajar"
  image_height: 300px
---

---

### Overview

This project demonstrates a cloud security investigation workflow using application evidence and AWS-native services:

- deployed a Flask application on an **EC2** instance with a URL-fetching endpoint;
- generated an **SSRF-style request** targeting the instance metadata service;
- reviewed identity activity and API calls in **CloudTrail**;
- reduced the risk by requiring **IMDSv2**, blocking internal destinations in the application, and applying **least privilege** to the IAM role.

![Proj-3 - Arquitetura - investigação de SSRF em nuvem](/assets/images/proj-3/SSRF-PROJ-PORT.png)

### Controlled setup: EC2 application and overly permissive IAM role

I ran a small Flask service on an EC2 instance with a URL-fetching endpoint. For lab purposes, the instance used an IAM role with broader permissions than necessary, demonstrating how access to the metadata service could increase the impact of an SSRF vulnerability.

This was a controlled scenario using only disposable resources and non-sensitive data.

![Proj-3 - Função IAM anexada à instância EC2](/assets/images/proj-3/EC2-role-attached.png)

### Signal: SSRF-style request in application telemetry

I sent a request to the `/fetch` endpoint targeting the EC2 metadata address (`169.254.169.254`), simulating an SSRF attempt against IMDS.

The focus of the project was investigation rather than exploitation. The relevant signal was that the application received a request targeting the cloud metadata service.

![Proj-3 - Requisição SSRF registrada em JSON](/assets/images/proj-3/APP-ssrf-log.png)

### CloudTrail investigation

Access to `169.254.169.254` was relevant because IMDS can provide instance information and temporary credentials associated with the IAM role.

I therefore reviewed identity activity and API calls in CloudTrail around the time of the suspicious request.

Within the analyzed window, I identified events such as:

- `GetCallerIdentity`;
- `DescribeRegions`;
- `DescribeInstances`.

These events provided visibility into identity and enumeration activity associated with the analyzed period.

> If temporary credentials had been exposed, a call such as `sts:GetCallerIdentity` would be relevant for confirming the AWS account and role context being used.

> ![Proj-3 - Sequência no CloudTrail](/assets/images/proj-3/CT-killchain-seq.png)

### Remediation: reducing the risk path

I applied three hardening measures in the lab to reduce the risk path between SSRF and cloud resource exposure:

- required **IMDSv2** on the EC2 instance;
- blocked requests from the `/fetch` endpoint to the metadata service and local destinations;
- reduced the instance role permissions according to the **principle of least privilege**.

![Proj-3 - IMDSv2 obrigatório](/assets/images/proj-3/EC2-imdsv2.png)

At the application layer, the control included destinations such as `169.254.169.254`, `127.0.0.0/8`, `localhost`, and `169.254.*` ranges.

After the change, a new request to the metadata service returned `403 Forbidden`.

![Proj-3 - Controle defensivo contra SSRF](/assets/images/proj-3/APP-egress-or-allowlist.png)

> As an improvement, this control could be expanded to validate IP addresses returned by DNS resolution and block additional private, loopback, link-local, and local IPv6 ranges.

### Conclusion

---

After hardening, the same request returned `403 Forbidden` while the signal remained visible in application telemetry. IMDSv2 also remained required, and the IAM role permissions were reduced.

The lab brought together a cloud security investigation workflow:

> **detect → review evidence in CloudTrail → remediate → replay the test → validate**
