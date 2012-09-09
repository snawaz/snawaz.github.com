---
layout: page
title: Introduction
---
{% include JB/setup %}

##TODO

I'll soon put some nice introduction here. 

As of now, I could just list the blogs I've written so far:

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

