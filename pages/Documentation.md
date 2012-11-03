---
layout: page
title: Documentation
<!-- group: navigation -->
---
{% for p in site.pages %}
  {% if p.group == 'doc' %}
  <li><a href="{{ BASE_PATH }}{{p.url}}">{{p.title}}</a></li>
  {% endif %}
{% endfor %}
