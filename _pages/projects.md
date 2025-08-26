---
title: "Projects"
permalink: /projects/
layout: single
author_profile: true
toc: true
sidebar:
  nav: "sidebar"
---

Below are my **project write-ups** (portfolio-style).  
Use tags like `web-security`, `networking`, `cloud` to organize.

{% assign project_posts = site.posts | where_exp: "p", "p.categories contains 'Projects'" %}
{% for post in project_posts %}

- [{{ post.title }}]({{ post.url | relative_url }}) — {{ post.date | date: "%b %d, %Y" }}
  {% endfor %}
