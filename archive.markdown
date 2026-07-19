---
layout: page
title: Writing archive
permalink: /archive/
description: Every published essay, ordered from newest to oldest.
---

<div class="archive-list">
{% assign current_year = '' %}
{% for post in site.posts %}
  {% assign post_year = post.date | date: '%Y' %}
  {% if post_year != current_year %}
    {% unless current_year == '' %}</div>{% endunless %}
    <h2>{{ post_year }}</h2>
    <div class="archive-year">
    {% assign current_year = post_year %}
  {% endif %}
  <article>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: '%b %-d' }}</time>
    <div>
      <h3><a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a></h3>
      <p>{{ post.description | default: post.excerpt | strip_html | normalize_whitespace | truncatewords: 24 }}</p>
    </div>
  </article>
{% endfor %}
{% unless current_year == '' %}</div>{% endunless %}
</div>
