---
title: Tags
permalink: /tags/
layout: page
excerpt: Sorted article by tags.
---

<style>
  .tag {
    font-weight: bold;
    font-size: 12px;
    letter-spacing: 0.5px;
  }
</style>

<p>
{% for tag in site.tags %}
  {% capture name %}{{ tag | first }}{% endcapture %}
  <a class="tag" href="#{{name | downcase | slugify}}">{{ name | upcase }}(2)</a>
{% endfor %}
</p>

{% for tag in site.tags %}
{% capture name %}{{ tag | first }}{% endcapture %}

  <h3 class="post-header" id="{{ name | downcase | slugify }}">
    {{ name | downcase }}
  </h3>

  <ul>
  {% for post in site.tags[name] %}
  <li>
      <a href="{{ post.url }}">{{ post.title | escape }}</a>
  </li>
  {% endfor %} 
  </ul>
{% endfor %}
