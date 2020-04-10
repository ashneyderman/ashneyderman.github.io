---
layout: default
title: Alex Shneyderman
---
<ul>
  {% for post in site.posts %}
    <li style="list-style: none; padding-left: 20px; }">
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
