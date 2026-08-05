---
title: "Mini SOC Lab: Web Triage, Detection, and Containment with Wazuh"
date: 2026-07-31
layout: single
lang: en
translation_key: mini-soc-wazuh-hardening
author_profile: true
author: julia_en

sidebar:
  nav: "sidebar"

toc: true
toc_sticky: true
categories: [Labs, SOC]
tags: [Wazuh, SIEM, pfSense, Proxmox, Detection-Engineering]
excerpt: "A Proxmox-based mini SOC environment with structured telemetry, custom Wazuh detection, suspicious web activity triage, and validated endpoint containment."
permalink: /labs/mini-soc-wazuh-hardening/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A SOC workflow covering signal generation, detection, triage, containment, and validation."
  image_height: 300px
---

### Executive summary

---

I built a mini SOC environment in **Proxmox** using Wazuh, pfSense, a Flask application behind Nginx, and a Jumpbox used to generate controlled traffic.

The lab analyzed three types of activity:

- repeated authentication failures;
- web resource reconnaissance;
- requests containing SQL injection-like patterns.

For the SQL injection scenario, I used a built-in Wazuh rule and created local rule `100200` to increase the alert priority.

Containment was applied directly on UbuntuApp with UFW, blocking `TCP/80` connections from the Jumpbox while preserving SSH access for management.

The demonstrated workflow was:

> **signal generation → detection → triage → containment → validation**

### Lab architecture

---

The environment consisted of four virtual machines:

| Component     | Role                                               |
| ------------- | -------------------------------------------------- |
| **Wazuh**     | All-in-one manager, indexer, and dashboard         |
| **UbuntuApp** | Nginx, Flask application, and Wazuh agent          |
| **pfSense**   | Gateway, firewall policies, and syslog source      |
| **Jumpbox**   | Bastion host and controlled source of test traffic |

![Lab layout in Proxmox](/assets/images/proj-1/PROXMOX-lab-layout.png)

**Relevant topology**

The analyzed traffic was generated inside VLAN20.

The Jumpbox acted as the request source, while UbuntuApp was the target web server.

```text
Jumpbox
192.168.20.10
      ↓
HTTP traffic inside VLAN20
      ↓
UbuntuApp
192.168.20.50
```

> **SOC note:** because the Jumpbox and UbuntuApp were in the same VLAN, lateral traffic could flow without traversing pfSense. Endpoint and application telemetry were therefore essential for analysis and containment.

The Flask application generated JSON logs containing the main HTTP request fields, which were later ingested and searched in Wazuh.

### Repeated authentication failures

---

From the Jumpbox, I generated approximately 20 failed login attempts against the `/login` endpoint.

![Failed login attempts](/assets/images/proj-1/LOGIN-ATTEMPTS.png)

In Wazuh, the query returned 20 `POST /login` events with status `401`, all originating from `192.168.20.10`.

![Authentication events filtered in Wazuh Discover](/assets/images/proj-1/WAZUH_COUNT_UP.png)

The relevant signal was not an isolated `401` response, but repeated failures from the same source within a short time window. In production, the detection should use a threshold grouped by `srcip`.

### Web resource reconnaissance

---

I also generated requests to different sensitive web resources, simulating web reconnaissance activity.

![Burst of requests to sensitive web resources](/assets/images/proj-1/QUICK BURST.png)

**DQL query**

```text
agent.name:"ubuntu-app"
and data.app:"ticketing-lab"
and data.event:"http_request"
and data.srcip:"192.168.20.10"
and (
  data.path:"/wp-login.php"
  or data.path:"/admin"
  or data.path:"/.git/config"
  or data.path:"/phpmyadmin"
  or data.path:"/.env"
  or data.path:"/server-status"
)
```

![Reconnaissance events filtered in Wazuh Discover](/assets/images/proj-1/wazuh_discover_hits.png)

<p style="font-size: 0.85em; font-style: italic; text-align: center;">
The query confirmed that a single source accessed multiple sensitive web resources in sequence.
</p>

Although the responses were `404`, the combined pattern was more relevant than each event in isolation. In production, the detection should consider resource diversity, request rate, and authorized scanners.

### Detecting a SQL injection-like pattern

---

I sent test requests to the `/search` endpoint containing URL-encoded patterns such as:

