---
layout: default
title: Posts
---
<div class="main-content">
<div class="posts-list">
  {% for post in site.posts %}
    <p><a href="{{ post.url }}">{{ post.title }}</a> 
    <br />
    {{ post.date | date: "%-d %B %Y" }}<br />
	{{ post.content | truncatehtml | truncatewords: 250 }} 
	</p>
  {% endfor %}
</div>
</div>
