---
layout: page
title: Blog
permalink: /blog/
---

{% assign sorted = site.tags | sort: 'tag' %}
{% for item in sorted %}
<h3>{{ item[0] }}</h3>
<ul>
    {% for post in item[1] %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
        {{ post.excerpt }}
      </li>
    {% endfor %}
  </ul>
{% endfor %}