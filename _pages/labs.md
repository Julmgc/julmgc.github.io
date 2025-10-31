---
# ...existing front matter...
redirect_from:
  - /projects/
  - /studies/

title: "Labs"
permalink: /labs/
layout: single
author_profile: true
toc: true
sidebar:
  nav: "sidebar"
---

Hands-on **labs** (pfSense, Proxmox, VLANs, Wireshark, AD, etc).

{% assign lab_posts = site.posts | where_exp: "p", "p.categories contains 'Labs'" %}
{% for post in lab_posts %}

- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%b %d, %Y" }}
  {% endfor %}
