{% capture test %}
# Include

foo
{% endcapture %}
{{ test | markdownify }}