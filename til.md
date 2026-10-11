---
layout: page
title: Today I Learned
---

Just write some note about what I learn today. Pick a category to browse its notes.

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

<div class="til-entries">
{% for cat in categories %}
  {%- assign cat_count = 0 -%}
  {%- for item in sorted_til -%}
    {%- assign item_cat = item.category | default: 'General' -%}
    {%- if item_cat == cat -%}
      {%- assign cat_count = cat_count | plus: 1 -%}
    {%- endif -%}
  {%- endfor %}
  <section class="til-category" id="{{ cat | slugify }}">
    <h3 class="til-category-heading">
      <a href="#{{ cat | slugify }}" class="til-category-link">{{ cat }} <span class="til-category-count">{{ cat_count }}</span></a>
      <a href="{{ '/til/' | relative_url }}" class="til-back">&larr; All categories</a>
    </h3>
    <ul class="til-posts">
      {%- for item in sorted_til -%}
        {%- assign item_cat = item.category | default: 'General' -%}
        {%- if item_cat == cat %}
      <li><a href="{{ item.url | relative_url }}">{{ item.title }}</a> <small class="text-muted">&mdash; {{ item.date | date: site.date_format }}</small></li>
        {%- endif -%}
      {%- endfor %}
    </ul>
  </section>
{% endfor %}
</div>
