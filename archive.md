---
layout: default
title: Archive
---

## Blog Posts

<div class="main-content">
<div class="posts-list">
{% for post in site.posts %}
  <p>* {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})</p>
{% endfor %}
</div>
</div>