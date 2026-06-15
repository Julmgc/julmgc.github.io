---
title: "Endpoint Alert Triage: Investigating a Lab-Generated PowerShell Signal with Sysmon + Wazuh"
date: 2026-03-23
layout: single
author_profile: true
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
excerpt: "A Windows endpoint triage case: generate a review-worthy PowerShell signal, investigate it in Wazuh using Sysmon evidence, compare it with nearby endpoint activity, and document an analyst verdict."
permalink: /labs/endpoint-alert-triage-powershell-wazuh/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A small crack in trust can open the door to deception — DNS poisoning in action."
  image_height: 300px
---

## Overview

Windows endpoint alerts are often ambiguous. A PowerShell event may be completely benign, mildly suspicious, or worth escalation depending on the command line, parent process, host context, and surrounding activity. This project focuses on that judgment step.

In this post, I:

- generated a small set of safe PowerShell executions on a Windows 10 endpoint
- used **Sysmon + Wazuh** to review the resulting endpoint events
- identified which execution deserved deeper inspection
- investigated the raw process creation evidence in Wazuh
- compared the reviewed event with nearby endpoint activity
- documented a practical analyst verdict instead of forcing containment

Artifacts: Wazuh event views, raw Sysmon process evidence, surrounding host context, and a short analyst note.

## Why this is SOC-realistic

- The signal is backed by **real Sysmon process creation telemetry**, not just a screenshot of a command prompt.
- The same broad Wazuh rule family can match multiple PowerShell executions, so the analyst has to inspect the **underlying event details**, not just the alert name.
- The outcome is not “I blocked something.” The outcome is a **clear analyst decision**: review, explain, monitor, or escalate depending on context.

## Lab Architecture

I used a small Windows-focused endpoint monitoring setup for this project:

- **Wazuh (SIEM)** — manager + indexer + dashboard
- **Windows 10 endpoint** — Sysmon + Wazuh agent installed
- **Sysmon** — process creation telemetry
- **Wazuh agent** — forwards endpoint events into Wazuh

![Proj-4 - Architecture - windows-endpoint-pipeline](/assets/images/proj-4/ARCH-endpoint-investigation 1.png)

## Endpoint telemetry: making Windows process activity reviewable

To make PowerShell executions reviewable in a SOC-like way, I used **Sysmon process creation events** on the Windows host and forwarded them into Wazuh through the Wazuh agent.

That gave me the fields that matter most during endpoint triage:

- timestamp
- host (`agent.name`)
- parent image
- command line
- parent command line
- user context
- rule description

Those details are what make a PowerShell alert useful. Without them, the analyst is left with a generic rule name and very little context.

## Lab-generated signal: four safe PowerShell executions

To make the triage exercise more realistic, I generated **four safe PowerShell commands** on the Windows endpoint within a short window. All four were harmless, but one of them was intentionally more review-worthy than the others.

Commands used:

```powershell
powershell.exe -Command "Get-Process | Select-Object -First 5"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
powershell.exe -Command "Get-Service | Select-Object -First 5"
powershell.exe -Command "whoami; hostname; Get-Date"
```

The idea was not to simulate malware. The idea was to create a small set of similar endpoint events and then practice deciding which one deserved deeper inspection.

## Initial signal: PowerShell-related event family in Wazuh

Wazuh grouped these executions under the same broad PowerShell-related detection family. That is realistic: in many SOC environments, the first signal is not a perfect detection but a broad review-worthy event that still needs analyst context.

![Proj-4 - Initial Wazuh PowerShell signal](/assets/images/proj-4/WIN-ps-initial-alert.png)

<small><em>
Initial Wazuh endpoint signal: PowerShell-related event(s) on the Windows endpoint that triggered the investigation, showing the host, timeline, and matching rule used as the starting point for triage. </em></small>

At this stage, the analyst questions were:

- What host is affected?
- What detection family triggered the review?
- How many related events occurred in the same time window?
- Which one deserves deeper inspection?

## Evidence review: selecting the most review-worthy execution

Of the four commands, one stood out as the strongest candidate for deeper review:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.55 -Port 1514"
```

This command warrants deeper inspection because it uses PowerShell with ExecutionPolicy Bypass to test connectivity to a specific internal host and port, blending legitimate administration with behavior that can also indicate reconnaissance.

The other three commands were still captured in Wazuh, but they looked more like routine administrative or identity-check commands:`Get-Process`; `Get-Service`; `whoami; hostname; Get-Date`.

This is where the project becomes a triage exercise instead of just “PowerShell happened.”

## Raw Sysmon evidence: one process creation event

After selecting the most review-worthy execution, I opened the raw Sysmon process creation event in Wazuh.

![Proj-4 - Raw Sysmon event - part 1](/assets/images/proj-4/WIN-ps-raw-event.png)

<small><em>
Raw Sysmon process creation event: timestamp, agent name, command line, parent command line, parent image, user context, and Wazuh rule description.
</em></small>

Triage notes

- The event was not just “PowerShell activity.”
- The command line showed why this execution deserved deeper review than the others.
- The parent process context helped frame how the execution was launched.
- This is the step that turns a broad detection into usable analyst evidence.

## Process context: surrounding endpoint activity on the same host

A single raw event is useful, but not enough by itself. After reviewing the selected Sysmon event, I widened the Wazuh view again to compare it with surrounding endpoint activity on the same host.

![Proj-4 - Process context in Wazuh](/assets/images/proj-4/WIN-ps-context.png)

<small><em>
Process context in Wazuh: the reviewed PowerShell execution shown alongside nearby endpoint activity on the same Windows host, helping frame the analyst assessment. </em></small>

This broader view was useful because it showed that the reviewed event existed inside a short cluster of nearby host activity rather than as an isolated screenshot with no surrounding evidence.

Triage notes

- The broad PowerShell rule was still useful, but the real assessment depended on event details and host context.
- The selected event was the strongest candidate for deeper review because of its flags and network-oriented action.
- The surrounding host activity did not show immediate evidence of persistence or follow-on malicious behavior in the same reviewed window.

## Analyst assessment: close, monitor, escalate, or contain?

This case was intentionally designed as a decision exercise rather than a containment exercise.

Because I generated the signal in a controlled lab, I already knew the root cause while testing. But the purpose of the project was to practice how I would assess the event if I encountered it as a queue item without that certainty.

My reasoning was:

- the event was **review-worthy**
- the command line justified analyst attention
- the process context was visible and explainable
- there was no evidence of persistence, credential access, or post-execution impact in the reviewed window
- the event did **not** justify containment on its own

That means the most realistic disposition for this case would be:

- **do not close immediately without review**
- **do not jump straight to containment**
- **document the context**
- **close as explained or monitor for recurrence**, depending on the environment
- **escalate only if additional suspicious evidence appears**

In a real SOC, this matters. Not every alert should become a high-severity incident, but not every alert should be dismissed either.

## What I’d improve next

- Add a custom Wazuh rule for more specific suspicious PowerShell patterns or unusual parent-child chains.
- Compare clearly benign PowerShell executions against more review-worthy ones in the same host timeline.
- Include a second case where PowerShell execution is followed by network or persistence activity.
- Build a small saved search or dashboard for Windows endpoint triage.
- Add a follow-up section showing how the disposition would change if the same behavior appeared on multiple hosts.
