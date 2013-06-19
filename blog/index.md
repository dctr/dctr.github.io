---
title: All Blog Posts
layout: default
---
{% for post in site.posts %}
- {{ post.date | date_to_string }}: [{{ post.title }}]({{ post.url }} "{{ post.title }}")
{% endfor %}