```text
' OR 1=1 --
```

The goal was to generate detectable telemetry in Wazuh without exploiting the application.

![Test requests sent to the search endpoint](/assets/images/proj-1/payloads.png)

To validate ingestion, I filtered one of the submitted strings in Wazuh.

**DQL query**

```text
agent.name:"ubuntu-app"
and data.app:"ticketing-lab"
and data.path:"/search"
and data.srcip:"192.168.20.10"
and data.query_string:"q=%27%20OR%201%3D1%20--"
```

![SQL injection-like pattern identified in Wazuh](/assets/images/proj-1/wazuh-SQLi.png)

<p style="font-size: 0.85em; font-style: italic; text-align: center;">
The query returned three events because the same request was repeated during the test.
</p>

### Custom prioritization rule

---

Wazuh includes a built-in rule for web patterns associated with SQL injection:

```text
rule.id: 31164
level: 6
```

To increase the visibility of the signal in the lab, I created a local rule that generates a new alert when the built-in rule fires.

```text
Local rule: 100200
Level: 12
Source rule: 31164
MITRE ATT&CK: T1190 — Exploit Public-Facing Application
```

![Rule 100200 in local_rules.xml](/assets/images/proj-1/RULE.ID-100200-local_rules.png)

Local rule `100200` generated level `12` alerts, making the signal easier to prioritize in Wazuh.

![Local rule 100200 alert with level 12 in Wazuh Discover](/assets/images/proj-1/ruleID100200-BACKUP.png)

**Rule limitation**

The demonstrated version generated a new alert for every occurrence of rule `31164`.

In production, it would be more appropriate to apply a time-based threshold grouped by `data.srcip`, while also considering authorized scanners, normal application behavior, and asset criticality.

### How triage would work in production

---

In production, triage would begin with the alert generated by rule `100200`:

```text
agent.name:"ubuntu-app" and rule.id:"100200"
```

The analyst would validate the source, endpoint, `query_string`, `full_log`, and event frequency.

The activity would then be correlated with other signals from the same source, such as authentication failures, web reconnaissance, and network connections, while also checking whether the activity was authorized.

The objective would be to determine whether the alert represented legitimate activity, an isolated attempt, or part of a broader offensive sequence.

### Triage support resources

---

I created a Wazuh dashboard to visualize alert sources, timestamps, and associated URLs.

![Web attack dashboard](/assets/images/proj-1/E4-SOC-WEB-ATTACKS.png)

I also developed a Python script to summarize IP addresses, endpoints, status codes, and query strings.

<a href="https://github.com/Julmgc/detections/blob/main/helpers/summarize_flask_logs.py" target="_blank" rel="noopener noreferrer">
summarize_flask_logs.py
</a>

Example usage:

```bash
python3 summarize_flask_logs.py /var/log/flaskapp/app.log
```

![Python triage summary](/assets/images/proj-1/python-triage-summary.png)

The script summarizes application logs by:

- top source IP addresses;
- most frequently accessed endpoints;
- HTTP status codes;
- observed query strings.

These resources support initial analysis but do not replace validation in Wazuh events and raw application logs.

### Containment and validation

---

Because the Jumpbox and UbuntuApp were in the same VLAN, the test requests were sent directly between the two machines without traversing pfSense.

I therefore applied containment directly on UbuntuApp with UFW, blocking `TCP/80` connections originating from `192.168.20.10`.

SSH access remained available for virtual machine management.

![UFW containment rule](/assets/images/proj-1/ufw-status.png)

After applying the UFW rule, I validated connectivity from the Jumpbox.

![Port 80 block validation with Nmap and Netcat](/assets/images/proj-1/ufw-port-80-validation.png)

Nmap showed that `22/tcp` remained open for SSH, while `80/tcp` appeared as `filtered`. The additional Netcat test also timed out on port 80.

These results confirmed that management access was preserved while HTTP connections from the Jumpbox were blocked.

### Conclusion

---

The lab integrated structured telemetry, Wazuh detection, alert analysis, and selective endpoint containment.

Validation showed that `80/tcp` was filtered for the Jumpbox while SSH access remained available for management.

As next steps, the application could incorporate parameterized queries, input validation, HTTPS, least privilege, and time-based alert correlation.
