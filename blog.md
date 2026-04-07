---
layout: page
title: Blog
permalink: /blog/
---

# Blog

Occasional notes on speech models, training infrastructure, and whatever
else I've been hacking on.

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ul>
{% else %}

*no posts yet — check back soon.*

{% endif %}
