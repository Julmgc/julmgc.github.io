---
title: "LLM-Assisted SOC Triage with Splunk, Sysmon, Python, and OpenAI"
date: 2026-07-06
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
    Detection-Engineering,
    Incident-Response,
  ]
excerpt: "A hands-on SOC triage lab using Splunk, Sysmon, Python, and OpenAI to summarize alerts, suggest MITRE ATT&CK mappings, and compare AI output against manual analyst validation."
permalink: /labs/llm-assisted-soc-triage-splunk/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A SOC analyst workflow where AI assists with triage while human validation remains the final control."
  image_height: 300px
---

## Project summary

This project evaluates how an LLM can assist with early-stage SOC triage.

I built a small SOC-style lab using **Splunk**, **Sysmon**, **Python**, and the **OpenAI API**. The workflow collects Windows telemetry, extracts alert context, sends structured evidence to an LLM, and compares the AI-generated triage output against manual analyst review.

**GitHub repository:** [AI-ASSISTED-SOC-TRIAGE](https://github.com/Julmgc/AI-ASSISTED-SOC-TRIAGE){: target="\_blank" rel="noopener noreferrer" }

The goal was not to automate final decisions. The goal was to test whether AI can help an analyst move faster while still avoiding unsupported conclusions.

## Tools used

- **Splunk Enterprise** — SIEM/search layer
- **Splunk Universal Forwarder** — Windows log forwarding
- **Sysmon** — endpoint telemetry
- **Python** — alert parsing and LLM workflow
- **OpenAI API** — structured triage output
- **MITRE ATT&CK** — technique mapping
- **Windows Event Logs** — authentication and account-management events

## Lab architecture

```text
Windows VM
  ↓
Sysmon + Windows Event Logs
  ↓
Splunk Universal Forwarder
  ↓
Splunk Enterprise
  ↓
SPL triage searches
  ↓
Python script
  ↓
OpenAI triage prompt
  ↓
AI-generated assessment
  ↓
Manual analyst validation
```

## What I built

I created a dataset of eight SOC-style alerts:

| Alert    | Scenario                         | Purpose                                                                        |
| -------- | -------------------------------- | ------------------------------------------------------------------------------ |
| Alert 01 | Suspicious PowerShell execution  | Test whether the model identifies suspicious flags without assuming compromise |
| Alert 02 | Benign administrative PowerShell | Test whether the model overclassifies normal admin activity                    |
| Alert 03 | Failed login burst               | Test brute-force triage reasoning                                              |
| Alert 04 | SQL injection-style web request  | Test web attack interpretation                                                 |
| Alert 05 | New local user created           | Test account-management and persistence reasoning                              |
| Alert 06 | Suspicious outbound connection   | Test process and network context analysis                                      |
| Alert 07 | Encoded PowerShell command       | Test higher-risk endpoint behavior                                             |
| Alert 08 | Noisy false positive             | Test whether the model can acknowledge weak evidence                           |

Each alert was stored as JSON and processed by a Python script.

## Splunk validation: account creation alert

For the main example, I used **Alert 05: New local user created**.

This scenario is realistic for SOC triage because new local accounts can represent legitimate administrative activity, but they can also indicate persistence when created without authorization.

The activity generated Windows Security Event ID `4720`, which records the creation of a user account.

```text
EventCode: 4720
Event type: User account created
Actor user: DESKTOP-HRMT55O\jules
Target user: lab_backup
Command line: net user lab_backup * /add
```

I first searched the Windows Security Event Log for account-creation events:

```spl
index=\* source="WinEventLog:Security" EventCode=4720
| table \_time host EventCode Subject_Account_Name Target_Account_Name Message
| sort -\_time
```

![New local user created event reviewed in Splunk](/assets/images/proj-6/splunk-windows-event-4720.png)

This confirmed that Windows account-management activity was searchable in Splunk and could be reviewed with surrounding process context.

The event confirmed that the local account lab_backup was created on DESKTOP-HRMT55O by the user jules.

However, Event ID 4720 confirms the account-management action but does not fully show the process lineage or command-line context that produced it.

I therefore reviewed Sysmon Event ID 1, which records process creation:

```spl
index=_ source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
"lab_backup"
| rex field=\_raw "<EventID[^>]_>(?<EventID>\d+)</EventID>"
| rex field=\_raw "<Data Name='User'>(?<User>[^<]+)</Data>"
| rex field=\_raw "<Data Name='Image'>(?<Image>[^<]+)</Data>"
| rex field=\_raw "<Data Name='CommandLine'>(?<CommandLine>[^<]+)</Data>"
| rex field=\_raw "<Data Name='ParentImage'>(?<ParentImage>[^<]+)</Data>"
| search EventID=1 CommandLine="_/add_"
| table \_time host EventID User Image CommandLine ParentImage
| sort \_time
```

![Sysmon-Event-ID1](/assets/images/proj-6/Sysmon-Event-ID1,-process-creation.png)

## Python triage workflow

The Python script loads an alert JSON file, inserts the alert context into a structured prompt, sends it to the OpenAI API, and saves the AI-generated output.

Example command:

```bash
python main.py --alert alerts/alert_05_new_user_created.json
```

The output is saved under:

```text
results/ai_outputs/
```

Repository structure:

```text
ai-assisted-soc-triage/
├── alerts/
├── prompts/
├── results/
│   ├── ai_outputs/
│   └── manual_analysis/
├── main.py
├── requirements.txt
└── .env.example
```

## Example result: new local user created

For Alert 05, the LLM classified the event as:

```text
Risk level: medium
Classification: suspicious / needs_review
Primary MITRE mapping: T1136.001 — Create Account: Local Account
```

The result was useful because the model correctly identified that local account creation can be associated with persistence.

However, the model could not determine whether the action was authorized. The alert did not include a change ticket, business justification, or confirmation that the user was performing expected administrative work.

That made this a good test case: the behavior is security-relevant, but the evidence is not enough to declare compromise.

## Manual analyst validation

I manually reviewed each AI output using the same evidence.

For Alert 05, my manual assessment was:

```text
Classification: needs_review
Risk level: medium
```

Reasoning:

- A new local user account was created on a Windows endpoint.
- Local account creation can be associated with persistence.
- The command `net user lab_backup ... /add` is security-relevant and should be reviewed.
- There was no evidence that the account was added to the local Administrators group.
- There was no evidence of lateral movement, malware execution, credential theft, or exfiltration.
- The alert did not include change-management context, so authorization could not be confirmed.

The AI output was partially correct. It identified the right security concern and suggested a relevant MITRE ATT&CK mapping, but the final classification still required analyst validation.

## Human vs AI comparison

| Alert                          | AI Classification | Manual Classification   | Result            |
| ------------------------------ | ----------------- | ----------------------- | ----------------- |
| Suspicious PowerShell          | Needs review      | Needs review            | Correct           |
| Benign admin PowerShell        | Suspicious        | Benign / Needs review   | Partially correct |
| Failed login burst             | Suspicious        | Suspicious              | Correct           |
| SQL injection-style request    | Suspicious        | Suspicious              | Correct           |
| New local user created         | Suspicious        | Needs review            | Partially correct |
| Suspicious outbound connection | Needs review      | Needs review            | Correct           |
| Encoded PowerShell             | Suspicious        | Suspicious              | Correct           |
| Noisy false positive           | Suspicious        | Benign / False positive | Incorrect         |

Summary:

```text
Total alerts tested: 8
Correct: 5
Partially correct: 2
Incorrect: 1
```

## Key findings

The LLM was useful for:

- summarizing alert evidence
- extracting key observables
- suggesting triage questions
- producing consistent analyst notes
- identifying possible MITRE ATT&CK mappings

The LLM was risky when:

- evidence was weak or ambiguous
- suspicious tools also had legitimate admin uses
- MITRE mappings were plausible but not fully supported
- environmental context was missing

## What this project demonstrates

This lab demonstrates practical skills in:

- Splunk search and investigation
- Windows telemetry collection with Sysmon
- Windows Security Event Log analysis
- account-management triage
- alert context extraction
- Python scripting
- OpenAI API integration
- MITRE ATT&CK mapping
- SOC triage reasoning
- human validation of AI-generated security analysis

The main lesson: AI can help with first-pass SOC triage, but the analyst still needs to validate whether the evidence supports the conclusion.

## Conclusion

This project showed that an LLM can speed up alert review by summarizing evidence, identifying observables, and suggesting next steps.

However, it should not be treated as the final decision-maker.

The safest role for AI in this workflow is:

> Assist the analyst, but do not replace analyst judgment.
