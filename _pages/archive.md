---
permalink: /archive
layout: page
title: All Posts
---


<div class="archive">
  <ul>
    {% for post in site.posts %}
      <li>
        <a href=".{{ post.url }}">{{ post.title }}: {{post.subtitle}}</a>
      </li>
    {% endfor %}
  </ul>
</div>
