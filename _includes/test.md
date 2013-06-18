{% capture test %}
# Include

foo bar
{% endcapture %}
{{ test | markdownify }}