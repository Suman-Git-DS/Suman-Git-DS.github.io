---
layout: default
title: Blog
permalink: /blog/
---

{% include nav.html %}

# Blog

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
{{ post.date | date: "%B %d, %Y" }}

{{ post.excerpt }}

---
{% endfor %}
