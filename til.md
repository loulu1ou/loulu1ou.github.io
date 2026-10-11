---
layout: page
title: Today I Learned
---

Just write some note about what I learn today.

{% assign sorted_til = site.til | sort: 'order' %}

{%- comment -%} Collect the unique categories (posts without one fall under "General") {%- endcomment -%}
{% assign categories = '' | split: '' %}
{% for item in sorted_til %}
  {% assign item_cat = item.category | default: 'General' %}
  {% unless categories contains item_cat %}
    {% assign categories = categories | push: item_cat %}
  {% endunless %}
{% endfor %}
{% assign categories = categories | sort %}

## Entries

{% for cat in categories %}
### {{ cat }}

{% for item in sorted_til -%}
{% assign item_cat = item.category | default: 'General' -%}
{% if item_cat == cat -%}
- [{{ item.title }}]({{ item.url | relative_url }}) <small class="text-muted">&mdash; {{ item.date | date: site.date_format }}</small>
{% endif -%}
{% endfor %}

{% endfor %}
