---
layout: single
title: "Proyectos"
permalink: /projects/
---

Proyectos de Machine Learning / Deep Learning y de hardware / embedded en una sola página.

## Proyectos ML/DL

<div class="entries-list">
  {% for post in site.projects %}{% unless post.categories contains "hardware" %}{% include archive-single.html type="list" %}{% endunless %}{% endfor %}
</div>

## Proyectos Hardware/Embedded

<div class="entries-grid">
  {% assign hw_projects = site.projects | where: "categories", "hardware" %}
  {% for post in hw_projects %}{% include archive-single.html type="grid" %}{% endfor %}
</div>
