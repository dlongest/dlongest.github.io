---
layout: default
title: Home
---
<div class="main-content">
<ul class="home-list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> {{ post.date | date: "%-d %B %Y" }}
    </li>
  {% endfor %}
</ul>
</div>

{% include foot.html %}