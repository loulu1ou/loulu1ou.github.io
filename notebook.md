---
layout: page
title: My Red Team Notebook
---

A collection of my notes, operational workflows, and research in active directory security, exploit development, and evasion techniques. Built for practical offensive operations with an eye on detection telemetry.

## Chapters

{% assign sorted_notebook = site.notebook | sort: 'order' %}
{% for item in sorted_notebook %}
- [{{ item.title }}]({{ item.url | relative_url }})
{% endfor %}
