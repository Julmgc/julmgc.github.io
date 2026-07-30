---
layout: archive
title: "Laboratórios"
permalink: /pt/laboratorios/
lang: pt-BR
author: julia_pt
sidebar:
  nav: "sidebar_pt"
author_profile: true
show_excerpts: true
---

{% assign posts_pt = site.posts | where: "lang", "pt-BR" %}

{% if posts_pt.size > 0 %}
{% for post in posts_pt %}
{% include archive-single.html type="list" %}
{% endfor %}
{% else %}

  <p>Nenhum laboratório em português foi publicado ainda.</p>
{% endif %}
