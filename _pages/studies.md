---
title: "Studies"
permalink: /studies/
layout: single
author_profile: true
toc: true
sidebar:
  nav: "sidebar"
---

**Study notes** and theory (malware, viruses, protocols, crypto, etc).

{% assign study_posts = site.posts | where_exp: "p", "p.categories contains 'Studies'" %}
{% for post in study_posts %}

- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%b %d, %Y" }}
  {% endfor %}
