---
layout: default
title: Home
---

## About

I am a software developer specializing in embedded systems, IoT devices, and real-time software in C, C++, and Rust.

For contracting work, contact me by email at <contact@tdudziak.com> or connect with me on [LinkedIn](https://www.linkedin.com/in/tomasz-dudziak-320bb7150).

## Blog Articles

{% for post in site.posts %}
<div style="display: flex; justify-content: space-between; align-items: baseline; gap: 1rem;">
  <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  <span><em>{{ post.date | date: "%d %b %Y" }}</em></span>
</div>
{{ post.summary }}
{% endfor %}
