---
layout: default
title: 分类
permalink: /categories.html
---

<h1>📂 分类</h1>
<ul>
  {% assign sorted_categories = site.categories | sort %}
  {% for category in sorted_categories %}
    <li>
      <a href="{{ site.baseurl }}/categories/{{ category[0] | slugify }}/">
        {{ category[0] }} ({{ category[1].size }})
      </a>
    </li>
  {% endfor %}
</ul>
