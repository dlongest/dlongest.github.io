---
layout: default
title: Home
---
<div class="main-content">
<div class="posts-list">
  {% for post in site.posts %}
    <p><a href="{{ post.url }}">{{ post.title }}</a> 
    <br />
    {{ post.date | date: "%-d %B %Y" }}</p>
  {% endfor %}
</div>
</div>
