---
layout: default
title: Home
---
Welcome to my homepage.

# Recent Blog Entries

{% assign counter=0 %}
{% for post in site.posts %}
{% if counter < 5 %}
## {{post.title}} *(from {{ post.date | date_to_string  }})*

{{ post.excerpt }}

[more...]({{post.url}} "Show complete post")
{% endif %}
{% assign counter=counter | plus:1 %}
{% endfor %}