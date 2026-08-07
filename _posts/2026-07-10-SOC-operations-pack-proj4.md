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
excerpt: "A Windows endpoint triage lab using PowerShell, Sysmon, and Wazuh to compare similar events and examine one selected execution in detail."
permalink: /labs/endpoint-alert-triage-powershell-wazuh/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "An endpoint investigation workflow where a PowerShell signal is reviewed using Sysmon and Wazuh."
  image_height: 300px
---

### Overview

In this lab, I generated four safe PowerShell executions on a Windows 10 endpoint and used **Sysmon + Wazuh** to review the resulting events.

The goal was to compare similar executions, select one for closer review, and examine the process creation evidence recorded by Sysmon.

The workflow was:

> **generate the signal → review it in Wazuh → select the event → examine the evidence → document the conclusion**

### Lab architecture

I used a small setup for Windows endpoint monitoring:

- **Wazuh** — manager, indexer, and dashboard;
- **Windows 10** — monitored endpoint;
- **Sysmon** — process creation telemetry;
- **Wazuh Agent** — forwards endpoint events to Wazuh.

The Sysmon events provided fields such as command line, user, host, parent process, and timestamp for analysis.

### PowerShell executions

To practice triage, I generated four safe PowerShell commands within a short time window:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

All four commands were harmless. The goal was to generate similar endpoint events and then compare their command lines.

### Events in Wazuh

The executions appeared in Wazuh under the same PowerShell-related detection family.

![PowerShell-related events in Wazuh](/assets/images/proj-4/WIN-ps-initial-alert.png)

I compared the events using the command-line data recorded by Sysmon.

### Reviewing the selected execution

Of the four commands, I selected the following one for closer review:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

The combination of `ExecutionPolicy Bypass` with a connectivity test to a specific host and port made this event stand out from the others.

The command can also be used for legitimate administrative tasks. For that reason, the assessment should not rely on the command line alone: it is important to determine whether the user and host normally perform this type of activity and whether the behavior is consistent with the endpoint's role.

The other commands (`Get-Process`, `Get-Service`, and `whoami; hostname; Get-Date`) were simple administrative checks involving processes, services, and host identity.

### Sysmon evidence

After selecting the execution, I opened the Sysmon process creation event in Wazuh.

![Sysmon process creation event](/assets/images/proj-4/WIN-ps-raw-event.png)

The event confirmed the full command line, the use of `-NoProfile` and `-ExecutionPolicy Bypass`, as well as the associated host, user, and timestamp.

These fields made it possible to verify exactly what command was executed and in what context.

### Conclusion

The selected event stood out because it combined `ExecutionPolicy Bypass` with a connectivity test to a specific host and port.

The Sysmon evidence confirmed the command line, user, host, and execution time, without indicating additional impact in the lab scenario.

The lab demonstrated a simple endpoint triage workflow:

> **generate the event → identify the signal → review the evidence → add context → decide**
