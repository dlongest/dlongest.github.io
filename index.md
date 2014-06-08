---
layout: default
title: Home
---
<ul class="home-list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> {{ post.date | date: "%-d %B %Y" }}
    </li>
  {% endfor %}
</ul>

{% include foot.html %}
