---
layout: default
title: Home
---
# Home

Welcome to my site. 

# Recent Blog Entries

{% assign counter=0 %}
{% for post in site.posts %}
{% if counter < 5 %}
## {{post.title}} <span class="pull-right annotation">*(from {{ post.date | date_to_string  }})*</span>

{{ post.excerpt }}

[more...]({{post.url}} "Show complete post")
{% endif %}
{% assign counter=counter | plus:1 %}
{% endfor %}
