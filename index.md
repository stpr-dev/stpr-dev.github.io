---
layout: home
title: Home
---


Systems engineer working across the stack: hardware-level acquisition, signal processing, and edge inference for real-time biomedical systems.

{% include banner.html %}

I write here about technical investigations that I find very interesting and that go deeper than a README but don't fit the format of a paper — benchmarks, architecture decisions, dead ends, and the occasional systems programming rabbit hole.



---

## Embedded Systems & Data Acquisition

{% assign embedded = site.posts | where_exp: "post", "post.categories contains 'embedded'" %}
{% if embedded.size > 0 %}
{% for post in embedded %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%b %Y" }}{% if post.excerpt %}<br>
  <small>{{ post.excerpt | strip_html | truncatewords: 20 }}</small>{% endif %}
{% endfor %}
{% else %}
*Posts coming soon.*
{% endif %}

---

## Machine Learning

{% assign ml = site.posts | where_exp: "post", "post.categories contains 'ml'" %}
{% if ml.size > 0 %}
{% for post in ml %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%b %Y" }}{% if post.excerpt %}<br>
  <small>{{ post.excerpt | strip_html | truncatewords: 20 }}</small>{% endif %}
{% endfor %}
{% else %}
*Posts coming soon.*
{% endif %}

---

## Ongoing Series

{% assign featured_series = "teensy-serial" %}  <!-- change this to feature a different series -->
{% assign series_posts = site.posts | where: "series", featured_series | sort: "date" %}
{% assign latest = series_posts | last %}

**{{ series_posts.first.title | split: " — " | first }}**
— {{ series_posts.size }} parts published, actively ongoing.
Latest: [{{ latest.title | split: " — " | last }}]({{ latest.url }}) ({{ latest.date | date: "%b %Y" }})
