---
layout: page
title: Devop Ninja
tagline: Randomly making things work with blood, sweat, and dirty hacks
---
{% include JB/setup %}

  {% for post in site.posts limit:5 %}
### [{{ post.title }}]({{ BASE_PATH }}{{ post.url }})
{{ post.date | date_to_string }}
{{ post.excerpt }}

<span><a href="{{ BASE_PATH }}{{ post.url }}">Read More</a></span> |
<a href="{{ BASE_PATH }}{{ post.url }}#disqus_thread" data-disqus-identifier="{{ post.url }}">Comments</a>

  {% endfor %}

