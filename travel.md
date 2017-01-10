---
layout: page
title: Travel
---

{% for dest in site.travel %}
<a href="{{ dest.url }}">{{ dest.title }}</a> ({{ dest.month }} {{ dest.year }})
{% endfor %}
