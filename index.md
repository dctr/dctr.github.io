---
title: Home
layout: default
---
# Hello World

Hi there and welcome to my homepage.


# Recent blog posts

{% assign counter=0 %}
{% for post in site.posts %}
{% if counter < 3 %}
{{ post.excerpt }}

[more...]({{post.url}} "Show complete post")
{% endif %}
{% assign counter=counter | plus:1 %}
{% endfor %}