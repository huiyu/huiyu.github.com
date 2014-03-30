---
layout: page
title: Hui Yu's Personal Tech Blog
---
{% include JB/setup %}


## Recent Posts
  {% for post in site.posts limit 5 %}
  <li>
      <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
  </li>
  {% endfor %}

