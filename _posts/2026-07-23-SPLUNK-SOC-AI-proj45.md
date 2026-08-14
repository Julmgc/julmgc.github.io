---
title: "LLM-Assisted SOC Alert Triage with Splunk, Sysmon, Python, and OpenAI"
date: 2026-07-23
layout: single
lang: en
translation_key: llm-assisted-soc-triage
author_profile: true
author: julia_en
toc: true
toc_sticky: true
categories: [Labs, SOC, AI-Security]
tags:
  [
    Splunk,
    SOC,
    SIEM,
    Sysmon,
    Python,
    OpenAI,
    LLM,
    MITRE-ATT&CK,
    Alert-Triage,
    Alert-Analysis,
  ]
excerpt: "A SOC triage lab using Splunk, Sysmon, Python, and OpenAI to structure alert analysis and compare LLM output with a manual assessment."
permalink: /labs/llm-assisted-soc-triage-splunk/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A triage workflow where AI supports initial alert analysis and the results are compared with manual assessment."
  image_height: 300px
---

### Project summary

This project tests the use of an LLM to support initial alert triage.

The lab uses **Windows Event Logs and Sysmon**, forwarded to **Splunk**, with **Python** processing and integration with the **OpenAI API** to generate structured assessments and compare them with a manual analysis based on the same evidence.

**GitHub repository:** [AI-ASSISTED-SOC-TRIAGE](https://github.com/Julmgc/AI-ASSISTED-SOC-TRIAGE){: target="\_blank" rel="noopener noreferrer" }

### Tools used

- **Splunk Enterprise**
- **Sysmon**
- **Windows Event Logs**
- **Python**
- **OpenAI API**
- **MITRE ATT&CK**

### What I built

I created three alert scenarios involving local account creation, PowerShell execution, and a controlled false positive.

Each alert was structured as JSON and processed by a Python script that sent the available evidence to the LLM using a constrained triage prompt.

The model-generated assessments were then compared with a manual analysis based on the same evidence.

### Alert 01: local account creation

The main case uses **Alert 01 — New local account created**.

The activity generated Windows Security Event ID `4720`.

I searched for the event in Splunk:

```spl
index=* source="WinEventLog:Security" EventCode=4720
| table _time host Subject_Account_Name Target_Account_Name
| sort -_time
```

![New local user created event reviewed in Splunk](/assets/images/proj-6/splunk-windows-event-4720.png)

The event confirmed that the `lab_backup` account was created on `DESKTOP-HRMT55O` by user `jules`.

I then reviewed Sysmon Event ID `1` to add more process context:

```spl
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
"lab_backup"
| search EventID=1
| table _time host User Image CommandLine ParentImage
```

![Sysmon-Event-ID1](/assets/images/proj-6/Sysmon-Event-ID1,-process-creation.png)
The Sysmon event added the command line and parent-process information associated with the account creation.

The LLM produced:

```json
{
  "risk_level": "medium",
  "classification": "needs_review",
  "mitre_attack_mapping": [
    {
      "technique_id": "T1136.001",
      "technique_name": "Create Account: Local Account",
      "confidence": "high"
    }
  ],
  "triage_questions": [
    "Is the user authorized to create local accounts on this endpoint?",
    "Was there an approved change ticket or lab activity?",
    "Was the account later added to a privileged group?",
    "Was the account subsequently used to log in?"
  ],
  "confidence": "medium"
}
```

My manual assessment reached the same result:

```text
Classification: needs_review
Risk level: medium
```

The evidence confirmed that the account had been created, but it did not establish whether the action was authorized or malicious.

The LLM also mapped the activity to `T1136.001 — Create Account: Local Account` without concluding, in the absence of additional evidence, that compromise or persistence had occurred.

### Additional cases

**Alert 02 — Suspicious PowerShell**

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Test-NetConnection 192.168.20.45 -Port 9997"
```

Both assessments classified the event as `needs_review`.

The LLM assigned a `low` risk level, while my manual assessment assigned `medium`, giving more weight to the combination of `ExecutionPolicy Bypass` and a network connectivity test.

The command alone was not treated as evidence of compromise.

---

**Alert 03 — Noisy false positive**

```cmd
cmd.exe /c whoami
```

The command was part of a documented test used to verify command-execution visibility in Splunk.

Both the LLM and the manual assessment classified the event as `benign` with `low` risk.

The model did not treat the use of `cmd.exe` or `whoami` alone as evidence of malicious activity.

### Results comparison

| Alert                      | AI                            | Manual assessment             | Result            |
| -------------------------- | ----------------------------- | ----------------------------- | ----------------- |
| **01 — New local account** | `needs_review`, `medium` risk | `needs_review`, `medium` risk | Aligned           |
| **02 — PowerShell**        | `needs_review`, `low` risk    | `needs_review`, `medium` risk | Partially aligned |
| **03 — False positive**    | `benign`, `low` risk          | `benign`, `low` risk          | Aligned           |

The main difference appeared in Alert 02: the classification was the same, but my manual assessment gave more weight to the combination of `ExecutionPolicy Bypass` and network connectivity activity.

### Key takeaways

The LLM was useful for:

- summarizing evidence and highlighting relevant observables;
- identifying missing context through triage questions;
- suggesting MITRE ATT&CK mappings with confidence levels;
- keeping conclusions limited to the available evidence.

The main limitation was its dependence on the context provided. Information about authorization, process relationships, execution results, and related activity could significantly change the assessment.

The value of the LLM in this lab was in providing structured support for analysis, not in making autonomous decisions.

> **AI as support for analysis, with conclusions limited to the available evidence.**
