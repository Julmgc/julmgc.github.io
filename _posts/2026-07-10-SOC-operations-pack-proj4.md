---
title: "PowerShell Alert Triage with Sysmon and Wazuh"
date: 2026-07-10
layout: single
lang: en
translation_key: endpoint-alert-triage-powershell-wazuh
author_profile: true
author: julia_en
toc: true
toc_sticky: true
categories: [Labs, SOC]
tags:
  [
    Wazuh,
    Windows,
    Sysmon,
    PowerShell,
    Endpoint-Security,
    Detection-Engineering,
    Incident-Response,
    MITRE-ATT&CK,
  ]
excerpt: "Hands-on endpoint triage lab using Sysmon and Wazuh to compare PowerShell executions, inspect process-creation telemetry, and distinguish suspicious-looking activity from legitimate administrative behavior."
permalink: /labs/endpoint-alert-triage-powershell-wazuh/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "An endpoint investigation workflow where a PowerShell signal is reviewed using Sysmon and Wazuh."
  image_height: 300px
---

---

### Project summary

In this lab, I generated four safe PowerShell executions on a **Windows 10** endpoint and used **Sysmon + Wazuh** to compare the events and review process creation telemetry.

### PowerShell executions

I generated four safe PowerShell commands within a short time window:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

### Events in Wazuh

All four executions appeared in Wazuh under the same PowerShell-related detection family.

![PowerShell events in Wazuh](/assets/images/proj-4/WIN-ps-initial-alert.png)

I compared the events using the command-line data recorded by Sysmon.

### Event selected for analysis

I selected the following event for closer review:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

The execution stood out because it combined `ExecutionPolicy Bypass` with a connection attempt to `192.168.20.55:1514`, while the other commands were simple administrative queries.

`ExecutionPolicy Bypass` increased the relevance of the event, but I did not treat it as evidence of malicious activity on its own. The review considered the command line, user, host, timestamp, and connection destination before placing the execution in context.

### Sysmon evidence

In Wazuh, I reviewed the process creation event recorded by Sysmon.

![Sysmon process creation evidence](/assets/images/proj-4/WIN-ps-raw-event.png)

The event recorded the full command line, including `-NoProfile` and `-ExecutionPolicy Bypass`, along with the associated user, host, and timestamp.

These fields made it possible to confirm which command had been executed and the context in which it occurred.

### Conclusion

Comparing the four events helped identify an execution that warranted additional review because it combined `ExecutionPolicy Bypass` with a connection to `192.168.20.55:1514`.

The Sysmon evidence validated the command and its context without indicating additional activity in the analyzed scenario.

> **generate the signal → compare events → review the evidence → add context → decide**
