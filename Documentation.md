---
layout : page
title : Documentation
group : navigation
comments : false
redirect : /Initialization.html
---
{% for p in site.pages %}
  {% if p.group == 'doc' %}
  <li><a href="{{ BASE_PATH }}{{p.url}}">{{p.title}}</a></li>
  {% endif %}
{% endfor %}
