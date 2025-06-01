---
layout: default
title: Welcome to TechVidhi
---

# 👩‍💻 TechVidhi — Embedded Engineer Blog & Projects

🚀 I write about:
- Embedded Firmware Development
- Real-time Debugging
- IoT Device Design
- And more...

👉 [Read the Blog](/techvidhi.in/_posts/)

👇 Featured Posts:
{% for post in site.posts limit:3 %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
