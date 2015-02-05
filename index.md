---
layout: page
title: Welcome
tagline: Blog & Share
description: [Youly,立水桥]
---
{% include JB/setup %}

## 最近文章

<ul class="posts">
    {% for post in site.posts %}
        {% if post.title == 'Resume' %}
            {% continue %}
        {% endif %}
    
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

