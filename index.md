---
title: Home
layout: default
---
Welcome to my homepage.

# Recent blog posts

{% assign counter=0 %}
{% for post in site.posts %}
{% if counter < 3 %}
## {{post.title}} ({{ post.date | date_to_string  }})

{{ post.excerpt }}

[more...]({{post.url}} "Show complete post")
{% endif %}
{% assign counter=counter | plus:1 %}
{% endfor %}