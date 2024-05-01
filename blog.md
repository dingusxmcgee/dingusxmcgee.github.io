---
layout: page
title: Blog
permalink: /blog/
---

{% assign sorted = site.tags | sort: 'tag' %}
{% for tag in sorted %}
<h3>{{ tag[0] }}</h3>
<ul>
    {% for post in tag[1] %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
        {{ post.excerpt }}
      </li>
    {% endfor %}
  </ul>
{% endfor %}