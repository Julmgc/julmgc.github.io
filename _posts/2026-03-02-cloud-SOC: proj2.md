---
title: "Cloud SOC MVP: Public S3 Exposure Detection, Alerting, and Investigation with Access Analyzer + CloudTrail"
date: 2026-03-02
layout: single
author_profile: true
toc: true
toc_sticky: true
categories: [Labs, SOC, Cloud]
tags:
  [
    AWS,
    S3,
    CloudTrail,
    EventBridge,
    SNS,
    IAM-Access-Analyzer,
    Incident-Response,
    Cloud-Security,
  ]
excerpt: "A cloud case: intentionally exposed an S3 bucket, detected it with IAM Access Analyzer, triggered an EventBridge + SNS alert, investigated the change with CloudTrail, and remediated it by restoring Block Public Access and removing the public bucket policy."
permalink: /labs/cloud-soc-s3-exposure-access-analyzer-cloudtrail/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A small crack in trust can open the door to deception — DNS poisoning in action."
  image_height: 300px
---

## Overview

This project demonstrates a focused cloud investigation workflow using AWS-native evidence:

- created a controlled **public S3 exposure** using a disposable bucket and dummy object
- detected the issue with **IAM Access Analyzer**
- configured **EventBridge + SNS** to send an email alert for S3 public-access-related changes
- investigated the change in **CloudTrail Event history** to confirm **what changed, when, and by which identity**
- remediated the exposure by restoring **Block Public Access** and removing the public bucket policy
- validated that the bucket was no longer publicly accessible

![Proj-2A - Architecture - aws-to-soc-pipeline](/assets/images/proj-2/ARCH-aws-to-soc.png)

<small><em>
A pipeline view showing S3 exposure → Access Analyzer detection → EventBridge + SNS alerting → CloudTrail investigation → remediation.
</em></small>

## Why this matters to a SOC analyst

Public S3 exposure is a common cloud security issue because it can unintentionally make stored data reachable from the internet. From an analyst perspective, the key tasks are to confirm the exposure, identify what control caused it, determine who made the change, ensure the team is notified quickly enough to respond, and verify that the risky state was fully removed.

This project focuses on that workflow using AWS-native evidence: detection, alerting, timeline building, attribution, remediation, and validation.

## Misconfiguration (controlled): public read on a dummy object

I created a disposable bucket and uploaded a single dummy file (`dummy.txt`). I then intentionally relaxed **Block Public Access** and applied a bucket policy granting `s3:GetObject` to all principals for the dummy object path.

![Proj-2A - S3 permissions showing public exposure](/assets/images/proj-2/S3-public-permissions.png)

<small><em>
The bucket became publicly readable because Block Public Access was disabled and the bucket policy granted `s3:GetObject` to all principals.
</em></small>

> **SOC note:** no sensitive content was ever stored in the bucket; exposure existed only briefly for the lab.

## Detection: IAM Access Analyzer finding (cloud-native signal)

IAM Access Analyzer produced the primary detection signal for this case. It identified that the bucket was publicly accessible, linked the exposure to the bucket policy, and showed that all principals had read access through `s3:GetObject`.

![Proj-2A - Access Analyzer finding for public S3 exposure](/assets/images/proj-2/AA-public-bucket-finding.png)

<small><em>
Detection signal — IAM Access Analyzer flagged the S3 bucket as publicly accessible, showing active public read access granted to all principals through the bucket policy.
</em></small>

## Investigation: CloudTrail Event history (who changed what, when)

To build a defensible incident timeline, I reviewed **CloudTrail Event history** around the exposure window and filtered for S3 configuration changes affecting the bucket.

What I looked for:

- bucket policy changes (`PutBucketPolicy` / `DeleteBucketPolicy`)
- public access block changes (`PutPublicAccessBlock` / `DeletePublicAccessBlock`)
- identity context (who performed the action)
- evidence supporting what changed and when

![Proj-2A - CloudTrail evidence of S3 policy/public access change](/assets/images/proj-2/CT-s3-policy-change.png)

<small><em>
Investigation proof — CloudTrail records the `PutBucketPolicy` event with the timestamp, actor, and affected S3 bucket, providing a defensible timeline for the exposure.
</em></small>

## Alerting: EventBridge + SNS email notification

To make the workflow more operationally realistic, I added an alerting step for S3 public-access-related changes.

For this lab, the alerting flow was:

- an active **CloudTrail trail** recorded S3 management events
- **EventBridge** matched configuration changes related to public exposure
- **SNS** sent an email notification with the affected bucket, API action, and event time

This added an operational monitoring layer to the workflow: the risky change was not only discoverable in AWS tools, but also actively notified as it happened.

![Proj-2A - EventBridge rule for S3 change alerting](/assets/images/proj-2/EVB-s3-public-alert-rule.png)

<small><em>
Alerting setup — EventBridge rule configured to match CloudTrail management events for S3 policy and public access block changes related to public exposure.
</em></small>

![Proj-2A - SNS email alert for S3 public-access change](/assets/images/proj-2/SNS-s3-alert-email.png)

<small><em>
Operational alert proof — SNS email triggered by the CloudTrail → EventBridge workflow after a `PutBucketPolicy` change, showing the AWS service, API action, and event time.
</em></small>

## Remediation: rollback the public exposure

Fix applied:

- re-enabled **Block Public Access**
- removed the public bucket policy

## Validation: confirm the bucket is no longer public

After remediation, I verified that:

- **Block Public Access** was restored
- the public bucket policy was removed
- the bucket no longer allowed public read access
- the risky configuration state was no longer present

![Proj-2A - S3 remediated (public access removed)](/assets/images/proj-2/S3-remediated.png)

<small><em>
Closure — after remediation, Block Public Access was restored and the public bucket policy was removed, so the bucket no longer allowed public read access.
</em></small>

## Mini cloud investigation checklist (TL;DR)

1. **Confirm the exposure** in S3 and Access Analyzer
2. **Review the alert** from EventBridge + SNS
3. **Build the timeline** in CloudTrail
4. **Attribute the change** to the responsible identity and timestamp
5. **Remediate** by restoring Block Public Access and removing the public policy
6. **Validate** that the bucket is no longer publicly accessible
