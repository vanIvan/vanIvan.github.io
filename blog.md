---
layout: page
title: Blog
permalink: /blog/
---

# Blog

Occasional notes on speech models, training infrastructure, and whatever
else I've been hacking on.

{% if site.posts.size > 0 %}
<ul class="post-cards">
  {% for post in site.posts %}
  <li class="post-card">
    <a href="{{ post.url | relative_url }}" class="post-card-link">
      {% if post.image %}
      <img src="{{ post.image | relative_url }}" alt="" class="post-card-image" loading="lazy">
      {% endif %}
      <h2 class="post-card-title">{{ post.title }}</h2>
      <p class="post-card-excerpt">{{ post.excerpt | default: post.description | strip_html | strip_newlines | truncate: 260 }}</p>
      <span class="post-card-date">{{ post.date | date: "%B %Y" }}</span>
    </a>
  </li>
  {% endfor %}
</ul>
{% else %}

*No posts yet — check back soon.*

{% endif %}
