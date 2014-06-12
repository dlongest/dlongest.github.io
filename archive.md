---
layout: default
title: Archive
---

<div class="main-content">
<div class="posts-list">
{% for post in site.posts %}
  <p>{{ post.date | date_to_string }} | <a href="{{ post.url }}">{{ post.title }}</a></p>
{% endfor %}
</div>
</div>