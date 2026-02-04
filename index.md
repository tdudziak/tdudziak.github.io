---
layout: default
title: Home
---

# Articles

{% for post in site.posts %}
<div style="display: flex; justify-content: space-between; align-items: baseline; gap: 1rem;">
  <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  <span><em>{{ post.date | date: "%d %b %Y" }}</em></span>
</div>
{{ post.summary }}  
{% endfor %}
