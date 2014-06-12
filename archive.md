---
layout: default
title: Archive
---

## Blog Posts

<div class="main-content">
<div class="posts-list">
{% for post in site.posts %}
  <p>{{ post.date | date_to_string }} | <a href="{{ post.url }}">{{ post.title }}</a></p>
{% endfor %}
</div>
</div>