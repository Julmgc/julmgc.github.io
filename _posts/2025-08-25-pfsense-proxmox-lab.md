---
title: "pfSense + Proxmox Lab: VLANs & Traffic Analysis"
date: 2025-08-25
categories: [Labs] # ← shows up under /labs/
image_path: /assets/_images/lock.jpg
tags: [pfSense, Proxmox, VLANs, Wireshark]
sidebar:
  nav: "sidebar"
---

I built a virtual lab with pfSense inside Proxmox to test VLAN segmentation.

<!--more-->

The goal was to separate DMZ, Internal, and Management traffic, then capture flows using Wireshark...
