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

<div class="lab-list">
  {% for post in site.categories.Labs %}
    <article class="lab-item">
      <h3 class="lab-title">
        <a href="{{ post.url | relative_url }}" class="lab-link">{{ post.title }}</a>
        <span class="lab-date">— {{ post.date | date: "%b %-d, %Y" }}</span>
      </h3>
      <p class="lab-excerpt"><em>{{ post.excerpt | markdownify | strip_html | truncate: 180 }}</em></p>
    </article>
  {% endfor %}
</div>
