---
layout: default
title: Tags
permalink: /tags
---

<h1>{{ page.title }}</h1>
<p>&nbsp;</p>

<!-- 排序一下，这样 C、CMake、Julia 在前面 -->
{% capture tags %}
  {% for tag in site.tags %}
    {{ tag[0] }}
  {% endfor %}
{% endcapture %}
{% assign sortedtags = tags | split:' ' | sort %}

{% for t in sortedtags %}
  <h3 id="{{ t }}"><a href="{{ t }}">{{ t }}</a> ({{ site.tags[t] | size }})</h3>
  <ul>
    {% for post in site.tags[t] limit:3 %}
    <li>
        {{ post.date | date: "%Y-%m-%d" }}: <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
    {% endfor %}
    </ul>
{% endfor %}