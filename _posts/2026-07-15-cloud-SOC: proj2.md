---
title: "Public S3 Exposure with Access Analyzer and CloudTrail"
date: 2026-07-15
layout: single
lang: en
translation_key: cloud-soc-s3-exposure-access-analyzer-cloudtrail
author_profile: true
author: julia_en
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
excerpt: "A cloud security lab to detect, investigate, and remediate public S3 exposure using Access Analyzer, CloudTrail, EventBridge, and SNS."
permalink: /labs/cloud-soc-s3-exposure-access-analyzer-cloudtrail/
header:
  teaser: /assets/images/DOOR-AJAR.jpg
  image: /assets/images/DOOR-AJAR.jpg
  overlay_filter: 0.3
  overlay_image: /assets/images/DOOR-AJAR.jpg
  image_description: "A workflow for detecting, investigating, alerting on, and remediating public exposure in Amazon S3."
  image_height: 300px
---

### Project summary

In this lab, I created a controlled public exposure in an S3 bucket and used AWS-native services to detect the insecure configuration, investigate the change, generate an alert, and restore the bucket to a secure state.

**IAM Access Analyzer** identified the exposure, **CloudTrail** provided the change history, and **EventBridge + SNS** generated an email notification.

![AWS security workflow architecture](/assets/images/proj-2/RESUMO-DO-PROJ.png)

### Controlled public exposure

I created a disposable bucket containing only a test file (`dummy.txt`).

I then temporarily disabled the relevant **Block Public Access** protections and applied a bucket policy allowing public `s3:GetObject` access to the test object.

![S3 permissions showing the public exposure](/assets/images/proj-2/S3-public-permissions.png)

No sensitive data was stored in the bucket during the lab.

### Detection with IAM Access Analyzer

**IAM Access Analyzer** identified that the bucket was publicly accessible.

The finding showed that the exposure was caused by the bucket policy and that public read access was available through `s3:GetObject`.

![IAM Access Analyzer finding showing public S3 exposure](/assets/images/proj-2/AA-public-bucket-finding.png)

The finding confirmed that the bucket was publicly exposed through the configured policy.

### Investigation with CloudTrail

After detection, I reviewed **CloudTrail** to confirm the change that caused the public exposure.

Because the insecure configuration had been introduced through the bucket policy, I located the `PutBucketPolicy` event and reviewed the timestamp, the identity responsible for the API call, and the affected resource.

![CloudTrail evidence of the S3 bucket policy change](/assets/images/proj-2/CT-s3-policy-change.png)

The event confirmed the policy change and made it possible to associate the modification with the identity that performed the API call.

### Alerting with EventBridge + SNS

In addition to the Access Analyzer detection, I configured an alerting workflow for changes related to public S3 access.

A **CloudTrail trail** recorded management events, **EventBridge** filtered relevant S3 configuration changes, and **SNS** sent an email notification.

![EventBridge rule for S3 access-related changes](/assets/images/proj-2/EVB-s3-public-alert-rule.png)

<small><em>
EventBridge rule configured to capture CloudTrail events related to bucket policy and Block Public Access changes.
</em></small>

![SNS email notification](/assets/images/proj-2/SNS-s3-alert-email.png)

<small><em>
SNS notification generated after a `PutBucketPolicy` event, including the API action, event time, identity, and affected bucket details.
</em></small>

The notification exposed the API action, event time, and context of the recorded change.

### Remediation and validation

To remediate the exposure:

- I re-enabled **Block Public Access**;
- I removed the public bucket policy.

After remediation, I validated that the bucket no longer allowed public read access and that the insecure configuration had been removed.

![S3 bucket after remediation](/assets/images/proj-2/S3-remediated.png)

<small><em>
Post-remediation validation: Block Public Access is enabled and the public bucket policy has been removed.
</em></small>

### Conclusion

The lab integrated AWS-native services to detect, investigate, and remediate public exposure in S3.

**IAM Access Analyzer** identified the exposure, **CloudTrail** provided the change history and responsible identity, and **EventBridge + SNS** converted the configuration change into a notification.

After remediation, I validated that **Block Public Access** was enabled again and that the public bucket policy had been removed.
